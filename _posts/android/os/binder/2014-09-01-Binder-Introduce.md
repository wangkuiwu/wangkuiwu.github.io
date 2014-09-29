---
layout: post
title: "Android Binder机制(一) Binder的设计和框架"
description: "android"
category: android
tags: [android]
date: 2014-09-01 09:01
---


> 这是关于Android中Binder机制的一系列纯技术贴。我花了一个多礼拜的时间，才终于将其整理完毕。行文于此，以做记录；也是将自己所得分享出来。  
> 和以往一样，介绍Binder时，先讲解框架，然后再从设计和细节等方面一一展开。若文章若错误或纰漏，请不吝指出。谢谢！

> **目录**  
> **1**. [Android消息机制的架构](#anchor1)  

TAG:SKYWANG-TODO





<a name="anchor1"></a>
# 1. Binder架构

[skywang-todo]

上面是Binder的架构图。Binder机制共包括4种角色：**Binder驱动**，**ServiceManager**，**Server**和**Client**。 为了便于理解，这里用MediaPlayerService代表了Server，而MediaPlayer则代表了Client。

Binder机制的目的**是实现IPC(Inter-Process Communication)，即实现进程间通信**。在上面的图中，由于MediaPlayerService是Server的代表，而MediaPlayer是Client的代表；因此，对于上图而言，Binder机制则表现为"实现MediaPlayerService和MediaPlayer之间的通信"。


<a name="anchor1_1"></a>
## 1.1 Binder驱动存在的原因和意义

在回答"Binder机制中Binder驱动存在的原因和意义"之前，先介绍几个基本的概念。

1. Linux系统中的内存划分

Android是基于Linux内核而打造的操作系统。  
以32位Linux系统而言，它的内存最大是4G。在这4G内存中，0~3G为用户空间，3~4G为内核空间。应用程序都运行在用户空间，而Kernel和驱动都运行在内核空间。用户空间和内核空间若涉及到通信(即，数据交互)，两者不能简单地使用指针传递数据，而必须在内核中调用copy_from_user(),copy_to_user(),get_user()或put_user()等函数传递数据。copy_from_user()和get_user()是将内核空间的数据拷贝到用户空间，而copy_to_user()和put_user()则是将用户空间的数据拷贝到内核空间。


2. 进程的基本概念

进程是拥有独立的内存单元的，它是系统进行资源分配和调度的基本单位。对于Linux系统而言，每一个运行在用户空间的应用程序都可以看作一个进程。  
不同的进程在不同的内存中，因此当一个程序崩溃之后，不会对其它的程序造成影响。


了解了"Linux的内存划分"和"进程"之后，再回到上面的框架图中。图中的ServiceManager, MediaPlayerService和MediaPlayer都位于用户空间，它们是不同的进程。前面说过，Binder机制的最终目的是实现"MediaPlayerService和MediaPlayer这两个不同进程之间的通信"。而进程又是独立的内存单元，它们之间不能直接通信；因此，便需要通过Binder驱动来辅助数据交互。当Client向Server发起请求(例如，MediaPlayer向MediaPlayerService发起请求)，Client会先将请求数据从用户空间拷贝到内核空间(将数据从MediaPlayer发给Binder驱动)；然后，再将内核空间中的数据拷贝到Server位于用户空间的缓存中(Binder驱动将数据发给MediaPlayerService)。

实际上，Binder驱动是整个Binder机制的核心。除了实现上面所说的数据传输之外，Binder驱动还是实现线程控制(通过中断等待队列实现线程的等待/唤醒)，以及UID/PID等安全机制的保证。



<a name="anchor1_2"></a>
## 1.2 ServiceManager存在的原因和意义

Binder是要实现Android的C-S架构的，即Client-Server架构。而ServiceManager的存在，则是为了更好的实现C-S架构。

ServiceManager也是运行在用户空间的一个独立进程。  
(01) 对于Binder驱动而言，**ServiceManager是一个守护进程，更是Android系统各个服务的管理者**。Android系统中的各个服务，都是添加到ServiceManager中进行管理的，而且每个服务都有一个名称。当Client获取某个服务时，则通过服务名来从ServiceManager中获取相应的服务。  
(02) 对于MediaPlayerService和MediaPlayer而言，**ServiceManager是一个Server服务端，是一个服务器**。当要将MediaPlayerService添加到ServiceManager中进行管理时，ServiceManager是服务器，它会收到MediaPlayerService进程的添加服务请求。当MediaPlayer要获取MediaPlayerService服务时，它会向ServiceManager发起获取服务请求。

当MediaPlayer和MediaPlayerService通信时，MediaPlayerService是服务端；而当MediaPlayerService则ServiceManager通信时，ServiceManager则是服务端。这样，就造就了ServiceManager的特殊性。于是，ServiceManager被赋予了特殊的句柄值0，通过这个特殊的句柄就能获取ServiceManager对象。





<a name="anchor2"></a>
## 2. 为什么是Binder机制，而不是其他的IPC通信方式

前面说过，Android是在Linux内核的基础上设计的。而在Linux中，已经拥有"管道/消息队列/共享内存/信号量/Socket等等"众多的IPC通信手段；但是，Google为什么还是单单使用了Binder呢？

这肯定是因为Binder具有无可比拟的优势。下面就从 "实用性(Client-Server架构)/传输效率/操作复杂度/安全性" 等几方面进行分析。  

1. **Binder能够很好的实现Client-Server架构**

对于Android系统，Google想提供一套基于Client-Server的通信方式。  
例如，将"电池信息/马达控制/wifi信息/多媒体服务"等等不同的服务都有不同的Server提供，当Client需要获取某Server的服务时，只需要Client向Server发送相应的请求，Server收到请求之后进行处理，处理完毕再将反馈内容发送给Client。

但是，目前Linux支持的"传统的管道/消息队列/共享内存/信号量/Socket等"IPC通信手段中，只有Socket支持Client-Server的通信方式。但是，Socket主要用于网络间通信以及本机中进程间的低速通信，它的传输效率太低。


2. **Binder的传输效率和可操作性很好**

前面已经说了，Socket传输效率很低，已经被排除。而消息队列和管道又采用存储-转发方式，使用它们进行IPC通信时，需要经过2次内存拷贝！效率太低！

为什么消息队列和管道的数据传输需要经过2次内存拷贝呢？ 先，数据先从发送方的缓存区(即，Linux中的用户存储空间)拷贝到内核开辟的缓存区(即，Linux中的内核存储空间)中，是第1次拷贝。接着，再从内核缓存区拷贝到接收方的缓存区(也是Linux中的用户存储空间)，这是第2次拷贝。  
而采用Binder机制的话，则只需要经过1次内存拷贝即可！ 即，从发送方的缓存区拷贝到内核的缓存区，而接收方的缓存区与内核的缓存区是映射到同一块物理地址的，因此只需要1次拷贝即可。

至于共享内存呢，虽然使用它进行IPC通信时进行的内存拷贝次数是0。但是，共享内存操作复杂，也将它排除。  


3. **Binder机制的安全性很高**

传统IPC没有任何安全措施，完全依赖上层协议来确保。传统IPC的接收方无法获得对方进程可靠的UID/PID(用户ID/进程ID)，从而无法鉴别对方身份。而Binder机制则为每个进程分配了UID/PID来作为鉴别身份的标示，并且在Binder通信时会根据UID/PID进行有效性检测。







<a name="anchor3"></a>
# 3. Binder设计原理

在了解了Binder的架构和优缺点之后，接下来开始讲解Binder的设计。

前面说过，Binder设计要实现Android的C-S架构。实现C-S架构，则需要做到以下几点：  
第一，Server端提供接入点，让Client能够获取到Server对象。  
第二，Client端要能够获取到Server接入点，进而向Server发送请求。  
第三，要有相信的通信协议，即Server和Client之间的请求和反馈要遵循一定的格式。

下面来一点点讲解。


# 3.1 Server端提供接入点

Server端提供的接入点是保存在ServiceManager中的。当Server启动时，Server会将自己注册到ServiceManager中，注册信息包括"Server的名字"和"Server对象本身"。








Binder采用面向对象的设计思想，一个Binder实体可以发送给其它进程从而建立许多跨进程的引用；另外这些引用也可以在进程之间传递，就象java里将一个引用赋给另一个引用一样。为Binder在不同进程中建立引用必须有驱动的参与，由驱动在内核创建并注册相关的数据结构后接收方才能使用该引用。









# Service Manager守护进程

先和Binder驱动通信；Binder驱动根据"MediaPlayerService的Binder引用的描述"获取到"MediaPlayerService进程"，进而唤醒"MediaPlayerService进程"并向该进程提交请求(发送事务到MediaPlayerService的待处理事务列表中)；"MediaPlayerService进程"被唤醒后，就对事务进行处理(从待处理事务列表中读取事务，然后发送给C++层的MediaPlayerService消息队列进行处理)；处理完毕，"MediaPlayerService进程"就将结果反馈给MediaPlayer进程。


