title: 树莓派安装CUPS打印服务支持Brother兄弟打印机
author: betteryaogang
date: 2021-09-22 17:09:14
tags:
---
&emsp;&emsp;家里的兄弟打印机是USB线连接的，必须插在电脑上才能打印，使用起来不是很方便，想改造一下支持手机打印。
淘了个二手树莓派3b安装了CUPS打印服务，打印机插树莓派上，这样手机都可以无线打印了，折腾好久记录一下。

### 步骤

**1. ** 安装树莓派系统本身就不用说了吧，这个没什么难度，需要注意的是安装完之后需要check一下几个环境变量，后面CUPS服务会使用到
命令：
```
locale
```
LANGUAGE

LC_ALL

LANG

注意已上几个变量，没有值的话，编辑 ~/.bashrc文件
export设置一下en_US.UTF-8

另外，最好执行一下raspi-config设置一下时区以及locale

apt换源
/etc/apt/sources.list
换中科大源 mirrors.ustc.edu.cn/raspbian


兄弟打印机官方只提供了intel i386平台的驱动，而树莓派本身CPU是ARM系的 直接安装deb驱动包就会得到一个错误
```
package architecture (i386) does not match system (armhf)
```
这个问题很长时间都没有头绪，尝试了很多方法，都没有成功，后来翻到一个很老的帖子，发现了兄弟官方一个支持armhf架构的通用驱动包，可以支持大部分的兄弟打印机型号，安装后成功驱动打印机。

最新更新：

用手机打印后发现格式错乱，无解，至此 CUPs改造有线usb打印机的方案宣告失败。