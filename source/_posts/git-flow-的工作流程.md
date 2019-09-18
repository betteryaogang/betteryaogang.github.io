title: git-flow 的工作流程
author: betteryaogang
tags:
  - git-flow
categories:
  - 技术
date: 2019-05-16 15:08:00
---
&emsp;&emsp;多人团队用git进行版本管理时，最好能够商定一个统一的工作流程，今天就来学习一下git-flow。  

&emsp;&emsp;我的开发平台时WIN 10，首先去 [官网下载安装](https://git-scm.com/downloads)，安装成功之后，鼠标右击应该会出现 `Git GUI Here`，`Git Bash Here`两个选项，说明安装完成，接下来就可以正式进行git-flow学习之路。

### 什么是git-flow？
&emsp;&emsp;git-flow可以看作是一组封装好的git命令集合，它出现的目的不是为了取代git。

### 怎么使用？
&emsp;&emsp;想在项目中引入git flow非常简单，只需在项目根目录执行`git flow init`命令，根据提示配置一些命名规则即可。

```
$ git flow init
Initialized empty Git repository in C:/CODE/test/.git/
Branch name for production releases: [master] 
Branch name for "next release" development: [develop] 

How to name your supporting branch prefixes?
Feature branches? [feature/] 
Release branches? [release/] 
Hotfix branches? [hotfix/] 
```
git flow允许你使用自己喜欢的branch名字。不过我建议直接使用默认的命名机制，不需要输入任何东西，直接一步步确定就行。

当项目引入git flow后，原有的git命令并不会受到任何影响，你既可以选择使用git flow命令，当然也可选择使用git命令。

### git flow的分支模式

&emsp;&emsp;git-flow 模式会预设两个主分支在仓库中：

- **master分支**  
	master 只能用来放正式产品代码。你不能直接在 master 分支上进行开发，而是在其他指定的，独立的特性分支中。不直接提交改动到 master 分支上也是很多工作流程的一个共同的规则。

- **develop分支**  
	develop 是你进行任何新的功能开发的基础分支。当你开始一个新的功能分支时，它将是**开发**的基础。另外，该分支也汇集所有已经完成的功能，并等待被整合到 master 分支中。
    
这两个分支是项目的**长期分支**。它们会存活在项目的整个生命周期中。而其他的分支，例如针对新feature的分支，针对release的分支，仅仅只是临时存在的。它们是根据需要来创建的，当它们完成了自己的任务之后就会被删除掉。

### 项目实践流程

**1. 开始新功能开发**  
&emsp;&emsp;当我们需要开发一个新功能时，首先我们执行git flow 命令 `git flow feature start my-feature`：

```
$ git flow feature start my-feature
Switched to a new branch 'feature/my-feature'

Summary of actions:
- A new branch 'feature/my-feature' was created, based on 'develop'
- You are now on branch 'feature/my-feature'
```
该命令实际上可以分解为几个git命令：  

- 首先git-flow 会创建一个基于develop分支的名为“feature/my-feature”的分支（这个 “feature/” 前缀 是一个可配置的选项设置）。在做新功能开发时使用一个独立的分支是版本控制中最重要的规则之一。
- 然后git-flow会帮你直接checkout这个新的分支，此时代码已经在“feature/my-feature”这个分支上了，这样你就可以直接开始干活了。

**2. 新功能开发完成**  
&emsp;&emsp;经过一段时间的开发，当我们终于开发完成的时候，我们需要执行git flow命令 `git flow feature finish my-feature`：

```
$ git flow feature finish my-feature
Switched to branch 'develop'
Updating 6bcf266..41748ad
Fast-forward
    Test.java | 0
    1 file changed, 0 insertions(+), 0 deletions(-)
    create mode 100644 Test.java
Deleted branch feature/my-feature (was 41748ad).
```
该命令所做的事情是把我们在`my-feature`分支上开发的内容合并到develop分支，并删除`my-feature`分支，同时切换到develop分支。

**3. 开始release**  
&emsp;&emsp;当我们develop分支开发的功能积累到一定成熟阶段的时候（这意味着：第一，它包括所有新的功能和必要的修复；第二，它已经被彻底的测试过了），我们就需要进行release发布，我们需要执行git flow命令 `git flow release start 2.1.1`：

```
$ git flow release start 2.1.1
Switched to a new branch 'release/2.1.1'
```
请注意：与进行feature开发不同，release 分支是使用版本号命名的。这是一个明智的选择，这个命名方案还有一个很好的附带功能，那就是当我们完成了release 后，git-flow 会自动为本次release打上版本tag。

有了这个 release 分支，再没有完成这个release分支时，我们还可以在这个分支上进行一些修补的工作。

**4. 完成release**  
&emsp;&emsp;当一切都准备就绪，我们就可以执行git flow命令 `git flow release finish 2.1.1`：

该命令会完成如下一系列的操作：

- 首先，git-flow 会拉取远程仓库，以确保目前是最新的版本。
- 然后，release 的内容会被合并到 “master” 和 “develop” 两个分支中去，这样不仅master分支为最新的版本，而且新的功能分支也将是最新代码。
- 为便于识别和做历史参考，release 提交会被打上这个 release 名字的tag（在我们的例子里是 “2.1.1”）。
- 最后会清理操作，release分支会被删除，并且项目代码会回到 “develop”分支。  

此时从 Git 的角度来看，release 版本现在已经完成。接下来你可以通过将代码推送到远程仓库来触发自动部署流程、或自己手动打包来进行部署完成本次版本发布。

**5. 开始hotfix热修复**  
&emsp;&emsp;很多时候，虽然版本是发布了，但是上线之后还是会出现一些小问题需要我们进行热修复，这种情况下我们不管使用feature分支还是release分支都不是太好，git flow为我们提供了hotfix分支来应对这种情况，我们可以执行git flow命令 `git flow hotfix start fixbug`

这个命令会创建一个名为“hotfix/fixbug”的分支。因为这是对产品代码进行修复，所以这个 hotfix 分支是基于“master”分支。
这也是和release分支及feature分支最明显的区别。因为你不应该在一个还不完全稳定的开发分支上对产品代码进行地修复。

**6. 完成hotfix热修复**  
&emsp;&emsp;在完成bug修复后，我们执行git flow命令 `git flow hotfix finish fixbug`

这个过程非常类似于发布一个 release 版本：
- 完成的改动会被合并到“master”中，同样也会合并到“develop”分支中，这样就可以确保这个错误不会再次出现在下一个 release 中。
- 这个 hotfix 程序将被标记起来以便于参考。
- 这个 hotfix 分支将被删除，然后切换到 “develop” 分支上去。
- 接下来和完成 release 的流程一样，还需要编译和部署你的产品。