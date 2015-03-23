---
layout: post
title: "Android培训(一)开始篇03之 APK支持不同的设备"
description: "android training"
category: android
tags: [android]
date: 2014-01-03 09:25
---

> 本章介绍在开发APK中，支持多种设备需要考虑的几个方面。主要包括：多语言，多分辨率和多Android版本。

> **目录**  
[1. 多语言](#anchor1)  
[2. 多分辨率](#anchor2)  
[3. 多Android版本](#anchor3)  


<a name="anchor1"></a>
# 1. 多语言

Android支持多语言。在APK提供多语言支持时，只需要定义不同语言对应的资源；然后放到相应目录即可。

例如，假如要提供支持：英文、简体中文、繁体中文。则提供以下三个目录下的资源即可：

    res/values/strings.xml
    res/values-zh-rCN/strings.xml
    res/values-zh-rTW/strings.xml

说明：请参考"[Android多语言示例](https://github.com/wangkuiwu/android_applets/tree/master/training/01_getting_started/03_different_devices/01_languages/app)"。下载安装该apk，然后将设备的语言分别设置到"英文/简体中文/繁体中文"查看效果--它会根据语言显示对应的文本。




<a name="anchor2"></a>
# 2. 多分辨率

Android支持多分辨率。在APK提供多分辨率支持时，也只需要定义对应分辨率的资源；然后放到相应目录即可。


## 2.1 分辨率常识

android中通过"大小(size)"和"密度(density)"来决定资源的大小。

android包含4种基本的大小：**samll, normal, large, xlarge**。  
这4种大小对应的密度是：**ldpi, mdpi, hdpi, xhdpi**。


如何区别android设备到底是属于哪一种大小呢？下面是参考标准：

    xlarge : 不小于 960dp x 720dp  
    large  : 不小于 640dp x 480dp  
    normal : 不小于 470dp x 320dp  
    small  : 不小于 426dp x 320dp  

实际上，android设备的大小没有非常明确的界限。典型的例子，就是800x400；该分辨率的设备可能是large，也可能是normal。 



## 2.2 资源布局

将对应的资源文件放在"<screen_size>"为后缀的目录下，android设备会根据它自身的大小来决定使用哪种资源。


### 例1. layout布局

下面layout布局文件的定义：

    res/layout/main.xml                 // 默认布局
    res/layout-large/main.xml           // large设备对应的布局
    res/layout-xlarge/main.xml          // xlarge设备对应的布局
    res/layout-xlarge-land/main.xml     // xlarge并且是横屏设备对应的布局


### 例2. 图片布局

不同分辨率对应的图片大小需求也不一样。例如，large需要的图片可能比normal需要的图片要大。  
下面是不同屏幕所需要图片的缩放尺寸：  

    xhdpi: 2.0  
    hdpi : 1.5  
    mdpi : 1.0 (baseline)  
    ldpi : 0.75  


当apk需要支持不同分辨率并用到图片时，将图片放到对应的目录即可。如下示例：

    drawable-xhdpi/awesomeimage.png     // xlarge对应的图片
    drawable-hdpi/awesomeimage.png      // large对应的图片
    drawable-mdpi/awesomeimage.png      // normal对应的图片
    drawable-ldpi/awesomeimage.png      // small对应的图片




<a name="anchor3"></a>
# 3. 多Android版本

Android有许多不同版本。有些Android功能，在有些版本才添加；因此apk需要对此进行相应的处理：可以指定apk支持的最低版本的Android。

具体做法是在manifest中添加"android:minSdkVersion"来做到。除此之外，还可以指定当前apk的目标版本(即，该apk在哪个版本上开发的)。示例如下：

    <manifest xmlns:android="http://schemas.android.com/apk/res/android" ... >
        <uses-sdk android:minSdkVersion="11" android:targetSdkVersion="19" />
        ...
    </manifest>

说明：Android的版本可以在[Android SDK版本](http://developer.android.com/intl/zh-cn/guide/topics/manifest/uses-sdk-element.html)中查询。  
(01) minSdkVersion=11，即对应API level 11的android版本，也就是Andrid3.0。  
(02) targetSdkVersion="19"，即对应API level 19的android版本，也就是Android4.4。  


