---
layout: post
title: "Android UI系统(四) BootAnimation流程"
description: "android"
category: android
tags: [android]
date: 2014-09-10 09:01
---


> 这是关于Android UI系统中的Gralloc。Android设备都大多有一个显示屏，这个显示屏通过驱动映射到文件节点/dev/graphics/fb0上。通过操作/dev/graphics/fb0就能更新显示内容，

> 注意：本文是基于Android 4.4.2版本进行介绍的！

> **目录**  
> **1**. [Binder架构解析](#anchor1)  



<a name="anchor1"></a>
# 1. 概述

**SurfaceFlinger**

SurfaceFlinger是一个合成器，它管理来自于不同应用的Surface。比如，可能有许多应用同时存在，与此对应的，存在许多独立的Surface需要被渲染。SurfaceFlinger决定屏幕上显示的内容，那些需要被覆盖，进行裁剪。

SurfaceFlinger使用的是OpenGL ES 1.1标准中的函数。为什么呢？如果使用OpenGL ES 2.0，就必须需要支持OpenGL ES 2.0的硬件GPU，这会使系统的启动更加复杂，也会使模拟器的实现更加困难。

**HW Composer**

硬件合成器是Honeycomb引入的一个HAL，SurfaceFlinger使用它，利用硬件资源来加速Surface的合成，比如3D GPU和2D的图形引擎。
