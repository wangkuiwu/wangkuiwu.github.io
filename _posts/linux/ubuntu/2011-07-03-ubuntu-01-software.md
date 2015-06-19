---
layout: post
title: "Linux学习基础篇03 ubuntu软件(一) 必备软件"
description: "linux ubuntu"
category: linux
tags: [linux]
date: 2011-07-03 09:01
---

<a name="anchor1"></a>
# 1. apt-get安装常用软件

**(01) 安装VIM**：VIM是Linux下一款非常棒的编辑器。

    $ sudo apt-get install vim

**(02) 安装git**：git是最棒的版本管理工具，没有之一！

    $ sudo apt-get git

**(03) 安装gimp**：gimp是ubuntu下的图片编辑工具。

    $ sudo apt-get gimp

**(04) 安装smplayer**：smplayer是ubuntu下的一款视频播放器。

    $ sudo apt-get install smplayer



<a name="anchor2"></a>
# 2. 新立得/软件中心安装常用软件

## 2.1 浏览器chromium

打开Software Center之后，搜索Chromium。然后选择安装。

### 2.1.1 chromium安装flash player插件

    $ sudo apt-get install pepperflashplugin-nonfree
    $ sudo update-pepperflashplugin-nonfree --install

然后重启chromium

### 2.1.2 firefox安装flash player插件

**(1) 下载flash player对应的tar.gz包**

打开[https://get.adobe.com/flashplayer/](https://get.adobe.com/flashplayer/)，下载tar.gz包之后解压，得到libflashplayer.so和usr目录。

**(2) 将usr目录拷贝到/usr中**

    $ sudo cp -r usr/* /usr

**(3) 将libflashplayer.so拷贝到浏览器对应的plugin目录**

    $ sudo cp libflashplayer.so /usr/lib/mozilla/plugins/
    $ sudo cp libflashplayer.so /usr/lib/chromium-browser/plugins/

然后重启fire fox。

<a name="anchor3"></a>
# 3. 手动安装的软件

<a name="anchor3_1"></a>
## 3.1 安装中文输入法ibus

ubuntu中比较常用的输入法有ibus和fcitx。两种输入法，择其一安装即可！推荐使用ibus。


### 3.1.1 ibus的安装


**(01) 安装ibus以及拼音包**

    $ sudo apt-get install ibus ibus-clutter ibus-gtk ibus-gtk3 ibus-qt4
    $ sudo apt-get install ibus-pinyin

**(02) 安装中文语言包**

进入"System Settings"(系统设置) --> "Language Support"(语言支持)。  
在"Language Support"(语言支持)中找到"Install/Remove Languages(添加或删除语言)"，并点击打开"添加或删除语言对话框"。  
在对话框中找到"Chinese(simplified)"，即中文(简体)，进行安装。  
安装完毕之后，重启X系统 或 重启操作系统。

**(03) 设iBus为默认输入法**

进入到"Language Support"(语言支持)中，将"Keyboard input method system"(键盘输入方式系统)设为iBus。

**(04) 在iBus中添加拼音**

打开"iBus"，中文名是"键盘输入法"。方法是通过文件浏览器，进入目录/usr/share/applications。

在iBus设置中，将"汉语拼音"设为选中的输入法即可！

以上设置都完成之后， 重启X系统 或 重启操作系统，即可！


### 3.1.2 fcitx的安装

下面介绍ubuntu中fcitx输入法的安装步骤。

**第1步：更新安装程序** 

    $ sudo apt-get update

**第2步：安装fcitx**

    $ sudo apt-get install fcitx

**第3步：安装fcitx-config**

    $ sudo apt-get install fcitx-config

**第4步：设置fcitx为默认输入法**

    $ im-switch -s fcitx -z default

**第5步：安装其他码表**

    $ sudo apt-get install fcitx-table-all

安装完成之后，需要重启系统才能生效！

**第6步：重启系统**

    $ sudo reboot



<a name="anchor3_2"></a>
## 3.2 安装VirtualBox虚拟机

下面介绍ubuntu中安装VirtualBox虚拟机，并在虚拟机中安装Win 7操作系统的步骤。

**第1步：下载VirtualBox虚拟机**

参考网址：[https://www.virtualbox.org/wiki/Linux_Downloads](https://www.virtualbox.org/wiki/Linux_Downloads)

**第2步： 下载win7 32bit镜像**

百度网盘上。[点击下载](http://pan.baidu.com/s/1jGgkrRK)，密码：tc3x

**第3步： 安装VirtualBox**

**第4步： 在VirtualBox中安装win7镜像**


