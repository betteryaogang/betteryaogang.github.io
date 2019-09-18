title: 通过docker运行gitlab， clone项目时显示hostname地址不对
author: betteryaogang
tags:
  - docker
  - gitlab
categories:
  - 技术
date: 2019-05-10 16:29:00
---
&emsp;&emsp;自己折腾在服务器搭了一个gitlab来管理代码，为了方便选择通过docker来运行  
启动命令：
```javascript
docker run \
    --publish 443:443 --publish 80:80 --publish 22:22 \
    --name gitlab \
    --volume /root/gitlab/config:/etc/gitlab \
    --volume /root/gitlab/logs:/var/log/gitlab \
    --volume /root/gitlab/data:/var/opt/gitlab \
    gitlab/gitlab-ce

```
按照教程一路下来倒也没出什么问题，顺利登入系统。  

&emsp;&emsp;OK，接下来导入原有项目，gitlab很贴心的提供了从主流平台import功能，项目导入过程也是很顺利。 下面clone该项目，gitlab打开该项目主页，点开clone按钮（一切还是熟悉的操作）
当我看到 `git@d9d5dfeda5a0:xxx/xxx.git` 这一串地址时，我的内心时有点懵逼的。可我还是抱着一丝丝希望执行了git clone,果然是失败了，看错误应该是找不到该hostname，`d9d5dfeda5a0`这一串神秘代码按照正常操作应该是一个IP或者域名才对，这TM是啥？？

&emsp;&emsp;从头回顾一下，大的流程应该没问题，突然想到gitlab是跑在docker container里啊，这一串看起来有点像CONTAINER ID呀？  
执行`docker ps`，果然是CONTAINER ID，看来找到问题了，那就好解决了，只需要在执行docker run 的时候添加`--hostname=你的ip/域名`就行了  
修改启动命令：  
```javascript
docker run \
    --publish 443:443 --publish 80:80 --publish 22:22 \
    --hostname xxx.com
    --name gitlab \
    --volume /root/gitlab/config:/etc/gitlab \
    --volume /root/gitlab/logs:/var/log/gitlab \
    --volume /root/gitlab/data:/var/opt/gitlab \
    gitlab/gitlab-ce

```
可以看到地址已经变成了ip，git clone也能成功。

-----
##### 另外还需要注意的一点  
&emsp;&emsp;还有当docker restart之后，想要container自启动的话 可以在docker run时加上参数`--restart=always`，但如果docker run时忘了加--restrt也不要紧，不需要重新docker run，只需要执行`docker container update --restart=always <container ID>`即可。  
例：CONTAINER ID 为`d9d5dfeda5a0`  
执行 `docker container update --restart=always d9d5`