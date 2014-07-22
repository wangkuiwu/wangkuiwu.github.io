---
layout: post
title: "Android反编译"
description: ""
category: gradle
tags: [gradle]
date: 2014-07-21 09:25
---

> 本文介绍ubuntu下APK反编译的相关内容。包括：反编译工具下载和环境搭建，反编译的详细步骤。


<a name="anchor1"></a>
# 资源下载和环境搭建

1 **点击下载**[android_decompile_tools_ForUbuntu_201309.zip](https://github.com/wangkuiwu/android_applets/blob/master/tools/android_decompile_tools_ForUbuntu_201309.zip?raw=true)


2 **通过以下命令解压资源包**

    $ unzip android_decompile_tools_ForUbuntu_201309.zip

若zip解压工具没有安装，可以通过以下命令安装：

    $ sudo apt-get install unzip 

解压后得到4个文件:  
> apktool1.5.2.tar.bz2  
> dex2jar-0.0.9.15.zip  
> jad158e.linux.intel.zip  
> libstdc++2.10-glibc2.2_2.95.4-27_i386.deb


3 **解压apktool1.5.2.tar.bz2**

apktool1.5.2.tar.bz2 是“反编译apk资源”工具，即反编译apk的manifest和res目录。通过以下命令将文件解压到当前目录：

    $ tar -xvf apktool1.5.2.tar.bz2

解压后得到文件 “apktool1.5.2/apktool.jar”。

补充：这里使用的是1.5.2版本的apktool工具。若要下载最新版本的apktool工具，点击链接：[http://code.google.com/p/android-apktool/](http://code.google.com/p/android-apktool/)


4 **解压dex2jar-0.0.9.15.zip**

dex2jar-0.0.9.15.zip 是“将dex转换为jar”工具。 通过以下命令将文件解压到当前目录：

    $ unzip dex2jar-0.0.9.15.zip

解压后得到目录 “dex2jar-0.0.9.15”。


补充：这里使用的是0.0.9.15版本的dex2jar工具。若要下载最新版本的dex2jar工具，点击链接： [http://code.google.com/p/dex2jar/](http://code.google.com/p/dex2jar/)


5 **解压 jad158e.linux.intel.zip**

jad158e.linux.intel.zip 是“将jar还原成java源文件”的工具。通过以下命令将文件解压到当前目录：

    $ unzip jad158e.linux.intel.zip

解压后，得到文件 “jad”。

补充：这里使用的是jad158e版本工具。若要下载最新版本的jad工具，点击链接： [http://www.varaneckas.com/jad](http://www.varaneckas.com/jad)


6 **安装 libstdc++2.10-glibc2.2_2.95.4-27_i386.deb**

到 [http://archive.debian.net/etch/i386/libstdc++2.10-glibc2.2/download](http://archive.debian.net/etch/i386/libstdc++2.10-glibc2.2/download) 中下载一个文件，然后通过以下命令安装deb文件：

    $ dpkg -i libstdc++2.10-glibc2.2_2.95.4-27_i386.deb

补充：若不执行这一步。在运行命令"$ ./jad "时，会报“libstdc++-libc6.2-2.so.3 not found”错误！


7 **将反编译工具添加到ubuntu环境变量中**

a) 假设第3步中得到的“apktool.jar”目录的路径为：      /home/skywang/android/decompile/apktool1.5.2/apktool.jar  
b) 假设第4步中得到的“dex2jar-0.0.9.15”目录的路径为： /home/skywang/android/decompile/dex2jar-0.0.9.15  
c) 假设第5步中得到的“jad”文件的路径为：              /home/skywang/android/decompile/jad  

编辑 ~/.bashrc ，添加以下内容：  
> # Android Decompile tools  
> export APKTOOL_PATH=/home/skywang/android/decompile/apktool1.5.2/apktool.jar  
> export JAD_PATH=/home/skywang/android/decompile  
> export DEX2JAR_PATH=/home/skywang/android/decompile/dex2jar-0.0.9.15  
> export PATH=$PATH:$JAD_PATH:$DEX2JAR_PATH

其中，APKTOOL_PATH是“apktool.jar文件”的路径，JAD_PATH是“jad文件”所在目录，DEX2JAR_PATH是“dex2jar-0.0.9.15目录”的路径，PATH是ubuntu自带的环境变量。

设置完毕之后，重新导入环境变量：

    $ source ~/.bashrc

若重新打开“终端”窗口，则不需要执行“$ source ~/.bashrc”。

补充：可以通过以下命令检查环境变量是否设置成功。

    $ echo $APKTOOL_PATH
    $ echo $PATH

显示的内容包括你所添加的路径，则说明环境变量设置成功！

工具和环境都设置ok之后，接下来讲解反编译步骤。反编译包括2部分：反编译资源 和 反编译源码。




<a name="anchor2"></a>
# 反编译步骤

## 1. 反编译“apk资源”

反编译“apk资源”，会得到“manifest文件”和“res目录”。

### 1.1 准备apk资源

   可以通过eclipse新建一个简单的Android工程；然后编译运行。此时，在工程的bin目录下会自动生成一个apk文件。

### 1.2 反编译资源

执行以下命令：

    $java -jar $APKTOOL_PATH d [apk_path.apk]

说明：  
(01) APKTOOL_PATH 是“apktool.jar路径”对应的环境变量。前面我们已经在~/.bashrc中进行了设置。  
(02) apk_path.apk 是“apk文件所在的路径”。  
上诉命令的作用就是：将apk反编译，得到的“manfiest”和“res目录”都是原始资源文件。而“smali”并不是我们需要的代码，我们需要按照下面的步骤来反编译代码。




## 2. 反编译“apk源码”

### 2.1 解压apk文件

通过以下命令解压apk文件到当前目录：

    $ unzip [apk_path.apk]

说明：  
(01) apk_path.apk 是“apk文件所在的路径”。  
(02) apk本质上是一个zip格式的压缩文件，所以可以用unzip命令解压缩。 此外，我们可以通过"$ file [apk_path.apk]"来查看apk的文件格式。  
上诉命令的作用是：解压apk，得到“classes.dex”文件。除了“classes.dex”之外的其它文件，都可以删除。


### 2.2 将dex转换为jar文件

执行以下命令：

    $ d2j-dex2jar.sh classes.dex

说明：上诉命令的作用是：在当前目录，生成一个“classes_dex2jar.jar”文件。


### 2.3 解压classes_dex2jar.jar文件

新建目录"folder"，并将classes_dex2jar.jar解压到该目录下。执行以下命令：

    $ mkdir folder
    $ unzip classes_dex2jar.jar -d folder

说明：  
(01) mkdir 的作用是，新建目录folder。  
(02) unzip 的作用是，将classes_dex2jar.jar解压到folder目录下。


### 2.4 将class转为java文件

执行以下命令：

    $ jad -o -r -sjava -dsrc folder/**/*.class

说明：执行上诉命令，会在当前目录生成“src”目录，并且“src”中的全部文件都是java源文件。

至此，就完成了“反编译得到java源文件”的工作！




