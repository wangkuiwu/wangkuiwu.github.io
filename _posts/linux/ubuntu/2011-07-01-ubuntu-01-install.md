---
layout: post
title: "Linux学习基础篇01 双系统中ubuntu的安装方法"
description: "linux ubuntu"
category: linux
tags: [linux]
date: 2011-07-01 09:01
---


> 本文是Linux学习的第一篇。从本文开始，就可以一步步的开始Linux学习之旅！  
> 本文介绍ubuntu系统的安装方法。更准确的说，是在已经安装Windows系统的电脑(台式机和笔记本通用)中，如何安装ubuntu系统。  


# 双系统安装说明

**给电脑安装双系统(Windows系统和Linux系统)时，一定要先装Windows系统，然后再安装Linux系统！**

为什么呢？

这是因为是电脑开机后，要先执行一段bootloader引导程序；再由引导程序启动操作系统。Windows的引导程序和Linux系统的引导程序是不同的！  
(01) Windows的引导程序只能识别Windows程序，无法识别到Linux。  
(02) Linux的引导程序则能识别到不同的操作系统！  
也就是说，如果你先安装Linux系统；然后，再安装Windows系统。那么，开机时会执行Linux的引导程序，而该引导程序只能识别到Windows系统；也就无法登录Linux系统了。


# Ubuntu安装步骤

下面开始介绍Ubuntu的安装步骤，其中涉及到的步骤分为两种情况。  
**情况1**： 在“32位的Windows XP/Windows 7”下 安装 “32位的ubuntu 12.04”  
**情况2**： 在“64位的Windows 7”下 安装 “64位的ubuntu 12.04”  

在讲解每个步骤时，可能"情况1"和"情况2"的处理方式不同。如果未加说明，则是统一处理。

**第1步：下载一个ubuntu-12.04.iso；放到C盘根目录下**

&nbsp;&nbsp;&nbsp;&nbsp; Ubuntu下载链接如下：[http://www.ubuntu.com/download](http://www.ubuntu.com/download)


**第2步：下载一个grub4dos，然后将压缩包中的：“grldr”、“grldr.mbr”、“ grub.exe”、“menu.lst”文件复制到 C 盘根目录下**

&nbsp;&nbsp;&nbsp;&nbsp; grub4dos下载链接如下：[http://files.cnblogs.com/skywang12345/grub4dos-0.4.4-2009-06-20.zip](http://files.cnblogs.com/skywang12345/grub4dos-0.4.4-2009-06-20.zip)

 
**第3步：第修改menu.lst文件**

在该步骤中，“情况1”和“情况2”的处理方式不同！

"情况1"：修改menu.lst，是它的内容如下：

    title Install Ubuntu
    root (hd0,0)
    kernel /vmlinuz boot=casper iso-scan/filename=/ubuntu-12.04.iso ro quiet splash locale=zh_CN.UTF-8
    initrd /initrd.lz


"情况2"：修改menu.lst，是它的内容如下：

    title Install Ubuntu
    root (hd0,0)
    kernel (hd0,0)/vmlinuz boot=casper iso-scan/filename=/ubuntu-12.04.iso ro quiet splash locale=zh_CN.UTF-8
    initrd (hd0,0)/initrd.lz

 

注意：此处**iso-scan/filename**的值必须和"第一步"中拷贝到C盘根目录下的ubuntu-12.04.iso的文件同名！

 


**第4步：在C盘根目录下打开（或win7下建立）一个 boot.ini 的文件，并修改boot.ini的内容**

在该步骤中，“情况1”和“情况2”的处理方式不同！

“情况1”：如果Windows系统是32位Windows XP，则在boot.init中添加以下：

    C:\grldr="grub"

 

“情况1”：如果Windows系统是32位Windows 7，则新建boot.ini，并且boot.ini的内容如下：

    [boot loader]
    [operating systems]
    C:\grldr="grub"

 

“情况2”：如果是win7，则新建boot.ini，并且boot.ini内容如下：

    [boot loader]
    [operating systems]
    c:\grldr.mbr="grub"

 

**第4步：将ubuntu-12.04.iso 中的“.disk”文件夹解压至C: ；再将casper目录下的“initrd.lz”、“vmlinuz”这两个文件也解压至C:**

**第5步：重新电脑，选择“grub”，选择最后一项“Install Ubuntu”**

**第6步：进入安装系统后，再终端输入如下命令：**

    sudo umount –l /isodevice

**第7步：选择桌面上的“安装Ubuntu…”，进入安装ubuntu**

注意：  
(01) 推荐手动选择分区。  
&nbsp;&nbsp;&nbsp;&nbsp; 如果你对如何分区不是很了解，最好的方式是请教他人。如果请教不到他人的话，那么我建议你分出两个分区：“swap分区” 和 “ext4分区”。swap分区，它相当于Windows系统中虚拟内存，建议的大小是8G。而"ext4分区"挂载路径选择"根目录"，即"/"，该分区是用来存储操作系统。  
(02) 选择bootloader安装路径时，要选择整个硬盘，而不要选择硬盘中某个分区的节点。否则，进入系统后，bootloader不能被识别到。

**第8步：安装完毕，重启进入ubuntu系统。更新grub，将Window系统加入启动项：**

    sudo update-grub

 
至此，ubuntu已经彻底安装完毕！

**第9步：删除已经之前复制到C系统下的安装文件**

重启系统，进入系统，然后删除C盘下的文件(vmlinuz，initrd.lz，grldr，grldr.mbr，grub.exe，menu.lst，ubuntu-12.04.iso)。如果是Windows XP，则不能，删除boot.ini；如果是Windows 7，则可以删除boot.ini。

