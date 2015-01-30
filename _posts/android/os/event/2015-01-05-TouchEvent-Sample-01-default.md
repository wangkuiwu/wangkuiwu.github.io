---
layout: post
title: "Android 触摸事件机制(五) 触摸事件示例1--默认处理方式"
description: "android"
category: android
tags: [android]
date: 2015-01-05 09:01
---


> 本文将通过示例演示触摸事件的传递流程。

> 注意：本文是基于Android 4.4.2版本进行介绍的！

> **目录**  
> **1**. [示例概述](#anchor1)  
> **1.1**. [示例简介](#anchor1_1)  
> **1.2**. [示例结论](#anchor1_2)  
> **2**. [示例源码](#anchor2)  
> **2.1**. [MyActivity的源码](#anchor2_1)  
> **2.2**. [MyViewGroup的源码](#anchor2_2)  
> **2.3**. [MyView的源码](#anchor2_3)  
> **3**. [运行结果](#anchor3)  


<a name="anchor1"></a>
# 1. 示例概述

<a name="anchor1_1"></a>
## 1.1 示例简介

本文的示例：自定义一个Activity，该Activity中的显示内容是包含一个自定义的ViewGroup，该ViewGroup中包含一个自定义的View。

(01) 自定义的Activity是MyActivity  
public boolean dispatchTouchEvent(MotionEvent ev): 调用系统默认的dispatchTouchEvent()  
public boolean onTouchEvent(MotionEvent ev): 调用系统默认的onTouchEvent()

(02) 自定义ViewGroup是MyViewGroup  
public boolean dispatchTouchEvent(MotionEvent ev): 调用系统默认的dispatchTouchEvent()  
public boolean onTouchEvent(MotionEvent ev): 调用系统默认的onTouchEvent()  
public boolean onInterceptTouchEvent(MotionEvent ev):: 调用系统默认的onInterceptTouchEvent() 

(03) 自定义View是MyView  
public boolean dispatchTouchEvent(MotionEvent ev): 调用系统默认的dispatchTouchEvent()  
public boolean onTouchEvent(MotionEvent ev): 调用系统默认的onTouchEvent()



<a name="anchor1_2"></a>
## 1.2 示例结论

(01) **MyActivity, ViewGroup和View的触摸事件相关API默认都返回false。即，上面列出的API的默认返回值都是false。**  
(02) **触摸事件的分发顺序是经过MyActivity --> MyViewGroup --> MyView。**  
    它们的触摸事件的入口都是dispatchTouchEvent()，即MyActivity将事件分发给MyViewGroup时，是通过MyActivity.dispatchTouchEvent()去调用MyViewGroup.dispatchTouchEvent()；同样的，MyViewGroup将事件分发给MyView时，也是通过MyViewGroup.dispatchTouchEvent()去调用MyView.dispatchTouchEvent()。  
    它们的对触摸事件的处理都是在onTouchEvent()中完成的。也就是说，会在它们的dispatchTouchEvent()中，皆会调用(它们各自的)onTouchEvent()来对事件进行处理。onTouchEvent()返回true，就表示消费了个事件，或者说接受了个事件。  
    前面说过消息的分发顺序是MyActivity --> MyViewGroup --> MyView。如果想在MyActivity中进行消息拦截(即，MyActivity不想将消息分发给它包含的视图)，则需要重载dispatchTouchEvent()。如果想在MyViewGroup中进行消息拦截(即，MyViewGroup收到触摸事件之后，不想分发给它的子视图)，则一般都会通过覆盖onInterceptTouchEvent()，并在onInterceptTouchEvent()返回true来拦截消息。  
(03) **MyViewGroup和MyView都没有接受ACTION_DOWN事件的话；那么，ACTION_MOVE和ACTION_UP等触摸事件也就不会发送给它们。**  

Activity中ACTION_DOWN的流程图如下：

<a href="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/event/event01.jpg"><img src="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/event/event01.jpg" alt="" /></a>


<a name="anchor2"></a>
# 2. 示例源码

点击查看：[触摸事件示例1的源码][link_android_event_sample01]

<a name="anchor2_1"></a>
## 2.1 MyActivity的源码

    public class MyActivity extends Activity {
        private static final String TAG = "##skywang-MyActivity";

        @Override
        public void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.main);
        }   
     
        @Override
        public boolean dispatchTouchEvent(MotionEvent event) {
            String actionName = Utils.getActionName(event);
            Log.d(TAG, "dispatchTouchEvent(start) :"+actionName);
            boolean ret = super.dispatchTouchEvent(event);
            Log.d(TAG, "dispatchTouchEvent( end ) :"+actionName+", ret="+ret);
            return ret;
        }   

        @Override
        public boolean onTouchEvent(MotionEvent event) {
            String actionName = Utils.getActionName(event);
            Log.d(TAG, "onTouchEvent(start) :"+actionName);
            boolean ret = super.onTouchEvent(event);
            Log.d(TAG, "onTouchEvent( end ) :"+actionName+", ret="+ret);
            return ret;
        }
    }

说明：MyActivity的layout是main.xml。虽然它覆盖了dispatchTouchEvent()和onTouchEvent()方法；但它们都是在调用父类的对应的方法的基础之上，添加了打印信息而已。


### 2.1.1 getActionName的源码

    public static String getActionName(MotionEvent event) {
        final int action = event.getAction(); 
        if (action == MotionEvent.ACTION_DOWN) {
            return "DOWN";     
        } else if (action == MotionEvent.ACTION_MOVE) {
            return "MOVE";     
        } else if (action == MotionEvent.ACTION_UP) {
            return "UP";       
        } else if (action == MotionEvent.ACTION_CANCEL) { 
            return "CANCEL";   
        } else {               
            return "NULL";     
        }
    }

### 2.1.2 main.xml的源码

    <?xml version="1.0" encoding="utf-8"?>
    <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
        android:orientation="vertical"
        android:layout_width="fill_parent"
        android:layout_height="fill_parent" >

        <TextView                  
            android:layout_width="fill_parent"
            android:layout_height="wrap_content"
            android:text="Hello World, EventTest-Default" />
      
        <com.skw.eventtest.MyViewGroup  
            android:layout_width="fill_parent"
            android:layout_height="400dp"
            android:background="#cccccc"
            android:layout_gravity="center"
            android:gravity="center" >
          
            <com.skw.eventtest.MyView       
                android:layout_width="200dp"
                android:layout_height="100dp"
                android:background="#451c0a" />
      
        </com.skw.eventtest.MyViewGroup>
    </LinearLayout>

说明：main.xml中包含了MyViewGroup，而MyViewGroup中又包含了MyView。



<a name="anchor2_2"></a>
## 2.2 MyViewGroup的源码

    public class MyViewGroup extends LinearLayout {
        private static final String TAG = "##skywang-MyViewGroup";
        
        public MyViewGroup(Context context){
            super(context);        
        } 
              
        public MyViewGroup(Context context, AttributeSet attrs) {
            super(context, attrs); 
        }     
      
        @Override
        public boolean dispatchTouchEvent(MotionEvent event) {
            String actionName = Utils.getActionName(event);
            Log.d(TAG, "dispatchTouchEvent(start) :"+actionName);
            boolean ret = super.dispatchTouchEvent(event);
            Log.d(TAG, "dispatchTouchEvent( end ) :"+actionName+", ret="+ret);
            return ret;
        }

        @Override
        public boolean onTouchEvent(MotionEvent event) {
            String actionName = Utils.getActionName(event);
            Log.d(TAG, "onTouchEvent(start) :"+actionName);
            boolean ret = super.onTouchEvent(event);
            Log.d(TAG, "onTouchEvent( end ) :"+actionName+", ret="+ret);
            return ret;
        }

        @Override
        public boolean onInterceptTouchEvent(MotionEvent event) {
            String actionName = Utils.getActionName(event);
            Log.d(TAG, "onInterceptTouchEvent(start) :"+actionName);
            boolean ret = super.onInterceptTouchEvent(event);
            Log.d(TAG, "onInterceptTouchEvent( end ) :"+actionName+", ret="+ret);
            return ret;
        }
    }

说明：MyViewGroup继承于ViewGroup。虽然它覆盖了dispatchTouchEvent(), onTouchEvent()和onInterceptTouchEvent()方法；但它们都是在调用父类的对应的方法的基础之上，添加了打印信息而已。



<a name="anchor2_3"></a>
## 2.3 MyView的源码

      
    public class MyView extends View {
        private static final String TAG = "##skywang-MyView";
      
        public MyView(Context context) {
            super(context);        
        }
        
        public MyView(Context context, AttributeSet attrs) {
            super(context, attrs); 
        } 
              
        @Override
        public boolean dispatchTouchEvent(MotionEvent event) {
            String actionName = Utils.getActionName(event);
            Log.d(TAG, "dispatchTouchEvent(start) :"+actionName);
            boolean ret = super.dispatchTouchEvent(event);
            Log.d(TAG, "dispatchTouchEvent( end ) :"+actionName+", ret="+ret);
            return ret;
        }

        @Override
        public boolean onTouchEvent(MotionEvent event) {
            String actionName = Utils.getActionName(event);
            Log.d(TAG, "onTouchEvent(start) :"+actionName);
            boolean ret = super.onTouchEvent(event);
            Log.d(TAG, "onTouchEvent( end ) :"+actionName+", ret="+ret);
            return ret;
        }
    }



<a name="anchor3"></a>
# 3. 运行结果

## 3.1 ACTION_DOWN事件

点击MyView所在的区域，ACTION_DOWN相关的log如下：

D/##skywang-MyActivity( 1935): dispatchTouchEvent(start) :DOWN
D/##skywang-MyViewGroup( 1935): dispatchTouchEvent(start) :DOWN
D/##skywang-MyViewGroup( 1935): onInterceptTouchEvent(start) :DOWN
D/##skywang-MyViewGroup( 1935): onInterceptTouchEvent( end ) :DOWN, ret=false
D/##skywang-MyView( 1935): dispatchTouchEvent(start) :DOWN
D/##skywang-MyView( 1935): onTouchEvent(start) :DOWN
D/##skywang-MyView( 1935): onTouchEvent( end ) :DOWN, ret=false
D/##skywang-MyView( 1935): dispatchTouchEvent( end ) :DOWN, ret=false
D/##skywang-MyViewGroup( 1935): onTouchEvent(start) :DOWN
D/##skywang-MyViewGroup( 1935): onTouchEvent( end ) :DOWN, ret=false
D/##skywang-MyViewGroup( 1935): dispatchTouchEvent( end ) :DOWN, ret=false
D/##skywang-MyActivity( 1935): onTouchEvent(start) :DOWN
D/##skywang-MyActivity( 1935): onTouchEvent( end ) :DOWN, ret=false
D/##skywang-MyActivity( 1935): dispatchTouchEvent( end ) :DOWN, ret=false


说明：很显然，ACTION_DOWN的流程如下：  
(01) MyActivity收到ACTION_DOWN，**进入MyActivity.dispatchTouchEvent()**。  
(02) MyActivity.dispatchTouchEvent()对ACTION_DOWN触摸事件进行分发，将消息传递给MyViewGroup。即，**进入MyViewGroup.dispatchTouchEvent()**。  
(03) MyViewGroup.dispatchTouchEvent()会调用MyViewGroup.onInterceptTouchEvent()检查自己有没有对触摸事件进行拦截。即先**进入MyViewGroup.onInterceptTouchEvent()**。
(04) 紧接着，MyViewGroup会**退出MyViewGroup.onInterceptTouchEvent()**。因为MyViewGroup没有对触摸事件进行拦截，MyViewGroup会继续分发事件。  
(05) MyViewGroup将触摸事件分发给MyView，即**进入MyView.dispatchTouchEvent()**。  
(06) MyView会调用onTouchEvent()对触摸事件进行处理，即**进入MyView.onTouchEvent()**  。
(07) 紧接着，MyView会**退出MyView.onTouchEvent()**。返回false给MyView.dispatchTouchEvent()。  
(08) MyView收到MyView.onTouchEvent()的返回值之后，**退出MyView.dispatchTouchEvent()**。返回false给MyViewGroup的MyViewGroup.dispatchTouchEvent()，表示MyView没有接受该触摸事件。  
(09) MyViewGroup则得知MyView没有接受该触摸事件之后，将自己当作一个View，调用View.dispatchTouchEvent()；View.dispatchTouchEvent()接着就会**进入MyViewGroup.onTouchEvent()**。  
(10) 紧接着，就会**退出MyViewGroup.onTouchEvent()**。MyViewGroup.onTouchEvent()没有消费该触摸事件，因此返回false。   
(11) 然后，View.dispatchTouchEvent()就会结束，并返回false。接着，MyViewGroup就会**退出MyViewGroup.dispatchTouchEvent()**。并返回false。  
(12) MyActivity在得知MyViewGroup没有接受该触摸事件之后，就会调用**进入MyActivity.onTouchEvent**。  
(13) 紧接着，就会**退出MyActivity.onTouchEvent**，并返回false。  
(14) 至此，MyActivity.dispatchTouchEvent()才结束。因此，会**退出MyActivity.dispatchTouchEvent()**，并返回false。

说明：触摸事件的分发顺序是经过MyActivity --> MyViewGroup --> MyView。

## 3.2 ACTION_MOVE事件

点击MyView所在的区域，ACTION_MOVE相关的log如下：

D/##skywang-MyActivity( 1935): dispatchTouchEvent(start) :MOVE
D/##skywang-MyActivity( 1935): onTouchEvent(start) :MOVE
D/##skywang-MyActivity( 1935): onTouchEvent( end ) :MOVE, ret=false
D/##skywang-MyActivity( 1935): dispatchTouchEvent( end ) :MOVE, ret=false

说明：由于MyViewGroup和MyView都没有接受ACTION_DOWN事件，因此ACTION_MOVE事件就不会再分发给它们。



## 3.3 ACTION_UP事件

点击MyView所在的区域，ACTION_UP相关的log如下：

D/##skywang-MyActivity( 1935): dispatchTouchEvent(start) :UP
D/##skywang-MyActivity( 1935): onTouchEvent(start) :UP
D/##skywang-MyActivity( 1935): onTouchEvent( end ) :UP, ret=false
D/##skywang-MyActivity( 1935): dispatchTouchEvent( end ) :UP, ret=false

说明：由于MyViewGroup和MyView都没有接受ACTION_DOWN事件，因此ACTION_UP事件就不会再分发给它们。



[link_android_event_sample01]: https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/events/01_event_default/EventTest
[link_android_event_sample02]: https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/events/02_event_view/EventTest
[link_android_event_sample03]: https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/events/03_event_viewgourp/EventTest
[link_android_event_sample04]: https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/events/04_event_viewgourp/EventTest
[link_android_event_sample05]: https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/events/05_event_viewgourp/EventTest

