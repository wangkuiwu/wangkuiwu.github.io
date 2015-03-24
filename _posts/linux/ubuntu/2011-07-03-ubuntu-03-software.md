---
layout: post
title: "Linux学习基础篇03 ubuntu软件(三) 串口工具minicom"
description: "linux ubuntu"
category: linux
tags: [linux]
date: 2011-07-03 09:03
---

> **目录**  
> [1. 安装minicom](#anchor1)   
> [2. 配置minicom](#anchor2)   
> [3. minicom常用功能](#anchor3)   


<a name="anchor1"></a>
# 1. 安装minicom

    $ sudo apt-get install minicom


<a name="anchor2"></a>
# 2. 配置minicom


**2.1 启动minicom**

    $ sudo minicom
 
**2.2 启动并配置minicom**

    $ sudo minicom -s



<a name="anchor3"></a>
# 3. minicom常用功能

**3.1 开启换行功能**

    $ sudo minicom -w


**3.2 lrz串口传输公呢**

启动minicom之后，Ctrl + A, Z

