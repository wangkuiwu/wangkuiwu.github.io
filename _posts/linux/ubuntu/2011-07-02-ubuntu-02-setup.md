---
layout: post
title: "Linux学习基础篇02 ubuntu的配置(一) 基本配置"
description: "linux ubuntu"
category: linux
tags: [linux]
date: 2011-07-02 09:01
---

> 前面一章，介绍了如何安装ubuntu系统。这一章，将介绍ubuntu相关的配置；我将ubuntu的配置分为两部分进行介绍：基本配置 和 高级配置。基本配置是日常使用到的配置，而高级配置则大多是我自己常用的配置总结。建议在初学的时候，先了解基本配置；而至于高级配置部分，随着后面对ubuntu的逐步了解，再来逐步学习！

> **目录**  
> [1. 开启root登录GUI系统](#anchor1)   
> [2. 设置默认开机选项](#anchor2)   


<a name="anchor1"></a>
# 1. 开启root登录GUI系统

ubuntu的GUI登录界面，默认是无法通过root直接登录的。如果你想直接通过root登录到ubuntu的GUI系统中，通过以下操作即可实现。

## 1.1 设置root账户和密码

**第一步：用普通账户登录。**

**第二步：登录之后，通过以下命令设置root账户密码：**

    $ sudo passwd root

**第三步：设置root密码成功后，在终端可以通过以下命令切换到root：**

    $ su
或

    $ su -

 
## 1.2 开启root登录界面

**第一步：编辑配置文件lightdm.conf，开启登录界面输入选项。**

编辑文件/etc/lightdm/lightdm.conf，可以采用以下命令：

    $ sudo gedit /etc/lightdm/lightdm.conf

在文件末尾，添加以下内容：

    greeter-show-manual-login=true

**第二步：通过reboot重启电脑**

    $ sudo reboot

**第三步：在登录界面就输入用户名(root)和密码进行登录了。**

 

<br/>

<a name="anchor2"></a>
# 2. 设置默认开机选项

在ubuntu和Windows双系统中，默认的开机项是ubuntu系统；如果你想将将默认的开机选项该为Windows系统，可以通过以下步骤实现。

**第一步：添加配置文件可写权限。**

    $ sudo chmod +w /boot/grub/grub.cfg

**第二步：进入ubuntu系统，编辑以下文件：**

    $ sudo gedit /boot/grub/grub.cfg

**第三步：将grub.cfg中的default选项设置“Windows对应的选项数值”。**

修改前的内容(默认第0项，即默认Ubuntu)：

    set default="0"

 修改后的内容(假设Windows系统是第项)：

    set default="4"

*注意：grub.cfg中每出现一个menuentry，就表示一项。起始项对应的序号是"0"，计算出Windows系统的序号，并将其赋值给default即可！*

**第四步：去掉配置文件的可写权限。**

    $ sudo chmod -w /boot/grub/grub.cfg

这样，就完成了设置！ 重启系统，默认就是进入windows系统。

