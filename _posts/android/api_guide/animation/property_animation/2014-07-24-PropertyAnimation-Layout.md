---
layout: post
title: "Android 之Animation动画(五)之 Property Animation的布局动画"
description: "android training"
category: android
tags: [android]
date: 2014-07-24 12:00
---

> 本章如何通过Property Animation设置Android的布局动画。

> **目录**  
> **1**. [Property Animation布局动画简介](#anchor1)  
> **2**. [Property Animation布局动画示例](#anchor2)  


<a name="anchor1"></a>
# Property Animation布局动画简介

Property Animation支持对ViewGroup中的View设置动画。

例如，当你添加或者移除ViewGroup中的View时，或者你调用View的setVisibility()方法来控制其显示或消失时；就可以设置相应的Property Animation动画。

Android的View视图支持动画的主要有四种行为：  
**APPEARING**：某个View被添加到ViewGroup中时，该View的动画。  
**DISAPPEARING**：某个View从ViewGroup中删除时，该View的动画。  
**CHANGE_APPEARING**：某个View被添加到ViewGroup中，并引起该ViewGroup中其他View的变化位置时，其他View的动画。  
**CHANGE_DISAPPEARING**：某个View从ViewGroup中删除，，并引起该ViewGroup中其他View的变化位置时，其他View的动画。



<a name="anchor2"></a>
# Property Animation布局动画示例

点击查看：[Property Animation布局动画示例](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/animation/property_animation/04_layout_animation/AnimationTest)

该示例中，包括：Android默认的动画，以及前面介绍的View的四种类型的动画的演示。以APPEARING来简单说明下布局动画的使用。

        // 设置ViewGroup(mSelfLayout)对应的LayoutTransition。
        LayoutTransition transition = new LayoutTransition();
        mSelfLayout.setLayoutTransition(transition);

        // (添加)动画：APPEARING
        ObjectAnimator animIn = ObjectAnimator.ofFloat(null, "rotationY", 90f, 0f);
        animIn.setDuration(transition.getDuration(LayoutTransition.APPEARING));
        transition.setAnimator(LayoutTransition.APPEARING, animIn);
        animIn.addListener(new AnimatorListenerAdapter() {
            @Override
            public void onAnimationEnd(Animator anim) {
                View view = (View) ((ObjectAnimator) anim).getTarget();
                view.setRotationY(0f);
            }   
        }); 


