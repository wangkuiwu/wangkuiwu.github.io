---
layout: post
title: "Linux学习基础篇02 ubuntu的配置(二) 设置默认开机选项"
description: "linux ubuntu"
category: linux
tags: [linux]
date: 2011-07-02 09:02
---

> 在ubuntu和Windows双系统中，默认的开机项是ubuntu系统。如果你想将将默认的开机选项该为Windows系统，可以通过以下步骤实现。

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

