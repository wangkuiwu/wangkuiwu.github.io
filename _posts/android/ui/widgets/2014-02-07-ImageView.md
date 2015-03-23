---
layout: post
title: "Android控件篇07之 ImageView"
description: "android training"
category: android
tags: [android]
date: 2014-02-07 09:11
---


<a name="anchor1"></a>
# 1. ImageView介绍

ImageView是图片显示控件，专门用来显示图片的。

ImageView的scaleType说明如下表

|     代码值         |        属性值         |              说明             |
| ------------------ | --------------------- | ----------------------------- |
| CENTER | center | 以原图的几何中心点和ImagView的几何中心点为基准,按图片的原来size居中显示，不缩放，当图片长/宽超过View的长/宽，则截取图片的居中部分显示ImageView的size.当图片小于View 的长宽时，只显示图片的size,不剪裁。 |
| CENTER_CROP | centerCrop | 以原图的几何中心点和ImagView的几何中心点为基准,按比例扩大(图片小于View的宽时)图片的size居中显示，使得图片长 (宽)等于或大于View的长(宽),并按View的大小截取图片。当原图的size大于ImageView时，按比例缩小图片，使得长宽中有一向等于ImageView,另一向大于ImageView。实际上，使得原图的size大于等于ImageView |
| CENTER_INSIDE | centerInside | 以原图的几何中心点和ImagView的几何中心点为基准，将图片的内容完整居中显示，通过按比例缩小原来的size使得图片长(宽)等于或小于ImageView的长(宽) |
| FIT_CENTER | fitCenter | 把图片按比例扩大(缩小)到View的宽度，居中显示 |
| FIT_END | fitEnd | 把图片按比例扩大(缩小)到View的宽度，显示在View的下部分位置 |
| FIT_START | fitStart | 把图片按比例扩大(缩小)到View的宽度，显示在View的上部分位置 |
| FIT_XY | fitXY | 把图片按照指定的大小在View中显示，拉伸显示图片，不保持原比例，填满View |
| MATRIX | matrix | 用matrix来绘制 |

例如：  
MATRIX对应的设置代码是**setType(ImageView.ScaleType.MATRIX)**，  
对应的属性是**android:scaleType="matrix"**   
对应的ImageView会用matrix来进行绘制。

