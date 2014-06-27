---
layout: post
title: "Android 之Activity启动模式(三)之 启动模式的其它属性"
description: "android training"
category: android
tags: [android]
date: 2014-06-26 12:10
---

> 前面两章分别介绍了"四种launchMode"以及"Intent中与启动模式相关的Flag标签"，本章补充介绍一下manifest中其他与启动模式相关的属性。除非是有特殊需求，否则本章涉及到的知识很少会被用到。


<a name="anchor1"></a>
# 属性介绍

    android:allowTaskReparenting=["true" | "false"]
    android:alwaysRetainTaskState=["true" | "false"]
    android:clearTaskOnLaunch=["true" | "false"]
    android:finishOnTaskLaunch=["true" | "false"]
    android:noHistory=["true" | "false"]
    android:taskAffinity="string"

1. **android:taskAffinity**

  它在前面两章都已经涉及到了。它的作用是描述了不同Activity之间的亲密关系。拥有相同的taskAffinity的Activity是亲密的，它们之间在相互跳转时，会位于同一个task中，而不会新建一个task！   
  如果在manifest中没有对Activity的android:taskAffinity进行配置，则每个Activity都采用和Application相同的taskAffinity；这也就意味着，同一个Application中的所有Activity的taskAffinity在默认情况下是相同的！


2. **allowTaskReparenting**

   与字面理解相同，本属性允许activity重新指定Task。默认值是false。  
   假设存在A并且它allowTaskReparenting为true。当系统中存在一个A的实例，并且A位于task1中时；此时，task2中的某一个Activity要跳转到A中，则此时会将A从task1转移到task2中。

 
3. **alwaysRetainTaskState**

   总是保留task的状态。默认值是false。  
   如果某个Activity的allowTaskReparenting设置为true；那么当该Activity位于某个task的栈底时，不管出现任何情况, 系统都会一直会保留task栈中Activity的状态。


4.  **clearTaskOnLaunch**

   默认值是false。  
   如果某个Activity的clearTaskOnLaunch设置为true。当该Activity位于某个task的栈底时，如果你离开当前的task而转到别的task；那么，该task中除了该Activity之外的其它Activity都会被删除！


5. **finishOnTaskLaunch**

   默认值是false。  
   如果某个Activity的finishOnTaskLaunch设置位true。只要你一离开这个task栈, 则系统会马上清除这个Activity, 不管这个Activity在堆栈的任何位置。
