---
layout: post
title: "Linux学习基础篇03 ubuntu软件(四) terminator"
description: "linux ubuntu"
category: linux
tags: [linux]
date: 2011-07-03 09:04
---

> **目录**  
> [1. 安装terminator](#anchor1)   
> [2. 配置terminator](#anchor2)   


<a name="anchor1"></a>
# 1. 安装terminator

    $ sudo apt-get terminator


<a name="anchor2"></a>
# 2. 配置terminator

安装之后，若出现快捷键(ctrl+alt+t)是打开terminator，而不是默认的gnome terminor；则可以通过以下方式设置。

### 2.1 设置终端快捷键

可以通过dconf-tools来修改ubuntu的默认设置。

    $ sudo apt-get install dconf-tools
    $ dconf-editor

使用dconf-editor打开dconf-tools，进入目录 org  -->  gnome  -->  desktop  -->  applications  --> terminal

将结果改为

    exec  gnome-terminal
    exec-arg -x

这样，默认快捷键的终端为系统终端。要想默认为terminator，可以更改为

    exec  x-terminal-emulator
    exec-arg -e
