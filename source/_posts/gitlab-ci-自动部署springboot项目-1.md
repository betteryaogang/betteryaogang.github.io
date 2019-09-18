title: gitlab ci 自动部署springboot项目
author: betteryaogang
tags:
  - gitlab
  - gitlab-CI/CD
  - gitlab-runner
categories:
  - 技术
date: 2019-05-16 10:00:00
---
&emsp;&emsp;项目代码转到gitlab管理之后，就想着能够自动部署，折腾了几天终于搞定了，记录一下。

### 服务器情况介绍
- 所有服务器都是centos7的系统，GitLab CE 11.10.4 
- gitlab服务器 因gitlab程序需要的内存占用比较大，为了稳定单独用一台服务器跑gitlab程序。
- 测试环境服务器
- 正式环境服务器
- 测试及正式环境服务器需分别安装基础的java、git、maven环境（根据项目实际情况安装）



### 步骤

**1. ~~在测试及正式服务器上安装gitlab-runner，我们要实现的自动部署就是通过gitlab-runner来实现的~~** 

**1. 现修改为在gitlab服务器上安装gitlab-runner、执行job，之后传输编译好的文件到目标服务器，再通过ssh到目标服务器上执行部署操作**    -2019年9月18日

- 安装方法可参考[官方文档](https://docs.gitlab.com/runner/install/)

- 添加Gitlab的官方源：

```
# For Debian/Ubuntu/Mint
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | sudo bash

# For RHEL/CentOS/Fedora
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.rpm.sh | sudo bash
```

- 安装

```
# For Debian/Ubuntu/Mint
sudo apt-get install gitlab-runner

# For RHEL/CentOS/Fedora
sudo yum install gitlab-runner
```
**2.注册Runner，Runner需要注册到Gitlab跟项目相关联才可以被项目所使用，一个gitlab-runner服务可以注册多个Runner。**

- 到gitlab项目主页，settings -> CI/CD选项可以看到：

```
Set up a specific Runner manually
	1. Install GitLab Runner
	2. Specify the following URL during the Runner setup: http://x.x.x.x/ 
	3. Use the following registration token during setup: UnRDvasdDRswbtcT4a1 
	4. Start the Runner!
```
我们需要关注的是第2、3步中的URL及token，一会注册Runner的时候会用到

- 注册  
运行gitlab-runner register，按提示操作 （网上有些教程会在该命令前面加sudo，这可能会导致后续执行的一个job stuck问题）
```
$ gitlab-runner register
Please enter the gitlab-ci coordinator URL (e.g. https://gitlab.com )
http://x.x.x.x/
Please enter the gitlab-ci token for this runner
UnRDvasdDRswbtcT4a1
Please enter the gitlab-ci description for this runner
my-runner
INFO[0034] fcf5c619 Registering runner... succeeded
Please enter the executor: shell, docker, docker-ssh, ssh?
shell
INFO[0037] Runner registered successfully. Feel free to start it, but if it's
running already the config should be automatically reloaded!
```
到此，我们的runner就已经注册好了，我们刷新gitlab页面，可以看到`Runners activated for this project`下面已经出现了我们刚注册的`my-runner`

**3. 配置构建任务**  
&emsp;&emsp;虽然现在我们已经注册好了runner，但此时项目还不会执行自动部署，因为runner并不知道要干什么。我们需要在项目根目录提供一个 `.gitlab-ci.yml`的配置文件来告诉runner应该做哪些事情  

```
stages:
  - build
  - deploy

build_pre_job:
  stage: build
  before_script:
    - echo "====================before_build_pre======================"
  script:
    - /home/gitlab-runner/sh/build_pre.sh
  after_script:
    - echo "====================after_build_pre========================"
  only:
    - develop

build_prod_job:
  stage: build
  before_script:
    - echo "====================before_build_prod======================"
  script:
    - /home/gitlab-runner/sh/build_prod.sh
  after_script:
    - echo "====================after_build_prod========================"
  only:
    - master
    
deploy_pre_job:
  stage: deploy
  variables:
    GIT_STRATEGY: none
    GIT_CHECKOUT: "false"
  before_script:
    - echo "====================before_deploy_pre======================"
  script: 
    - /home/gitlab-runner/sh/deploy_pre.sh
  after_script:
    - echo "====================after_deploy_pre========================"
  only:
    - develop
    

deploy_prod_job:
  stage: deploy
  variables:
    GIT_STRATEGY: none
    GIT_CHECKOUT: "false"
  before_script:
    - echo "====================before_deploy_prod======================"
  script: 
    - /home/gitlab-runner/sh/deploy_prod.sh
  after_script:
    - echo "====================after_deploy_prod========================"
  only:
    - master

```
- stages是用于定义场景阶段，可以被任务用于定义所属stage,stages定义的元素的顺序决定了任务执行的顺序:

	1. 任务指定的stage名相同，该多个任务将并行执行(但是经过测试，貌似也是按照A-Z的头字母顺序顺序执行的。只是GitLab-UI上看着是并行)
	2. 下一个场景阶段的任务将会在前一个场景阶段的任务都完成的情况下执行
- variables是自定义每个job特有的一些行为，因为每个job执行时runner首先会默认去fetch git仓库拉取最新代码，而我们的deploy任务是不需要再去git仓库拉代码的，因为上一build阶段已经拉去过最新代码，并将代码编译成可执行jar包
    
- ** 总结一下来讲，这个脚本要做的事情是:**

	1. develop分支改变，执行build_pre以及deploy_pre
	2. master分支改变，执行build_prod以及deploy_prod
    3. build阶段所做的事情是拉取最新代码，执行mvn package编译成jar包，并通过rsync命令传到要部署的服务器对应目录
    4. deploy阶段所做的事情是ssh登录到要部署的服务器上， kill掉老的进程，并启动新的jar包
   
 **4. 编写shell脚本**

&emsp;&emsp;上面步骤中配置的job中script是具体要执行的shell脚本，下面我们来看一下脚本里都做了哪些事情

<font color=red>
**开始之前，请先将gitlab服务器上gitlab-runner用户的ssh key 发送到你需要最终部署的服务器上，以方便后面发送文件及免密ssh的操作**</font>

&emsp;&emsp;~~**因为gitlab-runner是单独以gitlab-runner用户运行的 所以需要注意下用户的权限问题**~~

- build_pre.sh

```
#!/bin/bash
echo "===============================build_pre start================================"
cd /home/gitlab-runner/builds/x
echo "STEP1：开始编译代码"

mvn clean package -Ppre -Dmaven.test.skip=true

cd /home/gitlab-runner/builds/x/target
echo "STEP2：准备发送文件到测试服务器"

now=$(date +%Y%m%d-%H%M%S)

rsync  -avz --progress --stats --backup --suffix=$now  /home/gitlab-runner/builds/x/target/x*.jar 你的用户@你的服务器IP:/需要放置文件的目录

echo "===============================build_pre end========================="


```
执行mvn package打包，并将jar包传输到目标服务器的对应目录

- deploy_pre.sh

```
#!/bin/bash
echo "===============================deploy_pre start========================="
ssh -T 你的用户@你的服务器IP  << 'eof'

PID=$(ps -ef | grep "X" | grep -v grep | awk '{print $2}')
echo "dt pid:$PID"
sudo kill -9 $PID
cd /存放传输文件的目录
sudo nohup java -jar x.jar  >/root/nohup.out 2>&1 & #必须将nohup.out 文件重定向 否则gitlab页面上job结束不了，会一直刷log

echo "==============================deploy_pre end==========================="

time=$(date '+%F %H:%M:%S' )
content="X项目于$time部署成功"
#echo "'$content'"
curl 'https://oapi.dingtalk.com/robot/send?access_token=token' -H 'Content-Type:application/json' -d '{"msgtype":"text","text":{"content":"'"${content}"'"}}'
#
exit
eof
echo "=============deploy done!==================="

```
首先查找正在运行的老jar包进程pid并杀掉进程，执行启动命令,并curl推送部署消息到钉钉机器人。

值得注意的是`sudo nohup java -jar x.jar  >/root/nohup.out 2>&1 &` 这一命令，一开始我执行的是`nohup java -jar x.jar &`命令，程序后台执行，可是发现gitlab CI那边显示的job一直是running状态，console控制台也一直在输出log日志，导致job以及pipelines那边一直卡在那边，这种情况肯定是不能接受的，上网一番搜索，并没有找到解决方案。

后来想想既然是由于输出日志卡在那边，那我把日志输出重定向到另外一个文件，让gitlab runner的job进程结束掉应该就可以了，于是命令改写一下将日志输出到另外一个文件`/root/nohup.out`，再次执行job，成功结束job，pipelines status也显示`passed`。  

至此，通过gitlab ci 自动部署springboot项目整个环境搭建完成。


** 问题记录**

 1. This job is stuck, because the project doesn't have any runners online assigned to it. Go to Runners page
 
 	[stackoverflow解决方案](https://stackoverflow.com/questions/53370840/this-job-is-stuck-because-the-project-doesnt-have-any-runners-online-assigned)

找到你的项目 Settings -> CI/CD -> Expand Runners -> edit runner -> 勾选 `Indicates whether this runner can pick jobs without tags`选项。
