---
layout: post
title: "Android 之Animation动画(三)之 Property Animation的XML属性和使用示例"
description: "android training"
category: android
tags: [android]
date: 2014-07-24 10:00
---

> 前面介绍了Property Animation的相关类和类的基本使用方法，本章将介绍Property Animation的属性以及如何通过属性来使用Property Animation。

> **目录**  
> **1**. [Property Animation属性介绍](#anchor1)  
> **2**. [Property Animation属性使用示例](#anchor2)  


<a name="anchor1"></a>
# Property Animation属性介绍

我们可以用XML文件来定义Property Animation。XML的基本语法如下：

    <set
      android:ordering=["together" | "sequentially"]>

        <objectAnimator
            android:propertyName="string"
            android:duration="int"
            android:valueFrom="float | int | color"
            android:valueTo="float | int | color"
            android:startOffset="int"
            android:repeatCount="int"
            android:repeatMode=["repeat" | "reverse"]
            android:valueType=["intType" | "floatType"]/>

        <animator
            android:duration="int"
            android:valueFrom="float | int | color"
            android:valueTo="float | int | color"
            android:startOffset="int"
            android:repeatCount="int"
            android:repeatMode=["repeat" | "reverse"]
            android:valueType=["intType" | "floatType"]/>

        <set>
            ...
        </set>
    </set>


说明：  
(01) **<set>**: 是objectAnimator和animator的集合。<set>标签是可以嵌套的。  
(02) **<objectAnimator>**: 对应是ObjectAnimator动画。  
(03) **<animator>**: 对应是ValueAnimator动画。  
(04) **android:propertyName**: 属性名。仅ObjectAnimator才有该属性。  
(05) **android:duration**: 动画的总时间，以ms为单位，默认是300ms。  
(06) **android:valueFrom**: 动画的起始值。  
(07) **android:valueTo**: 动画的结束值。  
(08) **android:startOffset**: 动画的起始偏移时间，以ms为单位。  
(09) **android:repeatCount**: 动画重复播放重复次数。-1表示无穷次，默认是0。  
(10) **android:repeatMode**: 动画重复播放时的模式。repeat表示和原来一样从头开始播放，reverse表示反向播放；默认是repeat。  
(11) **android:valueType**: 属性的值的类型。可以位intType或floatType。  

假设存在res/anim/property_animator.xml文件，该文件中定义了Property Animation动画。则动画的使用方法如下：

    AnimatorSet set = (AnimatorSet) AnimatorInflater.loadAnimator(myContext,
        R.anim.property_animator);
    set.setTarget(myObject);
    set.start();

此外，补充说明两点：  
(01) View Animation的部分属性在Property Animation中也是可以使用的。例如，android:interpolator。  
(02) 为了区分Property Animation和View Animation的资源文件，从Android 3.1开始，Property Animation的xml文件存在res/animator/目录下（View Animation存在res/anim/目录下）， animator这个名是可选的。



<a name="anchor2"></a>
# Property Animation属性使用示例

点击查看：[Property Animation属性使用示例的源码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/animation/property_animation/02_xml_basic/AnimationTest)

在该示例中存在多种动画。下面列举一种：球加速下落，并具有弹跳效果。

动画的配置文件res/anim/view01.xml的内容如下：

    <?xml version="1.0" encoding="utf-8"?>
    <objectAnimator xmlns:android="http://schemas.android.com/apk/res/android"
        android:duration="1200"
        android:propertyName="y"
        android:valueFrom="0"
        android:valueTo="249"
        android:valueType="floatType"
        android:interpolator="@android:anim/bounce_interpolator"
        android:repeatCount="0" />

动画的使用代码如下：

        ObjectAnimator anim1 = (ObjectAnimator) AnimatorInflater.loadAnimator(context, R.anim.view01);
        anim1.setTarget(view1);
        anim1.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animation) {
                view1.invalidate();
            }
        });

