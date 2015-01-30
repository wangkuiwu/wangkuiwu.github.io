---
layout: post
title: "Android 触摸事件机制(二) Activity中触摸事件详解"
description: "android"
category: android
tags: [android]
date: 2015-01-02 09:01
---


> 本文将对Activity中触摸事件相关的内容进行介绍，重点介绍的是Activity中与触摸事件相关的两个API：dispatchTouchEvent()和onTouchEvent()。

> 注意：本文是基于Android 4.4.2版本进行介绍的！

> **目录**  
> **1**. [Activity中触摸事件的概述](#anchor1)  
> **2**. [Activity中触摸事件的源码解析](#anchor2)  
> **2.1**. [Activity中的dispatchTouchEvent](#anchor2_1)  
> **2.2**. [Activity中的onTouchEvent](#anchor2_2)  



<a name="anchor1"></a>
# 1. Activity中触摸事件的概述

  Activity中与触摸事件相关API主要是dispatchTouchEvent()和onTouchEvent()。dispatchTouchEvent()是传递触摸事件的API，而onTouchEvent()则是Activity处理触摸事件的API。

  Activity就是dispatchTouchEvent()将触摸事件传递给它所包含的根视图，从而实现将触摸事件传递给View或ViewGroup进行处理。  
  而在onTouchEvent()在是Activity自己对触摸事件的处理。例如，如果Activity是一个Dialog主题，即Activity相当于一个对话框；那么当onTouchEvent()收到点击事件，并且该点击事件的坐标在Activity之外的时候，onTouchEvent()就会结束Activity。


<a name="anchor2"></a>
# 2. Activity中触摸事件的源码解析

<a name="anchor2_1"></a>
## 2.1 Activity中的dispatchTouchEvent

    public boolean dispatchTouchEvent(MotionEvent ev) {
        // onUserInteraction默认不执行任何动作。
        // 它是提供给客户的接口。
        if (ev.getAction() == MotionEvent.ACTION_DOWN) {
            onUserInteraction();
        }
        // 这里会调用到ViewGroup的dispatchTouchEvent()，
        // 即会调用Activity包含的根视图的dispatchTouchEvent()。
        if (getWindow().superDispatchTouchEvent(ev)) {
            return true;
        }
        // 如果superDispatchTouchEvent()返回false，
        // 即Activity的根视图以及根视图的子视图都没有拦截该事件的话，则调用Activity的onTouchEvent()
        return onTouchEvent(ev);
    }

说明：该代码定义在frameworks/base/core/java/android/app/Activity.java中。  
Activity通过调用dispatchTouchEvent()将触摸事件分发给Activity所包含的视图；如果Activity中的视图都没有对触摸事件进行拦截的话，则调用Activity的onTouchEvent()对触摸事件进行处理。  
下面，先看看Activity是如何通过superDispatchTouchEvent()将事件分发给它所包含的View的。


### 2.1.1 Activity中的getWindow()

    private Window mWindow;

    public Window getWindow() {
        return mWindow;
    }

    final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config) {
    
        ...

        mWindow = PolicyManager.makeNewWindow(this);

        ...
    }

说明：getWindow()返回的是mWindow对象，而mWindow是在attach()中初始化的。attach()是Activity被加载时调用的，具体是如何attact()的，不是我们关心的重点；这里只需要了解，Activity被加载时，attach()会被执行即可。  
接着，我们就看看PolicyManager.makeNewWindow()是如何实现的。



### 2.1.2 PolicyManager中的makeNewWindow()

    public static Window makeNewWindow(Context context) {
        return sPolicy.makeNewWindow(context);
    }   

    private static final String POLICY_IMPL_CLASS_NAME =
        "com.android.internal.policy.impl.Policy";
            
    private static final IPolicy sPolicy;
            
    static {
        // Pull in the actual implementation of the policy at run-time
        try {
            Class policyClass = Class.forName(POLICY_IMPL_CLASS_NAME);
            sPolicy = (IPolicy)policyClass.newInstance();
        } catch (ClassNotFoundException ex) {
            throw new RuntimeException(
                    POLICY_IMPL_CLASS_NAME + " could not be loaded", ex);
        } catch (InstantiationException ex) {
            throw new RuntimeException(
                    POLICY_IMPL_CLASS_NAME + " could not be instantiated", ex);
        } catch (IllegalAccessException ex) {
            throw new RuntimeException(
                    POLICY_IMPL_CLASS_NAME + " could not be instantiated", ex);
        }
    }   

说明：该代码定义在frameworks/base/core/java/com/android/internal/policy/PolicyManager.java中。  
makeNewWindow()是调用的sPolicy.makeNewWindow()，而sPolicy是个静态变量，它的实现也是在静态代码块中。因此，在PolicyManager.java加载的时候，sPolicy就会被初始化为policyClass.newInstance()。而policyClass是通过Class得到的Policy对象。  
也就是说，PolicyManager中的makeNewWindow()会调用Policy中的makeNewWindow()。


### 2.1.3 Policy中的makeNewWindow

    public Window makeNewWindow(Context context) {
        return new PhoneWindow(context);
    }   

说明：该代码定义在frameworks/base/policy/src/com/android/internal/policy/impl/Policy.java中。makeNewWindow()会返回PhoneWindow对象。  
回到Activity的dispatchTouchEvent()中，也就是说getWindow()返回的是PhoneWindow对象。接着，就看看superDispatchTouchEvent()的实现。


### 2.1.4 PhoneWindow中的superDispatchTouchEvent

    @Override
    public boolean superDispatchTouchEvent(MotionEvent event) {
        return mDecor.superDispatchTouchEvent(event);
    }    

    private DecorView mDecor;

    private void installDecor() {
        if (mDecor == null) {
            mDecor = generateDecor();
            ...
        }

        ...
    }

    protected DecorView generateDecor() {
        return new DecorView(getContext(), -1); 
    }    


说明：该代码定义在frameworks/base/policy/src/com/android/internal/policy/impl/PhoneWindow.java中。  
superDispatchTouchEvent()会调用mDecor.superDispatchTouchEvent()；而mDecor是DecorView对象。mDecor是在installDecor()中被创建的。总之，PhoneWindow中的superDispatchTouchEvent()会调用DecorView中的superDispatchTouchEvent()。DecorView是PhoneWindow中的内部类，下面看看它的实现。



### 2.1.5 DecorView中的superDispatchTouchEvent
        
    private final class DecorView extends FrameLayout implements RootViewSurfaceTaker {
        ...

        public boolean superDispatchTouchEvent(MotionEvent event) {
            return super.dispatchTouchEvent(event);
        }

        ...
    }

说明：DecorView中的superDispatchTouchEvent()会调用父类的dispatchTouchEvent()。而DecorView的父类是FrameLayout，FrameLayout的父类又是GroupView；因此superDispatchTouchEvent()最终会调用到GroupView的dispatchTouchEvent()。

关于GroupView中的dispatchTouchEvent()的流程，在后面的文章中再来详细介绍！这里重点需要了解：**Activity在通过dispatchTouchEvent()传递触摸事件的时候，会调用到ViewGroup的dispatchTouchEvent()。从而实现，将Activity中的触摸事件传递给它所包含的View或ViewGroup。**


<a name="anchor2_2"></a>
## 2.2 Activity中的onTouchEvent

回顾一下Activity中dispatchTouchEvent()的内容。

    public boolean dispatchTouchEvent(MotionEvent ev) {
        // onUserInteraction默认不执行任何动作。
        // 它是提供给客户的接口。
        if (ev.getAction() == MotionEvent.ACTION_DOWN) {
            onUserInteraction();
        }
        // 这里会调用到ViewGroup的dispatchTouchEvent()，
        // 即会调用Activity包含的根视图的dispatchTouchEvent()。
        if (getWindow().superDispatchTouchEvent(ev)) {
            return true;
        }
        // 如果superDispatchTouchEvent()返回false，
        // 即Activity的根视图以及根视图的子视图都没有拦截该事件的话，则调用Activity的onTouchEvent()
        return onTouchEvent(ev);
    }

(01) 如果superDispatchTouchEvent()返回true的话，dispatchTouchEvent()就直接返回true了，不会执行onTouchEvent()。也就是说，如果Activity将触摸事件分发给它所包含的视图的时候，如果有视图拦截或消费了该事件，就不会轮到Activity来处理该事件了；即，不会执行Activity的onTouchEvent()了。  
(02) 如果superDispatchTouchEvent()返回false的话，意味着，Activity所包含的视图都没有拦截或消费该触摸事件；那么，就会调用Activity的onTouchEvent()来处理触摸事件。

下面就看看onTouchEvent()的代码。


    public boolean onTouchEvent(MotionEvent event) {
        if (mWindow.shouldCloseOnTouch(this, event)) {
            finish();
            return true;
        }    
     
        return false;
    }    

说明：代码很简单。它会先调用mWindow.shouldCloseOnTouch()，如果shouldCloseOnTouch()返回true，则意味着该触摸事件会触发"结束Activity"的动作。那么接下来，就调用finish()来结束Activity，并返回true，表示Activity消费了这个触摸事件。否则的话，就返回false。


### 2.2.1 Window的shouldCloseOnTouch()

前面分析过，mWindow是PhoneWindow对象，而PhoneWindow继承于Window。则mWindow.shouldCloseOnTouch()实际上会调用Window中的shouldCloseOnTouch()。

    public boolean shouldCloseOnTouch(Context context, MotionEvent event) {
        if (mCloseOnTouchOutside && event.getAction() == MotionEvent.ACTION_DOWN
                && isOutOfBounds(context, event) && peekDecorView() != null) {
            return true;
        }    
        return false;
    }    

说明：该代码定义在frameworks/base/core/java/android/view/Window.java中。  
(01) mCloseOnTouchOutside是一个boolean变量，它是由Window的android:windowCloseOnTouchOutside属性值决定。  
(02) isOutOfBounds(context, event)是判断该event的坐标是否在context(对于本文来说就是当前的Activity)之外。是的话，返回true；否则，返回false。  
(03) peekDecorView()则是返回PhoneWindow的mDecor。  
也就是说，如果设置了android:windowCloseOnTouchOutside属性为true，并且当前事件是ACTION_DOWN，而且点击发生在Activity之外，同时Activity还包含视图的话，则返回true；表示该点击事件会导致Activity的结束。



至此，Activity中关于触摸事件的代码就分析完毕了。总结来说：  
(01) **Activity中的dispatchTouchEvent会将触摸事件传递给Activity所包含的视图。具体的实现方式在通过调用到Activity所属Window的superDispatchTouchEvent，进而调用到Window的DecorView的superDispatchTouchEvent，进一步的又调用到ViewGroup的dispatchTouchEvent()。**  
 如果Activity所包含的视图拦截或者消费了该触摸事件的话，就不会再执行Activity的onTouchEvent()；  
 如果Activity所包含的视图没有拦截或者消费该触摸事件的话，则会执行Activity的onTouchEvent()。  
(02) **Activity中的onTouchEvent是Activity自身对触摸事件的处理。如果该Activity的android:windowCloseOnTouchOutside属性为true，并且当前触摸事件是ACTION_DOWN，而且该触摸事件的坐标在Activity之外，同时Activity还包含了视图的话；就会导致Activity被结束。**

