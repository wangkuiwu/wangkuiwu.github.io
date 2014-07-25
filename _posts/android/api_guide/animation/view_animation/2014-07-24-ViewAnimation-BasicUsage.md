---
layout: post
title: "Android 之Animation动画(六)之 View Animation"
description: "android training"
category: android
tags: [android]
date: 2014-07-25 09:00
---

> 本章介绍Android中的View Animation。

> **目录**  
> **1**. [View Animation简介](#anchor1)  
> **2**. [View Animation的语法规则](#anchor2)  
> **3**. [View Animation的示例](#anchor3)  


<a name="anchor1"></a>
# View Animation简介

View Animation(Tween Animation): 补间动画，给出两个关键帧，通过一些算法将给定属性值在给定的时间内在两个关键帧间渐变。

View Animation只能用来实现四种基本动作：透明/伸缩/移动/旋转。它与"Property Animation适用于任意Object类型不同"，View Animation只适用于View。但是View Animation相比于Property Animation的使用更加简单。


<a name="anchor2"></a>
# View Animation的语法规则

## 1. View Animation的样式

View Animation通常在**res/anim**目录下新建一个xml文件来定义。xml文件的格式如下：

    <?xml version="1.0" encoding="utf-8"?>
    <set xmlns:android="http://schemas.android.com/apk/res/android"
        android:interpolator="@[package:]anim/interpolator_resource"
        android:shareInterpolator=["true" | "false"] >
        <alpha
            android:fromAlpha="float"
            android:toAlpha="float" />
        <scale
            android:fromXScale="float"
            android:toXScale="float"
            android:fromYScale="float"
            android:toYScale="float"
            android:pivotX="float"
            android:pivotY="float" />
        <translate
            android:fromXDelta="float"
            android:toXDelta="float"
            android:fromYDelta="float"
            android:toYDelta="float" />
        <rotate
            android:fromDegrees="float"
            android:toDegrees="float"
            android:pivotX="float"
            android:pivotY="float" />
        <set>
            ...
        </set>
    </set>

说明：   
(01) **set**: 是动画的集合，相当于一个容器。  
(02) **interpolator**: 动画的动作类型，比如accelerate_interpolator类型的动画是加速的，它会越来越快。  
(03) **shareInterpolator**: 将set的interpolator应用到set所包行的动画中。  
(04) **alpha**: 透明度。  
&nbsp;&nbsp; a) **fromAlpha**: 起始动画的透明度。它的值是0~1.0之间；0表示完全透明，1.0表示完全不透明。  
&nbsp;&nbsp; b) **toAlpha**: 结束动画的透明度。它的值是0~1.0之间；0表示完全透明，1.0表示完全不透明。  
(05) **scale**: 缩放。  
&nbsp;&nbsp; a) **fromXScale**: 起始动画在X轴上的缩放倍数。0表示没有，2表示2倍。  
&nbsp;&nbsp; b) **toXScale**: 结束动画在X轴上的缩放倍数。0表示没有，2表示2倍。  
&nbsp;&nbsp; c) **fromYScale**: 起始动画在Y轴上的缩放倍数。0表示没有，2表示2倍。  
&nbsp;&nbsp; d) **toYScale**: 结束动画在Y轴上的缩放倍数。0表示没有，2表示2倍。  
&nbsp;&nbsp; e) **pivotX**: 动画缩放时，中心点在X轴上的位置(相对于原始的视图)。50%表示在视图的X轴中间。  
&nbsp;&nbsp; f) **pivotY**: 动画缩放时，中心点在Y轴上的位置(相对于原始的视图)。50%表示在视图的Y轴中间。  
(06) **training**: 移动。  
&nbsp;&nbsp; a) **fromXDelta**: 起始动画在X轴上的偏移像素(相对于视图左上角)。  
&nbsp;&nbsp; b) **toXDelta**: 结束动画在X轴上的偏移像素(相对于视图左上角)。  
&nbsp;&nbsp; c) **fromYDelta**: 起始动画在Y轴上的偏移像素(相对于视图左上角)。  
&nbsp;&nbsp; d) **toYDelta**: 结束动画在Y轴上的偏移像素(相对于视图左上角)。  
(07) **rotate**: 旋转。  
&nbsp;&nbsp; a) **fromDegrees**: 起始动画在的角度。可以是负数，也可以大于360。  
&nbsp;&nbsp; b) **toDegrees**: 起始动画在的角度。可以是负数，也可以大于360。  
&nbsp;&nbsp; c) **pivotX**: 动画旋转时，中心点在X轴上的位置(相对于原始的视图)。50%表示在视图的X轴中间。  
&nbsp;&nbsp; d) **pivotY**: 动画旋转时，中心点在Y轴上的位置(相对于原始的视图)。50%表示在视图的Y轴中间。  
rotate既可以顺时针旋转，也可以逆时针旋转。 

## 2. interpolator的样式

系统自带的interpolator样式如下：

@android:anim/accelerate_decelerate_interpolator
@android:anim/accelerate_interpolator
@android:anim/anticipate_interpolator
@android:anim/anticipate_overshoot_interpolator
@android:anim/bounce_interpolator
@android:anim/cycle_interpolator
@android:anim/decelerate_interpolator
@android:anim/linear_interpolator
@android:anim/overshoot_interpolator



<a name="anchor3"></a>
# View Animation的示例

点击查看: [View Animation的示例源码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/animation/view_animation/01_basic/AnimationTest)

该示例包括View Animation的"透明/伸缩/移动/旋转"，也包括"它们的组合"。

View Animation的具体播放代码如下：

            Animation anim = AnimationUtils.loadAnimation(this, R.anim.anim_alpha);
            view.startAnimation(anim);

说明：view是一个View对象。
