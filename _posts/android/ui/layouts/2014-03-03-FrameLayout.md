---
layout: post
title: "Android布局之 FrameLayout"
description: "android layout"
category: android
tags: [android]
date: 2014-03-03 09:01
---

> 本文介绍FrameLayout布局。

> 点击查看：[示例源码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/ui/widgets/layouts/FrameLayout)

> **目录**  
[1. FrameLayout简介](#anchor1)  
[2. FrameLayout示例](#anchor2)  


<a name="anchor1"></a>
# 1. FrameLayout简介

对于FrameLayout，官方介绍是：

> FrameLayout is designed to block out an area on the screen to display a single item. Generally, FrameLayout should be used to hold a single child view, because it can be difficult to organize child views in a way that's scalable to different screen sizes without the children overlapping each other. You can, however, add multiple children to a FrameLayout and control their position within the FrameLayout by assigning gravity to each child, using the android:layout_gravity attribute.

即，设计FrameLayout是为了显示单一项widget。通常，不建议使用FrameLayout显示多项内容；因为它们的布局很难调节。  
但实际上，Android系统自身也经常使用FrameLayout来显示多项内容。在它显示多项内容时，内容会重叠；此时，需要根据自己的需求使用layout_gravity属性来设置FrameLayout中子视图的位置。


layout_gravity可以使用如下取值：



|             属性名称            |                          说明                       |
| ------------------------------- | --------------------------------------------------- |
| top | 将对象放在其容器的顶部，不改变其大小 |
| bottom | 将对象放在其容器的底部，不改变其大小 |
| left | 将对象放在其容器的左侧，不改变其大小 |
| right | 将对象放在其容器的右侧，不改变其大小 |
| center_vertical | 将对象纵向居中，不改变其大小. 垂直对齐方式 |
| fill_vertical | 必要的时候增加对象的纵向大小，以完全充满其容器 |
| center_horizontal | 将对象横向居中，不改变其大小 |
| fill_horizontal | 必要的时候增加对象的横向大小，以完全充满其容器 |
| center | 将对象横和纵都居中，不改变其大小 |
| fill | 必要的时候增加对象的横纵向大小，以完全充满其容器 |
| clip_vertical | 附加选项，用于按照容器的边来剪切对象的顶部和/或底部的内容. 剪切基于其纵向对齐设置：顶部对齐时，剪切底部；底部对齐时剪切顶部；除此之外剪切顶部和底部 |
| clip_horizontal | 附加选项，用于按照容器的边来剪切对象的左侧和/或右侧的内容. 剪切基于其横向对齐设置：左侧对齐时，剪切右侧；右侧对齐时剪切左侧；除此之外剪切左侧和右侧 |


注意: **区分"android:gravity" 和 "android:layout_gravity"**。  
android:gravity：是对控件本身来说的，是用来设置“控件自身的内容”应该显示在“控件自身体积”的什么位置,默认值是左侧。  
android:layout_gravity：是相对于控件的父元素来说的，设置该控件在它的父元素的什么位置。



<a name="anchor2"></a>
# 2. FrameLayout示例

下面通过两则示例来演示FrameLayout的基本用法。

## 2.1 示例一
### 示例说明

定义一个FrameLayout，它的宽和高都是match_parent。FrameLayout里面包含2个控件：  
(01) 包含一个TextView，背景是红色，文本比较长。它会被FrameLayout默认摆放在左上角位置。  
(02) 包含一个"确定"按钮，背景是绿色，内容比TextView短。它也会被FrameLayout默认摆放在左上角位置。  
实际效果是，Button覆盖在TextView上面！

### layout布局

    <?xml version="1.0" encoding="utf-8"?>
    <FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
        android:layout_width="match_parent"
        android:layout_height="match_parent" >

        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="我是TextView，内容比较长"
            android:background="#ff0000"/>
        <Button
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:background="#00ff00"
            android:text="确定"/>

    </FrameLayout>




## 2.2 示例二
### 示例说明

定义一个FrameLayout，它的宽和高都是match_parent。FrameLayout里面包含2个控件：  
(01) 包含一个TextView，内容是"文本居左"，背景是红色。它会被默认摆放在FrameLayout的左上角位置。  
(02) 包含一个TextView，内容是"文本居中"，背景是绿色。它会被默认摆放在FrameLayout的中间位置。  
实际效果是，Button覆盖在TextView上面！

### layout布局

    <?xml version="1.0" encoding="utf-8"?>
    <FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
        android:layout_width="match_parent"
        android:layout_height="match_parent" >

        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="文本居左"
            android:background="#ff0000"
            android:gravity="center"
            android:layout_gravity="left"
            android:layout_margin="10dp"/>
        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="文本居中"
            android:background="#00ff00"
            android:gravity="center"
            android:layout_gravity="center"/>

    </FrameLayout>




