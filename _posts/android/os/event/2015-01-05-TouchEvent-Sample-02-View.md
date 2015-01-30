---
layout: post
title: "Android 触摸事件机制(五) 触摸事件示例2--View接受触摸事件"
description: "android"
category: android
tags: [android]
date: 2015-01-05 10:01
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

本文的示例是在[触摸事件示例(一)][link_android_event_sample01]的基础上修改的。即本文的示例仍然是：自定义一个Activity，该Activity中的显示内容是包含一个自定义的ViewGroup，该ViewGroup中包含一个自定义的View。

相比[触摸事件示例(一)][link_android_event_sample01]，本示例对MyView中的onTouchEvent()进行了修改。修改后的onTouchEvent()代码如下：

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        String actionName = Utils.getActionName(event);
        Log.d(TAG, "onTouchEvent(start) :"+actionName);
        // boolean ret = super.onTouchEvent(event);
        boolean ret = true;
        Log.d(TAG, "onTouchEvent( end ) :"+actionName+", ret="+ret);
        return ret;
    }   

这里的onTouchEvent()直接返回true，表示MyView消费了触摸事件。


<a name="anchor1_2"></a>
## 1.2 示例结论

(01) **如果MyView接受了ACTION_DOWN，那么就不会再再执行其他对象的onTouchEvent()函数的。即，不会执行MyViewGroup的onTouchEvent()和MyActivity的onTouchEvent()。因为MyView接受了ACTION_DOWN，意味着这个事件已经被消费了；就无须其他对象再来消费ACTION_DOWN了。**  
(02) **如果MyView接受了ACTION_DOWN，那么MyView能继续收到ACTION_MOVE和ACTION_UP这两种触摸触事件。并且ACTION_MOVE和ACTION_UP的处理流程和ACTION_DOWN的流程基本一样。**

Activity中ACTION_DOWN的流程图如下：

<a href="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/event/event02.jpg"><img src="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/event/event02.jpg" alt="" /></a>


<a name="anchor2"></a>
# 2. 示例源码

点击查看：[触摸事件示例2的源码][link_android_event_sample02]


<a name="anchor3"></a>
# 3. 运行结果

## 3.1 ACTION_DOWN事件

点击MyView所在的区域，ACTION_DOWN相关的log如下：

D/##skywang-MyActivity( 2273): dispatchTouchEvent(start) :DOWN
D/##skywang-MyViewGroup( 2273): dispatchTouchEvent(start) :DOWN
D/##skywang-MyViewGroup( 2273): onInterceptTouchEvent(start) :DOWN
D/##skywang-MyViewGroup( 2273): onInterceptTouchEvent( end ) :DOWN, ret=false
D/##skywang-MyView( 2273): dispatchTouchEvent(start) :DOWN
D/##skywang-MyView( 2273): onTouchEvent(start) :DOWN
D/##skywang-MyView( 2273): onTouchEvent( end ) :DOWN, ret=true
D/##skywang-MyView( 2273): dispatchTouchEvent( end ) :DOWN, ret=true
D/##skywang-MyViewGroup( 2273): dispatchTouchEvent( end ) :DOWN, ret=true
D/##skywang-MyActivity( 2273): dispatchTouchEvent( end ) :DOWN, ret=true



说明：很显然，ACTION_DOWN的流程如下：  
(01) MyActivity收到ACTION_DOWN，**进入MyActivity.dispatchTouchEvent()**。  
(02) MyActivity.dispatchTouchEvent()对ACTION_DOWN触摸事件进行分发，将消息传递给MyViewGroup。即，**进入MyViewGroup.dispatchTouchEvent()**。  
(03) MyViewGroup.dispatchTouchEvent()会调用MyViewGroup.onInterceptTouchEvent()检查自己有没有对触摸事件进行拦截。即先**进入MyViewGroup.onInterceptTouchEvent()**。
(04) 紧接着，MyViewGroup会**退出MyViewGroup.onInterceptTouchEvent()**。因为MyViewGroup没有对触摸事件进行拦截，MyViewGroup会继续分发事件。  
(05) MyViewGroup将触摸事件分发给MyView，即**进入MyView.dispatchTouchEvent()**。  
(06) MyView会调用onTouchEvent()对触摸事件进行处理，即**进入MyView.onTouchEvent()**  。
(07) 紧接着，MyView会**退出MyView.onTouchEvent()**。此时的，MyView.onTouchEvent()返回的是true；表示MyView消费了此次触摸事件。  
(08) MyView.dispatchTouchEvent()得知MyView.onTouchEvent()消费此次触摸事件之后；也就返回true，表示MyView接受该此次触摸事件。  
(09) MyViewGroup则得知MyView接受了该触摸事件之后，就**退出MyViewGroup.dispatchTouchEvent()**，并返回true。  
(10) MyActivity得知MyViewGroup接受了该触摸事件之后，就会调用**退出MyActivity.dispatchTouchEvent()**，并返回true。

对比，[触摸事件示例(一)][link_android_event_sample01]中的ACTION_DOWN路径。在本示例中，MyView消费了ACTION_DOWN事件之后；触摸事件就没有再发送给MyViewGroup.onTouchEvent()以及MyActivity.onTouchEvent()。



## 3.2 ACTION_MOVE事件

点击MyView所在的区域，ACTION_MOVE相关的log如下：

D/##skywang-MyActivity( 2273): dispatchTouchEvent(start) :MOVE
D/##skywang-MyViewGroup( 2273): dispatchTouchEvent(start) :MOVE
D/##skywang-MyViewGroup( 2273): onInterceptTouchEvent(start) :MOVE
D/##skywang-MyViewGroup( 2273): onInterceptTouchEvent( end ) :MOVE, ret=false
D/##skywang-MyView( 2273): dispatchTouchEvent(start) :MOVE
D/##skywang-MyView( 2273): onTouchEvent(start) :MOVE
D/##skywang-MyView( 2273): onTouchEvent( end ) :MOVE, ret=true
D/##skywang-MyView( 2273): dispatchTouchEvent( end ) :MOVE, ret=true
D/##skywang-MyViewGroup( 2273): dispatchTouchEvent( end ) :MOVE, ret=true
D/##skywang-MyActivity( 2273): dispatchTouchEvent( end ) :MOVE, ret=true

说明：由于MyView接受了ACTION_DOWN；因此，ACTION_MOVE事件会继续分发给MyView。ACTION_MOVE的分发路径和ACTION_DOWN的路径基本上一样！



## 3.3 ACTION_UP事件

点击MyView所在的区域，ACTION_UP相关的log如下：

D/##skywang-MyActivity( 2273): dispatchTouchEvent(start) :UP
D/##skywang-MyViewGroup( 2273): dispatchTouchEvent(start) :UP
D/##skywang-MyViewGroup( 2273): onInterceptTouchEvent(start) :UP
D/##skywang-MyViewGroup( 2273): onInterceptTouchEvent( end ) :UP, ret=false
D/##skywang-MyView( 2273): dispatchTouchEvent(start) :UP
D/##skywang-MyView( 2273): onTouchEvent(start) :UP
D/##skywang-MyView( 2273): onTouchEvent( end ) :UP, ret=true
D/##skywang-MyView( 2273): dispatchTouchEvent( end ) :UP, ret=true
D/##skywang-MyViewGroup( 2273): dispatchTouchEvent( end ) :UP, ret=true
D/##skywang-MyActivity( 2273): dispatchTouchEvent( end ) :UP, ret=true


说明：由于MyView接受了ACTION_DOWN；因此，ACTION_UP事件会继续分发给MyView。ACTION_UP的分发路径和ACTION_DOWN的路径基本上一样！


[link_android_event_sample01]: https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/events/01_event_default/EventTest
[link_android_event_sample02]: https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/events/02_event_view/EventTest
[link_android_event_sample03]: https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/events/03_event_viewgourp/EventTest
[link_android_event_sample04]: https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/events/04_event_viewgourp/EventTest
[link_android_event_sample05]: https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/events/05_event_viewgourp/EventTest

