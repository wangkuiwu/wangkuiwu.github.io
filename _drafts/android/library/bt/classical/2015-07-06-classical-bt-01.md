---
layout: post
title: "Android中的经典蓝牙(一) 基本介绍"
description: "android"
category: android
tags: [android]
date: 2015-07-06 09:01
---

> 本文介绍Android中经典蓝牙的基本情况。

> **目录**  
[1. 经典蓝牙简介](#anchor1)  
[2. Android中的经典蓝牙](#anchor2)  
[3. 参考链接](#anchor3)  


<a name="anchor1"></a>
# 1. 经典蓝牙简介

Android 4.3开始支持蓝牙4.0。也就是从Android才开始支持蓝牙BLE技术(Bluetooth low energy，即蓝牙低能耗)、iBeacon等技术。  
在Android 4.3以前，都只支持经典蓝牙。这里的经典蓝牙是指蓝牙4.0以前的蓝牙，包括蓝牙2.0、蓝牙2.1、蓝牙3.0。

本文的主要内容就是介绍Android中经典蓝牙有关的一些内容。通常情况下，我们对蓝牙的操作主要有：  
(1) 开启和关闭蓝牙  
(2) 蓝牙扫描  
(4) 蓝牙配对  
(5) 蓝牙连接  
(5) 蓝牙收发数据  

后面会通过示例依次对这些功能进行说明。


<a name="anchor2"></a>
# 2. Android中的经典蓝牙

目前还没有仔细研究蓝牙的系统架构，后续有必要的话，再深入研究并给出框架图。这里只是站在应用的角度来看待问题，主要说明Android中蓝牙涉及到的类  
(1) BluetoothAdapter：蓝牙适配器。可以看作是本机蓝牙设备。  
(2) BuletoothDevice：蓝牙(远程)设备。  
(3) BluetoothSocket：客户端的蓝牙socket。  
(4) BluetoothServerSocket：服务端的蓝牙socket。  


<a name="anchor3"></a>
# 3. 参考链接

1. [Android官网关于经典蓝牙的介绍](http://developer.android.com/guide/topics/connectivity/bluetooth.html)
2. [android蓝牙通信](http://blog.csdn.net/eric41050808/article/details/16967189)
