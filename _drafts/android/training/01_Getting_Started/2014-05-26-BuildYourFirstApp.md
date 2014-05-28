---
layout: post
title: "Android培训(一)开始篇01之 创建第一个程序"
description: "android training"
category: android
tags: [android]
date: 2014-05-26 09:25
---

> 本章介绍如何在命令行中创建Android工程。

> **目录**  
> **1**. [创建和运行项目](#anchor1)  


<a name="anchor1"></a>
# 创建和运行项目

下面介绍Linux系统下，通过命令行创建项目的方式。

## 1. 工具下载

下载"[Android开发工具][link_android_download_index]"。



## 2. 环境变量设置

(01) 将sdk中的tools和platform-tools添加到系统的环境变量中。如果是Ubuntu系统，则在.bashrc中添加以下类似于内容：

    SDK_PATH=/home/skywang/opt/adt-bundle-linux-x86_64-20130522/sdk
    export PATH=$PATH:$SDK_PATH/tools:$SDK_PATH/platform-tools

注意：SDK_PATH是android的sdk路径。  
tools是google提供的工具包，包括创建android工程的工具：tools/android。  
platform-tools是平台工具，主要用到的是adb。


(02) 安装ant，它是编译android工程以及打包apk的。

    $ sudo apt-get install ant




## 3. 创建Android工程

将tools添加到环境变量之后，重启终端。然后就能通过以下指令创建工程了。

    android create project \
    --target <target_ID> \
    --name <your_project_name> \
    --path path/to/your/project \
    --activity <your_activity_name> \
    --package <your_package_namespace>

说明：  
(01) target 是目标。可以通过"android list targets"来查看目标。通常target对应的target_ID是1。  
(02) name 是App名。  
(03) path 是工程路径。  
(04) activity 是默认的Activity名。  
(05) package 是包名。  


<br/>
示例：

    android create project \
    --target 1 \
    --name MyAndroidApp \
    --path ./MyAndroidAppProject \
    --activity MyAndroidAppActivity \
    --package com.example.myandroid


## 4. 编译

在工程的根目录(MyAndroidAppProject)下执行指令"**ant debug**"就可以编译工程。

**ant clean**: 清空工程。即删除所有编译生成的文件。  
**ant installd**: 安装一个已经编译的debug工程。类似于"adb install bin/MyAndroidApp-debug.apk"。  





[link_android_download_index]: http://developer.android.com/intl/zh-cn/tools/index.html
