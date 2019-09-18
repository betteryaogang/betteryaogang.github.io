title: mybatis+mysql 返回tinyint(1)类型的数据时 会自动转为boolean类型的问题解决
author: betteryaogang
tags:
  - mybatis
  - mysql
categories:
  - 技术
date: 2019-07-17 15:52:00
---
&emsp;&emsp;今天再弄一个历史项目时需要查一张表，有一个state字段定义为 `tinyint(1)` 类型，用来表示这一行记录的状态，值通常会有0、1、2等。

&emsp;&emsp;正常来讲mysql中的tinyint类型会映射成Java中的Byte数据类型，然而当我以 `Map<String, Object>` 来接收数据时，state字段却意外的变成boolean类型的`true/false`，经过测试，state为0的时候会返回`false`,其他值一律会返回`true`,这样的返回值当然是有问题的。

经过查找，解决方案如下：

** 1. 编写mybatis xml文件时对该字段特殊处理一下**
- 在select该字段时用ifnull处理
```
 ifnull(f.state, 0) AS state
```


** 2. 在jdbc连接上加上参数tinyInt1isBit=false**

```
jdbc:mysql://localhost:3306/test?tinyInt1isBit=false
```
** 3. 日常开发中应尽量避免定义tinyint(1)类型**

&emsp;&emsp; mysql中tinyInt(1) 类型一般只用来代表Boolean含义的字段，0代表False，1代表True，如果一个字段需要存储除0或1之外的数值，那么需要将字段定义成tinyint(N), N>1。例如tinyint(4)。