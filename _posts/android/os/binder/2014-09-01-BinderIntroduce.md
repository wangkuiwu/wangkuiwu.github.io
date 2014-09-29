---
layout: post
title: "Android Binder机制(一) Binder简介"
description: "android"
category: android
tags: [android]
date: 2014-09-01 09:01
---


> 本章对Android中的IPC机制(即，进程间通信机制)--Binder机制，进行简单的介绍。





# 2. Android为什么要采用Binder机制。

Android系统是基于Linux内核而打造的操作系统。在Linux中，已经拥有"管道/消息队列/共享内存/信号量/Socket等"IPC通信手段，但是，Google还是使用了Binder来实现进程间通信，这说明Binder具有无可比拟的优势。

下面从 实用性(Client-Server架构)/传输效率/操作复杂度/安全性 等几方面来分析Binder的优势：   
(01) **Binder能够很好的实现Client-Server架构**。  Google想提供一套基于Client-Server的通信方式。但是，目前Linux支持的"传统的管道/消息队列/共享内存/信号量/Socket等"IPC通信手段中，只有Socket支持Client-Server的通信方式。但是，Socket主要用于网络间通信以及本机中进程间的低速通信，它的传输效率太低。
(02) **Binder的传输效率和可操作性很好**。  前面已经说了，Socket传输效率很低；消息队列和管道又采用存储-转发方式，数据从Client传输到Server需要经过2次内存拷贝！即，数据先从发送方的缓存区(即，Linux中的用户存储空间)拷贝到内核开辟的缓存区(即，Linux中的内核存储空间)中，然后再从内核缓存区拷贝到接收方的缓存区(即，Linux中的用户存储空间)，至少有两次拷贝过程。而采用Binder机制只需要经过1次拷贝，从发送方的缓存区拷贝到内核的缓存区，而接收方的缓存区与内核的缓存区是映射到同一块物理地址的，因此只需要1次拷贝即可。关于Binder机制的传输过程，后面会详细介绍。  而共享内存虽然需要0次拷贝，但是共享内存操作复杂。  
(03) **Binder机制的安全性很高**。 传统IPC没有任何安全措施，完全依赖上层协议来确保。首先传统IPC的接收方无法获得对方进程可靠的UID/PID（用户ID/进程ID），从而无法鉴别对方身份。Binder机制为每个进程分配了UID/PID来作为鉴别身份的标示。




Binder采用面向对象的设计思想，一个Binder实体可以发送给其它进程从而建立许多跨进程的引用；另外这些引用也可以在进程之间传递，就象java里将一个引用赋给另一个引用一样。为Binder在不同进程中建立引用必须有驱动的参与，由驱动在内核创建并注册相关的数据结构后接收方才能使用该引用。


