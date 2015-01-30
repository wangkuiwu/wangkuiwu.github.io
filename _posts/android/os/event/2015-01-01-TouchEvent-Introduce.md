---
layout: post
title: "Android 触摸事件机制(一) 简介"
description: "android"
category: android
tags: [android]
date: 2015-01-01 09:01
---


> 本系列文章将介绍Android中触摸事件，即Touch Event的传递机制。

> 注意：本文是基于Android 4.4.2版本进行介绍的！

> **目录**  
> **1**. [触摸事件概述](#anchor1)  
> **2**. [Activity, ViewGroup, View中的触摸事件API](#anchor2)  
> **3**. [OnTouchListener接口](#anchor3)  



<a name="anchor1"></a>
# 1. 触摸事件概述

本文介绍的触摸事件API和接口主要是：dispatchTouchEvent(), onTouchEvent(), onInterceptTouchEvent()和OnTouchListener接口。这些内容中，最复杂的莫过于dispatchTouchEvent(), onTouchEvent()和onInterceptTouchEvent()这三者之间的关系。如果你能认真读完本系列文章，相信对它们之间的关系，它们的原理和用法，很有很清晰的认识。

本文先对这些接口做个大致介绍，建立一个整体概念。后续再通过阅读Activity, View和ViewGroup中触摸事件API的源码，来对认识这些API；最后，再通过几个示例来进一步了解它们，同时也了解它们的用法。


<a name="anchor2"></a>
# 2. Activity, ViewGroup, View中的触摸事件API

**1. Activity中的触摸事件API**  
public boolean dispatchTouchEvent(MotionEvent ev)；  
public boolean onTouchEvent(MotionEvent ev); 

**2. ViewGroup中的触摸事件API**  
public boolean dispatchTouchEvent(MotionEvent ev)；  
public boolean onTouchEvent(MotionEvent ev);  
public boolean onInterceptTouchEvent(MotionEvent ev);

**3. View中的触摸事件API**  
public boolean dispatchTouchEvent(MotionEvent ev)；  
public boolean onTouchEvent(MotionEvent ev); 


下面简单的说明一下涉及到的三个API的作用。

**dispatchTouchEvent**：它是传递触摸事件的接口。  
(01) Activity将触摸事件传递给ViewGroup，ViewGroup将触摸事件传递给另一个ViewGroup，以及ViewGroup将触摸事件传递给View；这些都是通过dispatchTouchEvent()来传递的。  
(02) dispatchTouchEvent(), onInterceptTouchEvent(), onTouchEvent()以及onTouch()它们之间的联系，都是通过dispatchTouchEvent()体现的。它们都是在dispatchTouchEvent()中调度的！因此，理解dispatchTouchEvent()是理解Android事件机制的关机；而其中，最关机的就是ViewGroup中的dispatchTouchEvent()  
(03) 返回值：true，表示触摸事件被消费了；false，则表示触摸事件没有被消费。  
**onTouchEvent**：它是处理触摸事件的接口。  
(01) 无论是Activity, ViewGroup还是View，对触摸事件的处理，基本上都是在onTouchEvent()中进行的。因此，我们说它是处理触摸事件的接口。  
(02) 返回值：返回true，表示触摸事件被它处理过了；或者，换句话说，表示它消费了触摸事件。否则，表示它没有消费该触摸事件。  
**onInterceptTouchEvent**：它是拦截触摸事件的接口。  
(01) 只有ViewGroup中才有该接口。如果ViewGroup不想将触摸事件传递给它的子View，则可以在onInterceptTouchEvent中进行拦截。  
(02) 返回值：true，表示ViewGroup拦截了该触摸事件；那么，该事件就不会分发给它的子View或者子ViewGroup。否则，表示ViewGroup没有拦截该事件，该事件就会分发给它的子View和子ViewGroup。



<a name="anchor3"></a>
# 3. OnTouchListener接口

OnTouchListener一个interface接口，它是在View中声明的。OnTouchListener中只包含了onTouch()函数。  
那么，onTouch()和onTouchEvent()有什么相同和不同点呢？

**相同点**  
onTouch()与onTouchEvent()都是用户处理触摸事件的API。

**不同点**  
(01)，onTouch()是View专门提供给用户的接口，目的是为了方便用户自己处理触摸事件。而onTouchEvent()是Android系统自己实现的接口。  
(02)，onTouch()的优先级比onTouchEvent()的优先级更高。  
    dispatchTouchEvent()中分发事件的时候，会先将事件分配给onTouch()进行处理，然后才分配给onTouchEvent()进行处理。  如果onTouch()对触摸事件进行了处理，并且返回true；那么，该触摸事件就不会分配在分配给onTouchEvent()进行处理了。只有当onTouch()没有处理，或者处理了但返回false时，才会分配给onTouchEvent()进行处理。

关于它们之间的内容，在讲解View的dispatchTouchEvent()时，会详细说明。
