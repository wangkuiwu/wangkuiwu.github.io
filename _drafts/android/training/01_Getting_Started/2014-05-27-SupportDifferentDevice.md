---
layout: post
title: "Android培训(一)开始篇03之 APK支持不同的设备"
description: "android training"
category: android
tags: [android]
date: 2014-05-27 09:25
---

> 本章介绍在开发APK中，支持多种设备需要考虑的几个方面。主要包括：多语言，多分辨率和多Android版本。

> **目录**  
> **1**. [多语言支持](#anchor1)  


<a name="anchor1"></a>
# 多语言支持

提供多语言支持，定义不同语言对应的资源；然后放到相应目录即可。

例如，假如要提供支持：英文、简体中文、繁体中文。则提供以下三个目录下的资源即可：

    res/values/strings.xml
    res/values-zh-rCN/strings.xml
    res/values-zh-rTW/strings.xml




# 不同分辨率


## 1. 大小和密度

android中通过"大小(size)"和"密度(density)"来决定资源的大小。

android包含4种基本的大小：**samll, normal, large, xlarge**。
这4种大小对应的密度是：**ldpi, mdpi, hdpi, xhdpi**。


<br/>
如何区别android设备到底是属于哪一种大小呢？

xlarge screens are at least 960dp x 720dp
large screens are at least 640dp x 480dp
normal screens are at least 470dp x 320dp
small screens are at least 426dp x 320dp



## 2. 资源布局

将对应的资源文件放在"<screen_size>"为后缀的目录下，android设备会根据它自身的大小来决定使用哪种资源。


### 例1. layout布局

以layout布局文件来说，假如有以下两个布局文件：

    res/layout/main.xml
    res/layout-large/main.xml

说明：layout是默认布局，而layout-large是大尺寸(large)设备对应的布局。


android机器有许多不同分辨率；不同分辨率的设备在选择资源文件时，会在res目录下查找对应



[link_android_download_index]: http://developer.android.com/intl/zh-cn/tools/index.html
[link_grade_download]: http://www.gradle.org/downloads
[link_grade_ver112]: https://services.gradle.org/distributions/gradle-1.12-all.zip
