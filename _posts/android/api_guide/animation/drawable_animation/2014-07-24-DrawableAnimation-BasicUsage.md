---
layout: post
title: "Android 之Animation动画(七)之 Drawable Animation"
description: "android training"
category: android
tags: [android]
date: 2014-07-26 09:00
---

> 本章介绍Android中的Drawable Animation。

> **目录**  
> **1**. [Drawable Animation的简介和语法](#anchor1)  
> **2**. [Drawable Animation的示例](#anchor2)  


<a name="anchor1"></a>
# Drawable Animation的简介和语法

Drawable Animation(Frame Animation)：帧动画，就像GIF图片，通过一系列Drawable依次显示来模拟动画的效果。


Drawable Animation的定义是通过在**res/anim**目录下新建一个xml文件来定义。xml文件的格式如下：

    <?xml version="1.0" encoding="utf-8"?>
    <animation-list xmlns:android="http://schemas.android.com/apk/res/android"
        android:oneshot="false">
        <item android:drawable="@drawable/rocket_thrust1" android:duration="200" />
        <item android:drawable="@drawable/rocket_thrust2" android:duration="200" />
        <item android:drawable="@drawable/rocket_thrust3" android:duration="200" />
    </animation-list>

说明：  
(01) oneshot表示是否循环播放。true表示不循环播放，否则就循环播放。  
(02) duration表示每帧的播放时间。  


<a name="anchor2"></a>
# Drawable Animation的示例

点击查看：[Drawable Animation示例的完整源码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/animation/drawable_animation/01_basic/AnimationTest)

示例中是通过ImageView来使用Drawable Animation的。 下面是获取ImageView的AnimationDrawable对象的方法：

        mImage = (ImageView)findViewById(R.id.animation);
        mImage.setBackgroundResource(R.anim.anim_kof);
        mAnimation = (AnimationDrawable) mImage.getBackground();

说明：res/anim/anim_kof.xml就是动画的定义。


得到AnimationDrawable对象之后，就可以通过mAnimation.start()直接启动动画了。 


