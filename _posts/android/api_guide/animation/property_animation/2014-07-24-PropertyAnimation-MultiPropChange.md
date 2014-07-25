---
layout: post
title: "Android 之Animation动画(四)之 Property Animation的多属性变化"
description: "android training"
category: android
tags: [android]
date: 2014-07-24 11:00
---

> 本章介绍Property Animation中多属性变化的情况。

> **目录**  
> **1**. [Property Animation的多属性变化的种类](#anchor1)  
> **2**. [Property Animation的关键帧](#anchor2)  
> **3**. [Property Animation的完整示例](#anchor3)  


<a name="anchor1"></a>
# Property Animation的多属性变化的种类

给同一个View实现同一个动画效果(同时变化x和y)，有下面三种方法。

**方法一：用多个ObjectAnimator对象** 

    ObjectAnimator animX = ObjectAnimator.ofFloat(myView, "x", 50f);
    ObjectAnimator animY = ObjectAnimator.ofFloat(myView, "y", 100f);
    AnimatorSet animSetXY = new AnimatorSet();
    animSetXY.playTogether(animX, animY);
    animSetXY.start();
 

**方法二：用一个ObjectAnimator对象加多个PropertyValuesHolder**

    PropertyValuesHolder pvhX = PropertyValuesHolder.ofFloat("x", 50f);
    PropertyValuesHolder pvhY = PropertyValuesHolder.ofFloat("y", 100f);
    ObjectAnimator.ofPropertyValuesHolder(myView, pvhX, pvyY).start();
 

**方法三：用ViewPropertyAnimator**

    myView.animate().x(50f).y(100f);




<a name="anchor2"></a>
# Property Animation的关键帧

通过关键帧，我们能实现较为复杂的动画；例如，实现曲线运动。下面给出关键帧的使用示例：

    // ==== view4的动画 ==== (利用"关键帧"实现曲线运动)
    PropertyValuesHolder anim4Y = PropertyValuesHolder.ofFloat(
            "y", 0f, (float)(view2.getHeight() - view2.getWidth()));
    float x = view2.getX();
    // 三个关键帧
    Keyframe kf0 = Keyframe.ofFloat(0f, x); 
    Keyframe kf1 = Keyframe.ofFloat(.5f, x + 20f);
    Keyframe kf2 = Keyframe.ofFloat(1f, x); 
    PropertyValuesHolder anim4X = PropertyValuesHolder.ofKeyframe(
            "x", kf0, kf1, kf2);
    ObjectAnimator anim4 = ObjectAnimator.ofPropertyValuesHolder(view4, anim4Y, anim4X);
    anim4.setDuration(1000);
    anim4.setInterpolator(new AccelerateInterpolator());
    anim4.setRepeatCount(1);
    anim4.setRepeatMode(ValueAnimator.REVERSE);
    anim4.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
        @Override
        public void onAnimationUpdate(ValueAnimator animation) {
            //view4.invalidate();
            mContainer.invalidate();
        }   
    }); 



<a name="anchor3"></a>
# Property Animation的完整示例

点击查看：[多属性变化和关键帧的完整源码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/animation/property_animation/03_multi_action/AnimationTest)


