---
layout: post
title: "Android 触摸事件机制(五) 触摸事件示例3--ViewGroup拦截但不消费触摸事件"
description: "android"
category: android
tags: [android]
date: 2015-01-05 11:01
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

在[触摸事件示例(二)][link_android_event_sample02]中，MyView接受了触摸事件。  
可是，在有的时候，我们希望MyViewGroup对触摸事件进行拦截；而不希望这个事件发送给MyView进行处理。此时，就需要重载GroupView的onInterceptTouchEvent()来拦截触摸事件。这就是本文要讲到的示例。


本文的示例仍然是在[触摸事件示例(一)][link_android_event_sample01]的基础上修改的。与[触摸事件示例(二)][link_android_event_sample02]不同，本文的示例仅仅只对MyViewGroup中的onInterceptTouchEvent()进行了修改。修改后的onInterceptTouchEvent()代码如下：


    @Override
    public boolean onInterceptTouchEvent(MotionEvent event) {
        String actionName = Utils.getActionName(event);
        Log.d(TAG, "onInterceptTouchEvent(start) :"+actionName);
        // boolean ret = super.onInterceptTouchEvent(event);
        boolean ret = true;
        Log.d(TAG, "onInterceptTouchEvent( end ) :"+actionName+", ret="+ret);
        return ret;
    }   


这里的onTouchEvent()直接返回true，表示MyView消费了触摸事件。


<a name="anchor1_2"></a>
## 1.2 示例结论

(01) **MyViewGroup拦截了ACTION_DOWN，并没有消费该ACTION_DOWN。既然MyViewGroup拦截了ACTION_DOWN，那就意味着该事件就不会分发给MyViewGroup的子类。但是由于MyViewGroup没有消费该事件，即它并没有接受该事件；那么，ACTION_DOWN会继续查找其他对象来消费它自己，这也意味着该触摸事件仍然会发送MyActivity的onTouchEvent()。**  
  如果MyActivity中有和MyViewGroup同级别的GroupView的话，在得知MyViewGroup拦截了ACTION_DOWN，却没有消费该ACTION_DOWN之后；MyActivity仍然能够向这个同级的GroupView分发消息。  
(02) **MyViewGroup并没有消费ACTION_DOWN，那么，MyViewGroup就不能接受到ACTION_MOVE和ACTION_UP这两种触摸触事件。至于MyViewGroup的子类MyView，就更加不可能接受到ACTION_MOVE和ACTION_UP了。**

Activity中ACTION_DOWN的流程图如下：

<a href="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/event/event03.jpg"><img src="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/event/event03.jpg" alt="" /></a>


<a name="anchor2"></a>
# 2. 示例源码

点击查看：[触摸事件示例3的源码][link_android_event_sample03]


<a name="anchor3"></a>
# 3. 运行结果

## 3.1 ACTION_DOWN事件

点击MyView所在的区域，ACTION_DOWN相关的log如下：

D/##skywang-MyActivity( 2371): dispatchTouchEvent(start) :DOWN
D/##skywang-MyViewGroup( 2371): dispatchTouchEvent(start) :DOWN
D/##skywang-MyViewGroup( 2371): onInterceptTouchEvent(start) :DOWN
D/##skywang-MyViewGroup( 2371): onInterceptTouchEvent( end ) :DOWN, ret=true
D/##skywang-MyViewGroup( 2371): onTouchEvent(start) :DOWN
D/##skywang-MyViewGroup( 2371): onTouchEvent( end ) :DOWN, ret=false
D/##skywang-MyViewGroup( 2371): dispatchTouchEvent( end ) :DOWN, ret=false
D/##skywang-MyActivity( 2371): onTouchEvent(start) :DOWN
D/##skywang-MyActivity( 2371): onTouchEvent( end ) :DOWN, ret=false
D/##skywang-MyActivity( 2371): dispatchTouchEvent( end ) :DOWN, ret=false



说明：很显然，ACTION_DOWN的流程如下：  
(01) MyActivity收到ACTION_DOWN，**进入MyActivity.dispatchTouchEvent()**。  
(02) MyActivity.dispatchTouchEvent()对ACTION_DOWN触摸事件进行分发，将消息传递给MyViewGroup。即，**进入MyViewGroup.dispatchTouchEvent()**。  
(03) MyViewGroup.dispatchTouchEvent()会调用MyViewGroup.onInterceptTouchEvent()检查自己有没有对触摸事件进行拦截。即先**进入MyViewGroup.onInterceptTouchEvent()**。
(04) 紧接着，MyViewGroup会**退出MyViewGroup.onInterceptTouchEvent()**。此时，MyViewGroup.onInterceptTouchEvent()返回true。表示MyViewGroup拦截了该触摸事件。  
(05) MyViewGroup在得知自己拦截了触摸事件之后，将触摸事件交给自己的onTouchEvent()进行处理，即**进入MyViewGroup.onTouchEvent()**。  
(06) 紧接着，MyViewGroup会**退出MyViewGroup.onTouchEvent()**。而MyViewGroup自身并没有消费该事件，因此MyViewGroup.onTouchEvent()返回false。  
(07) 随后，**退出MyViewGroup.dispatchTouchEvent()**，并返回false。表示MyViewGroup没有接受该触摸事件。  
(08) MyActivity得知MyViewGroup没有接受该触摸事件之后，就会调用**进入MyActivity.onTouchEvent()**。  
(09) 紧接着，MyActivity会**退出MyActivity.onTouchEvent()**，并返回false。表示MyActivity也没有消费触摸事件。  
(10) 最后，MyActivity会**退出MyActivity.dispatchTouchEvent()**，并返回false。表示此次触摸事件没有被消费。

对比，[触摸事件示例(一)][link_android_event_sample01]中的ACTION_DOWN路径。在本示例中，MyViewGroup拦截了ACTION_DOWN，但是没有消费ACTION_DOWN事件。 (01) MyViewGroup拦截了ACTION_DOWN事件，意味着该事件不会继续往下分发。 (02) MyViewGroup没有消费该事件，意味着该事件就继续往上分发。



## 3.2 ACTION_MOVE事件

点击MyView所在的区域，ACTION_MOVE相关的log如下：

D/##skywang-MyActivity( 2371): dispatchTouchEvent(start) :MOVE
D/##skywang-MyActivity( 2371): onTouchEvent(start) :MOVE
D/##skywang-MyActivity( 2371): onTouchEvent( end ) :MOVE, ret=false
D/##skywang-MyActivity( 2371): dispatchTouchEvent( end ) :MOVE, ret=false


说明：由于MyViewGroup拦截了ACTION_DOWN，却没有消费给ACTION_DOWN；导致ACTION_MOVE不会分发给MyViewGroup。既然没有分发给MyViewGroup，就更加谈不上分发给MyView了。



## 3.3 ACTION_UP事件

点击MyView所在的区域，ACTION_UP相关的log如下：

D/##skywang-MyActivity( 2371): dispatchTouchEvent(start) :UP
D/##skywang-MyActivity( 2371): onTouchEvent(start) :UP
D/##skywang-MyActivity( 2371): onTouchEvent( end ) :UP, ret=false
D/##skywang-MyActivity( 2371): dispatchTouchEvent( end ) :UP, ret=false


说明：ACTION_UP的路径和ACTION_MOVE的路径一样！


[link_android_event_sample01]: https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/events/01_event_default/EventTest
[link_android_event_sample02]: https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/events/02_event_view/EventTest
[link_android_event_sample03]: https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/events/03_event_viewgourp/EventTest
[link_android_event_sample04]: https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/events/04_event_viewgourp/EventTest
[link_android_event_sample05]: https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/events/05_event_viewgourp/EventTest

