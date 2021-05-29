---
layout: post
title: Linux基础系列(一)之命令的分类与shell及简单的进程管理指令
categories: Operating_System Linux
description: 本篇是Linux入门的开篇，从这一篇开始我们将简单的认识Linux，本系列主要学习目标是熟悉Linux操作，了解常用的Linux命令。本篇将简单介绍Linux命令的分类及shell的简单认识及简单的进程管理指令
keywords: Operating_System Linux
---

本篇是Linux入门的开篇，从这一篇开始我们将简单的认识Linux，本系列主要学习目标是熟悉Linux操作，了解常用的Linux命令。本篇将简单介绍Linux命令的分类及shell的简单认识及简单的进程管理指令
======

### Linux指令分类

Linux的指令分为两种，一种是内部指令，一种是外部指令。内部指令的意思就是shell自带的指令，外部指令的意思就是其他非shell程序提供的指令

#### type

- 我们可以通过type指令来鉴别某一个指令是内部指令还是外部指令

  ![image](\images\posts\Operating_System\2021-04-03-Linux基础系列(一)之命令的分类与shell-1.png)

  我们可以看到，cd是shell内部自带的，这就是内部指令

  ![image](\images\posts\Operating_System\2021-04-03-Linux基础系列(一)之命令的分类与shell-2.png)

  我们在查看ifconfig指令的type的时候发现直接返回了一个路径，这说明ifconfig是一个外部指令，我们前往去查看一下

  ![image](\images\posts\Operating_System\2021-04-03-Linux基础系列(一)之命令的分类与shell-3.png)

#### vi

- 既然找到了ifconfig，我们就可以使用vi进入查看

  ![image](\images\posts\Operating_System\2021-04-03-Linux基础系列(一)之命令的分类与shell-4.png)

  我们可以发现，这里面是一堆的乱码，这说明，这个文件不是**文本类型**的，我们来查看一下这个文件的类型

#### file

![image](\images\posts\Operating_System\2021-04-03-Linux基础系列(一)之命令的分类与shell-5.png)

这个文件的类型就是 ELF ，是一种可执行的二进制文件

### Bash Shell

#### 介绍

bash shell是一个程序，这个程序的作用是将我们在命令行上输入的字符串切割后转换成为指令，最后交给Linux系统内核去执行

#### 切割方式

yum install man

- 我们以上面的指令为例，bash shell会将字符串按照空格切分，将切分出来的第一个作为指令，后面的作为参数。
- 如果是内部指令，那么shell直接就将指令和指令的参数发送给kernel
- 如果是外部指令，那么shell就会将这个外部指令的可执行文件也一并交给内核，让内核直接执行

#### 外部指令的效率

- 有人可能就会好奇，那么外部指令在执行的时候是不是要暴力匹配一下文件系统中的所有文件夹来找出外部指令的可执行文件呢？其实不是的

#### echo

- echo指令有点像Java的syso，可以将一些字符串打印到命令行

  ![image](\images\posts\Operating_System\2021-04-03-Linux基础系列(一)之命令的分类与shell-6.png)

- 我们还可以自定义一个参数，然后将这个参数打印出来

  ![image](\images\posts\Operating_System\2021-04-03-Linux基础系列(一)之命令的分类与shell-7.png)

- 除了自定义参数，我们还可以自定义数组

  ![image](\images\posts\Operating_System\2021-04-03-Linux基础系列(一)之命令的分类与shell-8.png)

  这里要注意！数组定义外面要有一层**小括号**，并且每一个元素**必须使用空格分割**，其他的分隔符不行

  想要打印数组中的对象必须按照 ${变量名[打印元素的index]}的格式来，不然也会出错

- 了解了echo指令后，我们来看一下Linux系统的一个变量  PATH

  ![image](\images\posts\Operating_System\2021-04-03-Linux基础系列(一)之命令的分类与shell-9.png)

  这个PATH变量就相当于windows的环境变量，当我们的bash shell在执行一个命令的时候，如果发现这个命令是一个外部命令，那么bash shell程序就会从PATH变量中保存的目录下去找这个程序，如果找不到就会报错 command not found，通过这样的方式就不需要暴力匹配路径了

  除了在PATH里面寻找以外，Linux还有一层优化，它会将执行了的指令的路径以hash的形式存储起来，下一次再次执行的时候，效率就会更加高

  ![image](\images\posts\Operating_System\2021-04-03-Linux基础系列(一)之命令的分类与shell-10.png)

  ![image](\images\posts\Operating_System\2021-04-03-Linux基础系列(一)之命令的分类与shell-11.png)

### 进程管理指令

#### ps -fe

首先，我们可以使用 echo $$ 来打印出当前bash shell的进程号

![image](\images\posts\Operating_System\2021-04-03-Linux基础系列(一)之命令的分类与shell-12.png)

接下来我们可以调用 ps -fe 指令来查看当前所有的进程

![image](\images\posts\Operating_System\2021-04-03-Linux基础系列(一)之命令的分类与shell-13.png)

我们重新启动一个xshell，发现这个xshell的进程号不同了

![image](\images\posts\Operating_System\2021-04-03-Linux基础系列(一)之命令的分类与shell-14.png)

#### Kill -9

![image](\images\posts\Operating_System\2021-04-03-Linux基础系列(一)之命令的分类与shell-15.png)

我们直接杀死这个进程

![image](\images\posts\Operating_System\2021-04-03-Linux基础系列(一)之命令的分类与shell-16.png)

![image](\images\posts\Operating_System\2021-04-03-Linux基础系列(一)之命令的分类与shell-17.png)