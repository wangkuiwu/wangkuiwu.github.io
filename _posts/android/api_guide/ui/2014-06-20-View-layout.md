---
layout: post
title: "Android API指南(二)自定义控件04之 位置说明"
description: "android training"
category: android
tags: [android]
date: 2014-06-20 12:11
---


> 本文介绍位置信息。点击查看：[位置信息的测试源码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/ui/self_view/04_position/ViewTest)


<a name="anchor1"></a>
# 1. android:padding和android:layout_margin

**android:layout_margin**：指该控件距离边父控件的边距。  
**android:layout_marginLeft**：指该控件的左边缘距离边父控件的边距。

**android:padding**：指该控件内容距离控件边缘的距离；例如，TextView中的文本距离TextView自身的边距。  
**android:paddingLeft**：指该控件内容的左边缘距离控件的距离；例如，TextView中的文本的左边缘距离TextView自身的边距。

## 示例

    <com.skw.viewtest.MyView
        android:layout_width="100dip"
        android:layout_height="100dip"
        android:padding="10dip"
        android:layout_margin="20dip"
        />

说明：MyView是一个自定义视图。  
(01) android:padding="10dip"，意味着它的内容距离MyView视图本身的上下左右边距都是10dip。  
(02) android:layout_margin="20dip"，意味着MyView距离"它的父容器"的上下左右边距都是20dip。  



# 2. View的相关位置

View的位置涉及到TransformationInfo。TransformationInfo记录的是在View发生"缩放/旋转"等变化时的大小，如果没有发生"缩放/旋转"，则TransformationInfo记录的位置信息都是0。

关于水平方向的位置：  
**getPaddingLeft()**：指该控件内容的左边缘距离控件自身的距离。   
**getLeft()**： 该View相对于"父view"的x坐标。  
**getX()**：TransformationInfo.mTranslationX + getLeft()的值。

同理，  
**getPaddingLeft()**：指该控件内容的上边缘距离控件自身的距离。   
**getTop()**： 该View相对于"父view"的y坐标。  
**getY()**：TransformationInfo.mTranslationY + getTop()的值。




# 3. MotionEvent点击事件的相关位置

**getX()**： 触摸点相对于"监听该点击事件的控件"的x坐标。  
**getRawX()**：触摸点相对于屏幕的x坐标。

同理，  
**getY()**： 触摸点相对于"监听该点击事件的控件"的y坐标。  
**getRawY()**：触摸点相对于屏幕的y坐标。
