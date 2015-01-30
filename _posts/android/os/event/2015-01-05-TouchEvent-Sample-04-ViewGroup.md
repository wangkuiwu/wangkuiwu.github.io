---
layout: post
title: "Android 触摸事件机制(五) 触摸事件示例4--ViewGroup拦截并消费触摸事件"
description: "android"
category: android
tags: [android]
date: 2015-01-05 12:01
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

在[触摸事件示例(三)][link_android_event_sample03]中，MyViewGroup只是拦截了触摸事件，但是并没有消费触摸事件。  
而在本文的示例中，MyViewGroup将在拦截触摸事件的基础上，同时消费触摸事件。


本文的示例仍然是在[触摸事件示例(三)][link_android_event_sample03]的基础上修改的。与[触摸事件示例(三)][link_android_event_sample03]相比，本文的示例对MyViewGroup中的onTouchEvent()进行了修改。修改后的onTouchEvent()代码如下：


    @Override
    public boolean onTouchEvent(MotionEvent event) {
        String actionName = Utils.getActionName(event);
        Log.d(TAG, "onTouchEvent(start) :"+actionName);
        // boolean ret = super.onTouchEvent(event);
        boolean ret = true;
        Log.d(TAG, "onTouchEvent( end ) :"+actionName+", ret="+ret);
        return ret;
    }   




<a name="anchor1_2"></a>
## 1.2 示例结论

(01) **MyViewGroup拦截并消费了ACTION_DOWN。那么，该事件就不会分发给MyViewGroup的子类，也不会调用MyActivity的onTouchEvent()。**  
(02) **MyViewGroup拦截并消费了ACTION_DOWN。那么，MyViewGroup就会接受到ACTION_MOVE和ACTION_UP这两种触摸触事件。而且对于ACTION_MOVE和ACTION_UP事件，不会再执行拦截操作，即不会调用MyViewGroup.onInterceptTouchEvent()；而是直接调用MyViewGroup.onTouchEvent()对事件进行处理。**  
  为什么在ACTION_MOVE和ACTION_UP中，没有执行MyViewGroup.onInterceptTouchEvent()呢？查看[ViewGroup中的dispatchTouchEvent()源码]即可得到答案，MyViewGroup在分发ACTION_MOVE时，没有执行"第3步"和"第5步"，而是直接执行"第6步"；进而调用View.dispatchTouchEvent()进行的处理。

Activity中ACTION_DOWN的流程图如下：

<a href="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/event/event04.jpg"><img src="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/event/event04.jpg" alt="" /></a>


<a name="anchor2"></a>
# 2. 示例源码

点击查看：[触摸事件示例4的源码][link_android_event_sample04]


<a name="anchor3"></a>
# 3. 运行结果

## 3.1 ACTION_DOWN事件

点击MyView所在的区域，ACTION_DOWN相关的log如下：

D/##skywang-MyActivity( 2465): dispatchTouchEvent(start) :DOWN
D/##skywang-MyViewGroup( 2465): dispatchTouchEvent(start) :DOWN
D/##skywang-MyViewGroup( 2465): onInterceptTouchEvent(start) :DOWN
D/##skywang-MyViewGroup( 2465): onInterceptTouchEvent( end ) :DOWN, ret=true
D/##skywang-MyViewGroup( 2465): onTouchEvent(start) :DOWN
D/##skywang-MyViewGroup( 2465): onTouchEvent( end ) :DOWN, ret=true
D/##skywang-MyViewGroup( 2465): dispatchTouchEvent( end ) :DOWN, ret=true
D/##skywang-MyActivity( 2465): dispatchTouchEvent( end ) :DOWN, ret=true



说明：很显然，ACTION_DOWN的流程如下：  
(01) MyActivity收到ACTION_DOWN，**进入MyActivity.dispatchTouchEvent()**。  
(02) MyActivity.dispatchTouchEvent()对ACTION_DOWN触摸事件进行分发，将消息传递给MyViewGroup。即，**进入MyViewGroup.dispatchTouchEvent()**。  
(03) MyViewGroup.dispatchTouchEvent()会调用MyViewGroup.onInterceptTouchEvent()检查自己有没有对触摸事件进行拦截。即先**进入MyViewGroup.onInterceptTouchEvent()**。
(04) 紧接着，MyViewGroup会**退出MyViewGroup.onInterceptTouchEvent()**。此时，MyViewGroup.onInterceptTouchEvent()返回true。表示MyViewGroup拦截了该触摸事件。  
(05) MyViewGroup在得知自己拦截了触摸事件之后，将触摸事件交给自己的onTouchEvent()进行处理，即**进入MyViewGroup.onTouchEvent()**。  
(06) 紧接着，MyViewGroup会**退出MyViewGroup.onTouchEvent()**，并返回true。表示MyViewGroup消费了该事件。  
(07) 随后，MyViewGroup会**退出MyViewGroup.dispatchTouchEvent()**，并返回true。表示MyViewGroup接受了该触摸事件。  
(08) MyActivity得知MyViewGroup接受了该触摸事件之后，就会**退出MyActivity.dispatchTouchEvent()**，并返回true。表示此次触摸事件被消费了。




## 3.2 ACTION_MOVE事件

点击MyView所在的区域，ACTION_MOVE相关的log如下：

D/##skywang-MyActivity( 2465): dispatchTouchEvent(start) :MOVE
D/##skywang-MyViewGroup( 2465): dispatchTouchEvent(start) :MOVE
D/##skywang-MyViewGroup( 2465): onTouchEvent(start) :MOVE
D/##skywang-MyViewGroup( 2465): onTouchEvent( end ) :MOVE, ret=true
D/##skywang-MyViewGroup( 2465): dispatchTouchEvent( end ) :MOVE, ret=true
D/##skywang-MyActivity( 2465): dispatchTouchEvent( end ) :MOVE, ret=true


说明：由于MyViewGroup接受了ACTION_DOWN；因此，ACTION_MOVE事件会继续分发给MyViewGroup。不过此时，是直接调用onTouchEvent()进行消息处理，而不再需要执行onInterceptTouchEvent()来拦截消息。   
为什么没有执行MyViewGroup.onInterceptTouchEvent()呢？查看[ViewGroup中的dispatchTouchEvent()源码]即可得到答案，MyViewGroup在分发ACTION_MOVE时，没有执行"第3步"和"第5步"，而是直接执行"第6步"；进而调用View.dispatchTouchEvent()进行的处理。  



## 3.3 ACTION_UP事件

点击MyView所在的区域，ACTION_UP相关的log如下：

D/##skywang-MyActivity( 2595): dispatchTouchEvent(start) :UP
D/##skywang-MyViewGroup( 2595): dispatchTouchEvent(start) :UP
D/##skywang-MyViewGroup( 2595): onTouchEvent(start) :UP
D/##skywang-MyViewGroup( 2595): onTouchEvent( end ) :UP, ret=true
D/##skywang-MyViewGroup( 2595): dispatchTouchEvent( end ) :UP, ret=true
D/##skywang-MyActivity( 2595): dispatchTouchEvent( end ) :UP, ret=true


说明：ACTION_UP的路径和ACTION_MOVE的路径一样！


[link_android_event_sample01]: https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/events/01_event_default/EventTest
[link_android_event_sample02]: https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/events/02_event_view/EventTest
[link_android_event_sample03]: https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/events/03_event_viewgourp/EventTest
[link_android_event_sample04]: https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/events/04_event_viewgourp/EventTest
[link_android_event_sample05]: https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/events/05_event_viewgourp/EventTest

