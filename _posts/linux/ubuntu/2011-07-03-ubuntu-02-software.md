---
layout: post
title: "Linux学习基础篇03 ubuntu软件(二) JDK"
description: "linux ubuntu"
category: linux
tags: [linux]
date: 2011-07-03 09:02
---

> 本文介绍java jdk环境的搭建方式


**1.1 下载JDK**

下面以jdk1.7.0_75来进行说明(其他版本的jdk方法与此类似)。

到[JDK官方网站](http://www.oracle.com/technetwork/java/javase/downloads/jdk7-downloads-1880260.html) 中去下载jdk1.7.0_75。

注意：   
(01) 下载时，要下载与系统匹配的jdk。例如，32位的ubuntu选择linux..._x86版本进行下载，而64位的ubuntu则选择linux..._x64版本进行下载。   
(02) 下载时，请尽量选择压缩包。本文的示例是jdk-7u75-linux-i586.tar.gz。  
如果是32位的jdk1.7.0_75，可以通过[网络文件](http://pan.baidu.com/s/1sjBd98p)来下载。


**1.2 安装JDK**

将jdk-7u75-linux-i586.tar.gz解压到~/opt目录下。如果不存在~/opt，则新建该目录。

    $ mkdir  ~/opt
    $ tar -xvf jdk-7u75-linux-i586.tar.gz -C ~/opt

解压之后的jdk就在~/opt/jdk1.7.0_75/目录中。


**1.3 配置JDK**

编辑~/.bashrc，添加以下内容。

    export JAVA_HOME=/home/skywang/opt/jdk1.7.0_75
    export JRE_HOME=$JAVA_HOME/jre
    export CLASSPATH=$CLASSPATH:$JAVA_HOME/lib:$JRE_HOME/lib
    export PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin

其中，JAVA_HOME就是解压得到的jdk的目录！

**1.4 验证JDK**

    $ java -version

如果该指令能正确的输出jdk版本，则表示JDK配置成功。

