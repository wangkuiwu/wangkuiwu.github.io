---
layout: post
title: "Android 之Animation动画(二)之 Property Animation的基本介绍和使用示例"
description: "android training"
category: android
tags: [android]
date: 2014-07-24 09:00
---

> 本章介绍Android的Property Animation。

> **目录**  
> **1**. [Property Animation简介](#anchor1)  
> **2**. [Property Animation的基本用法和示例源码](#anchor2)  


<a name="anchor1"></a>
# Property Animation简介

Property Animation是属性动画。它是在Android 3.0中才引进的，它比View Animation和Drawable Animation功能更加强大。


## 1. Property Animation支持的属性

在Property Animation中，可以对动画应用以下属性：  
**Duration**：动画的持续时间。  
**TimeInterpolation**：属性值的计算方式，如先快后慢。  
**TypeEvaluator**：根据属性的开始、结束值与TimeInterpolation计算出的因子计算出当前时间的属性值。  
**Repeat Count and behavoir**：重复次数与方式，如播放3次、5次、无限循环，可以此动画一直重复，或播放完时再反向播放。  
**Animation sets**：动画集合，即可以同时对一个对象应用几个动画，这些动画可以同时播放也可以对不同动画设置不同开始偏移。  
**Frame refreash delay**：多少时间刷新一次，即每隔多少时间计算一次属性值，默认为10ms，最终刷新时间还受系统进程调度与硬件的影响。  


## 2. Property Animation的工作原理

对于下图的动画，这个对象的X坐标在40ms内从0移动到40 pixel。默认的10ms刷新一次，这个对象会移动4次，每次移动40/4=10pixel。

<a href="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/api_guide/animation/property_animation/pic/01.png"><img src="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/api_guide/animation/property_animation/pic/01.png" alt="" /></a>

也可以改变属性值的改变方法，即设置不同的interpolation，在下图中运动速度先快后慢。

<a href="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/api_guide/animation/property_animation/pic/02.png"><img src="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/api_guide/animation/property_animation/pic/02.png" alt="" /></a>



## 3. Property Animation的框架

Animator  
&nbsp;&nbsp; -- &nbsp;&nbsp; ValueAnimator  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; -- &nbsp;&nbsp; ObjectAnimator  
&nbsp;&nbsp;  --  AnimatorSet  
AnimatorInflater  
Keyframe  
KeyframeSet  
PropertyValuesHolder  
AnimatorListenerAdapter.java
TypeEvaluator  
&nbsp;&nbsp; --  IntEvaluator  
&nbsp;&nbsp; --  FloatEvaluator  
&nbsp;&nbsp; --  ArgbEvaluator  

说明：  
(01) Animator, ValueAnimator, ObjectAnimator是描述动画的核心类。其中，Animator是父类，它定义了动画开始/结束/暂停/恢复/重复等接口，并实现了公共函数。ValueAnimator和ObjectAnimator是描述动画的具体类。    
(02) AnimatorSet是动画集合。  
(03) AnimatorInflater是解析xml定义的动画的核心类。  
(04) Keyframe是关键帧，通过关键帧可以实现较复杂的动画(例如，曲线运动等)。KeyframeSet是关键帧的辅助类。  
(05) PropertyValuesHolder通常用于动画中有多个属性需要同时变化的情况。  
(06) AnimatorUpdateListener中实现了全部的动画监听接口。但是，监听函数体都没有执行任何动作。在我们需要监听动画相应动作时，可以实现Animator提供的接口，也可以继承于AnimatorUpdateListener。  
(07) TypeEvaluator则是动画中需要变化的属性值的计算类。Android提供了三种：用于计算int类型属性的IntEvaluator，用于计算float类型属性的FloatEvaluator，和用于计算rgb颜色类属性的ArgbEvaluator。若上面的三种均无法满足你的需求，则你可以自定义属性计算类。  


## 4. Property Animation和View Animation的区别

1. Property Animation的动画对象是Object类型，而View Animation仅仅适用于View对象。

2. View Animation动画的功能有限：它可以进行缩放和旋转，但是却无法改变背景色。

3. View Animation动画在变化时，仅仅改变了View的绘制位置，并没有改变View本身的实际位置。  
   比如，如果当通过View Animation让一个按钮移动到屏幕上的另一个位置时；虽然它绘制在目标位置，但是它的点击区域并没改变，还是和变化之前的点击区域一样。

Property Animation就不存在上面的问题，它是确实地改变了View对象的属性。虽然，View Animation存在上述缺点；但它一个明显的有点就是使用方法更简单。在View Animation能满足你的需求时，就不需要使用Property Animation。




<a name="anchor2"></a>
# Property Animation的基本用法和示例源码

Android提供的Property Animation动画类主要有两个：ValueAnimator和ObjectAnimator。

点击查看: [PropertyAnimation基本用法的完整源码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/animation/property_animation/01_basic/AnimationTest)

## 1. ValueAnimator的基本用法

在显示动画的具体图像之前，需要执行两步操作：  
(01) 计算属性值。  
(02) 根据此时的属性值执行相应的动作，如改变对象的某一属性。  

ValueAnimator只完成了第一步。当我们使用ValueAnimator时，需要自己完成第二步。如下示例：

    // 创建ValueAnimator动画。ofFloat()的参数是从"动画开始" 到 "动画结束"对应的值。
    ValueAnimator anim2 = ValueAnimator.ofFloat(0f, (float)(view2.getHeight() - view2.getWidth()));
    // 总的显示时间
    anim2.setDuration(500);
    // 变化模式(加速)
    anim2.setInterpolator(new AccelerateInterpolator());
    // 监听：每次变化时的回调函数
    anim2.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
        @Override
        public void onAnimationUpdate(ValueAnimator animation) {
            // 设置view2的纵坐标y
            view2.setY((Float) animation.getAnimatedValue());
            // 更新view2
            view2.invalidate();
        }
    });


说明：  
(01) view2是自定义的View试图，它是一个圆。  
(02) 对于ValueAnimator而言，需要我们实现AnimatorUpdateListener()接口，并在接口中处理试图的位置变化。例如，view2.setY()就是用于设置view2的位置。这就是上面所说的第二步。  
(03) onAnimationUpdate()是动画变化的回调函数。当动画发生变化时，回执行该函数。  


## 2. ObjectAnimator的基本用法

ObjectAnimator继承于ValueAnimator。它相比于ValueAnimator，完成了第二部。下面看看ObjectAnimator的使用方法。  

    // 创建ObjectAnimator动画。view1中必须有setY方法
    ObjectAnimator anim1 = ObjectAnimator.ofFloat(view1, "y", 0f, (float)(view1.getHeight() - view1.getWidth()));
    // 总的显示时间
    anim1.setDuration(1200);
    // 变化模式(弹跳)
    anim1.setInterpolator(new BounceInterpolator());
    // 监听：每次变化时的回调函数
    anim1.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
        @Override
        public void onAnimationUpdate(ValueAnimator animation) {
            // 更新view1
            view1.invalidate();
        }
    });

说明  
(01) 虽然ObjectAnimator完成了第二步。ObjectAnimator的操作对象必须有对应的set<PropertyName>方法。例如，上面的示例中操作的属性是"y"，因此view1类中必须要有setY(float)函数。  
(02) 虽然Object完成了位置的计算和设置。但是，我们还必须在onAnimationUpdate()中更新要显示的视图。  



## 3. AnimatorSet的基本用法

AnimatorSet是动画集合。用于来管理多个动画的播放次序。 如下示例：

        AnimatorSet animSet = new AnimatorSet();
        animSet.playTogether(anim1, anim2, anim3);// 并行
        animSet.playSequentially(anim3, anim4, anim5);// 串行
        animSet.start();


