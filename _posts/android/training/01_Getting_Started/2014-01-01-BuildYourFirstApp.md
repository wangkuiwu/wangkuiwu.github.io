---
layout: post
title: "Android培训(一)开始篇01之 创建第一个程序"
description: "android training"
category: android
tags: [android]
date: 2014-01-01 09:01
---

> 本章以ubuntu系统位前提，先介绍ubuntu中Java和Android开发环境的搭建方法；然后，再说明如何通过ant来调试Android工程。

> **目录**  
[1. 搭建Java环境](#anchor1)  
[2. 搭建Android环境](#anchor2)  
[3. 使用Ant调试Android工程](#anchor3)  





<a name="anchor1"></a>
# 1. 搭建Java开发环境

这里介绍"**ubuntu系统**"下安装Java开发环境的方式。

**第一步**：下载JDK

到[甲骨文官网](http://www.oracle.com/technetwork/java/javase/downloads/index.html)上去下载JDK。

注意：选择JDK时，要注意"操作系统"、"32位/64位"、"安装方式"几个方面。  
(01) 操作系统：对于ubuntu系统，请选择linux。  
(02) 32位/64位：如果是系统32位请选择x86，是64位则选择x64。  
(03) 安装方式：对于ubuntu系统，请选择.tgz包。  

说明：我的是"ubuntu12.04 64位"操作系统，选择的是"[jdk-8u5-linux-x64.tar.gz](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)"包。


**第二步**：安装JDK包

由于安装的JDK包是压缩包，直接将其解压即可。

以jdk-8u5-linux-x64.tar.gz为例，将其解压到~/opt目录(如~/opt不能存在，则新建该目录)。

    $ tar -xvf jdk-8u5-linux-x64.tar.gz -C ~/opt/

说明：解压后在~/opt/下等到JDK目录jdk1.8.0_05。


**第三步**：设置JDK环境变量

安装完JDK包之后，将JDK的路径添加到环境变量中。在~/.bashrc中添加以下内容：


    # 设置JDK环境变量
    export JAVA_HOME=/home/skywang/opt/jdk1.8.0_05
    export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
    export PATH=$PATH:$JAVA_HOME/bin

说明：请将/home/skywang/opt/jdk1.8.0_05修改为你PC上JDK的路径。



**第四步**：验证

设置完毕之后，可以通过"java -version"指令来验证是否成功。

以ubuntu来说，设置完之后，**重新打开一个终端**；否则输入`java -version`。如果能正常输出JDK版本号，则意味着Java开发环境配置成功！





<a name="anchor2"></a>
# 2. 搭建Android开发环境

前面搭建好Java开发环境之后，接下来搭建Android开发环境。

总的来说，Android开发环境主要包括两大类：**使用Eclipse开发** 和 **使用其它工具**。使用Eclipse的好处是上手快，而且Eclipse是一个IDE，它集成了大量的工具。但对于习惯使用vim或emacs，更倾向于直接使用vim或emacs等编辑器进行开发；然后通过ant或gradle等工具打包apk。

由于我个人更喜欢使用vim。所以，本文介绍的是在ubuntu系统下面，搭建命令行的Android开发环境。


**第一步**：下载ADT包

下载"[Android开发工具](http://developer.android.com/intl/zh-cn/tools/index.html)"。

搭建Android环境时，请选择ADT，而不是单纯的SDK。原因很简单，ADT是"Ecplise+SDK"的组合，它包行了SDK；虽然我们是通过命令行开发，但保不准哪一天也会用到Eclipse！

说明：我的PC是ubuntu 64位系统，下载的是"[adt-bundle-linux-x86_64-20140321.zip](http://dl.google.com/android/adt/22.6.2/adt-bundle-linux-x86_64-20140321.zip)"



**第二步**：解压ADT包

将下载的ADT包解压到~/opt目录(如~/opt不能存在，则新建该目录)。

    $ unzip adt-bundle-linux-x86_64-20140321.zip -d ~/opt/

说明：解压后在~/opt/下得到目录adt-bundle-linux-x86_64-20140321，在adt-bundle-linux-x86_64-20140321中有"eclipse"和"sdk"两个目录。



**第三步**：设置SDK环境变量


将sdk中的tools和platform-tools添加到系统的环境变量中。在.bashrc中添加以下内容：

    ANDROID_HOME=/home/skywang/opt/adt-bundle-linux-x86_64-20140321/sdk
    export PATH=$PATH:$ANDROID_HOME/tools:$ANDROID_HOME/platform-tools

说明：  
(01) ANDROID_HOME是android的sdk路径。  
(02) sdk的tools是google提供的工具包，它包括创建android工程的工具android等。  
(03) platform-tools是平台工具，主要用到的工具是adb。



**第四步**：验证

将sdk添加到环境变量之后，重新打开终端以便导入新的环境变量。接着，就使用sdk中的工具来创建Android工程了。

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
说明：直接使用以下使用就可以创建一个Android工程。

    android create project \
    --target 1 \
    --name MyAndroidApp \
    --path ./MyAndroidAppProject \
    --activity MyAndroidAppActivity \
    --package com.example.myandroid

如果当前目录下就出现了一个名称为MyAndroidApp的工程，那么恭喜你：Android的开发环境搭建成功了！




<a name="anchor3"></a>
# 3. 使用Ant调试Android工程

下面介绍使用Ant编译/调试APK的方法

**第一步** 安装ant

在ubuntu系统下，直接使用以下指令安装ant：

    $ sudo apt-get install ant



**第二步** 使用ant编译工程

以MyAndroidApp工程为例，切换到工程的根目录，然后指令"ant debug"就可以编译工程了。

    $ cd ~/.../MyAndroidApp
    $ ant debug

说明：ant debug是作用是编译工程，并生成debug版本的apk；如果编译成功，就会在工程的bin目录下生成一个`<apk-name>-debug.apk`的程序。


**第三步** 安装apk

如果当前有adb设备的话，可以通过`adb devices`来确认。可以通过"ant installd"来将apk安装到设备上。

    $ ant installd

说明：ant installd是安装一个已经编译完成的debug版本的apk。类似于"adb install ...apk"。  


**ant debug**: 编译工程，并生成debug版本的apk。  
**ant clean**: 清空工程。即删除所有编译生成的文件。  
**ant installd**: 安装一个已经编译的debug工程。类似于"adb install bin/MyAndroidApp-debug.apk"。  




