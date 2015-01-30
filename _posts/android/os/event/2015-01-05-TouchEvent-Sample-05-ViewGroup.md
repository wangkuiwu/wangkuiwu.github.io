---
layout: post
title: "Android 触摸事件机制(五) 触摸事件示例5--ViewGroup没拦截但是却消费了触摸事件"
description: "android"
category: android
tags: [android]
date: 2015-01-05 13:01
---


> 本文将通过示例演示触摸事件的传递流程。

> 注意：本文是基于Android 4.4.2版本进行介绍的！

> **目录**  
> **1**. [示例概述](#anchor1)  
> **1.1**. [示例简介](#anchor1_1)  
> **1.2**. [示例结论](#anchor1_2)  
> **2**. [示例源码](#anchor2)  
> **3**. [运行结果](#anchor3)  


<a name="anchor1"></a>
# 1. 示例概述

<a name="anchor1_1"></a>
## 1.1 示例简介

本文的示例是在[触摸事件示例(一)][link_android_event_sample01]的基础上修改的。与[触摸事件示例(一)][link_android_event_sample01]相比，本文的示例对MyViewGroup中的onTouchEvent()进行了修改。修改后的onTouchEvent()代码如下：


    @Override
    public boolean onTouchEvent(MotionEvent event) {
        String actionName = Utils.getActionName(event);
        Log.d(TAG, "onTouchEvent(start) :"+actionName);
        // boolean ret = super.onTouchEvent(event);
        boolean ret = true;
        Log.d(TAG, "onTouchEvent( end ) :"+actionName+", ret="+ret);
        return ret;
    }   

说明：修改后的MyViewGroup没有拦截触摸事件，但是消费了触摸事件。



<a name="anchor1_2"></a>
## 1.2 示例结论

(01) **MyViewGroup没有拦截却消费了ACTION_DOWN。由于MyViewGroup没有拦截ACTION_DOWN，因此，该事件会继续分发给MyViewGroup的子类MyView。由于MyViewGroup消费了ACTION_DOWN，因此该事件不会分发给MyActivity的onTouchEvent()。**  
(02) **MyViewGroup没有拦截却消费了ACTION_DOWN。那么，MyViewGroup仍然可以接受到ACTION_MOVE和ACTION_UP这两种触摸触事件。但是对于MyView而言，由于MyView没有接受该事件；因此，MyView不会收到ACTION_MOVE和ACTION_UP。**  
   试想想，如果MyView接受了ACTION_DOWN事件的话；它是否会收到ACTION_MOVE和ACTION_UP事件呢？答案是：会。感兴趣的读者可以自行验证。

Activity中ACTION_DOWN的流程图如下：

<a href="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/event/event05.jpg"><img src="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/event/event05.jpg" alt="" /></a>

<a name="anchor2"></a>
# 2. 示例源码

点击查看：[触摸事件示例5的源码][link_android_event_sample05]


<a name="anchor3"></a>
# 3. 运行结果

## 3.1 ACTION_DOWN事件

点击MyView所在的区域，ACTION_DOWN相关的log如下：

D/##skywang-MyActivity( 2950): dispatchTouchEvent(start) :DOWN
D/##skywang-MyViewGroup( 2950): dispatchTouchEvent(start) :DOWN
D/##skywang-MyViewGroup( 2950): onInterceptTouchEvent(start) :DOWN
D/##skywang-MyViewGroup( 2950): onInterceptTouchEvent( end ) :DOWN, ret=false
D/##skywang-MyView( 2950): dispatchTouchEvent(start) :DOWN
D/##skywang-MyView( 2950): onTouchEvent(start) :DOWN
D/##skywang-MyView( 2950): onTouchEvent( end ) :DOWN, ret=false
D/##skywang-MyView( 2950): dispatchTouchEvent( end ) :DOWN, ret=false
D/##skywang-MyViewGroup( 2950): onTouchEvent(start) :DOWN
D/##skywang-MyViewGroup( 2950): onTouchEvent( end ) :DOWN, ret=true
D/##skywang-MyViewGroup( 2950): dispatchTouchEvent( end ) :DOWN, ret=true
D/##skywang-MyActivity( 2950): dispatchTouchEvent( end ) :DOWN, ret=true




## 3.2 ACTION_MOVE事件

点击MyView所在的区域，ACTION_MOVE相关的log如下：

D/##skywang-MyActivity( 2950): dispatchTouchEvent(start) :MOVE
D/##skywang-MyViewGroup( 2950): dispatchTouchEvent(start) :MOVE
D/##skywang-MyViewGroup( 2950): onTouchEvent(start) :MOVE
D/##skywang-MyViewGroup( 2950): onTouchEvent( end ) :MOVE, ret=true
D/##skywang-MyViewGroup( 2950): dispatchTouchEvent( end ) :MOVE, ret=true
D/##skywang-MyActivity( 2950): dispatchTouchEvent( end ) :MOVE, ret=true


## 3.3 ACTION_UP事件

点击MyView所在的区域，ACTION_UP相关的log如下：

D/##skywang-MyActivity( 2950): dispatchTouchEvent(start) :UP
D/##skywang-MyViewGroup( 2950): dispatchTouchEvent(start) :UP
D/##skywang-MyViewGroup( 2950): onTouchEvent(start) :UP
D/##skywang-MyViewGroup( 2950): onTouchEvent( end ) :UP, ret=true
D/##skywang-MyViewGroup( 2950): dispatchTouchEvent( end ) :UP, ret=true
D/##skywang-MyActivity( 2950): dispatchTouchEvent( end ) :UP, ret=true


[link_android_event_sample01]: https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/events/01_event_default/EventTest
[link_android_event_sample02]: https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/events/02_event_view/EventTest
[link_android_event_sample03]: https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/events/03_event_viewgourp/EventTest
[link_android_event_sample04]: https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/events/04_event_viewgourp/EventTest
[link_android_event_sample05]: https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/events/05_event_viewgourp/EventTest

