---
layout: post
title: "Android Binder机制(一) Binder的设计和框架"
description: "android"
category: android
tags: [android]
date: 2014-09-01 09:01
---


> 这是关于Android中Binder机制的一系列纯技术贴。花了一个多礼拜的时间，才终于将其整理完毕。行文于此，以做记录；也是将自己所得与大家分享。  
> 和以往一样，介绍Binder时，先讲解框架，然后再从设计和细节等方面一一展开。若文章若错误或纰漏，请不吝指出。谢谢！

> **目录**  
> **1**. [Android消息机制的架构](#anchor1)  

TAG:SKYWANG-TODO





<a name="anchor1"></a>
# 1. Binder简介

<a name="anchor1_1"></a>
## 1.1 Binder模型

[skywang-todo]

上图中涉及到Binder模型的4类角色：**Binder驱动**，**ServiceManager**，**Server**和**Client**。 因为后面章节讲解Binder时，都是以MediaPlayerService和MediaPlayer为代表进行讲解的；这里就使用MediaPlayerService代表了Server，而MediaPlayer则代表了Client。

Binder机制的目的**是实现IPC(Inter-Process Communication)，即实现进程间通信**。在上图中，由于MediaPlayerService是Server的代表，而MediaPlayer是Client的代表；因此，对于上图而言，Binder机制则表现为"实现MediaPlayerService和MediaPlayer之间的通信"。




<a name="anchor1_2"></a>
## 1.2 Binder驱动存在的原因和意义

在回答"Binder机制中Binder驱动存在的原因和意义"之前，先介绍几个基本的概念。

1. Linux系统中的内存划分

Android是基于Linux内核而打造的操作系统。  
以32位Linux系统而言，它的内存最大是4G。在这4G内存中，0~3G为用户空间，3~4G为内核空间。应用程序都运行在用户空间，而Kernel和驱动都运行在内核空间。用户空间和内核空间若涉及到通信(即，数据交互)，两者不能简单地使用指针传递数据，而必须在"内核"中通过copy_from_user(),copy_to_user(),get_user()或put_user()等函数传递数据。copy_from_user()和get_user()是将内核空间的数据拷贝到用户空间，而copy_to_user()和put_user()则是将用户空间的数据拷贝到内核空间。


2. 进程的基本概念

进程拥有独立的内存单元，它是系统进行资源分配和调度的基本单位。对于Linux系统而言，每一个运行在用户空间的应用程序都可以看作一个进程。  
不同的进程在不同的内存中，因此当一个程序崩溃之后，不会对其它的程序造成影响。


<br/>
通过上面的"Linux的内存划分"和"进程"，我们可以了解到：**应用程序都运行在用户空间，每个应用程序都有它自己独立的内存空间；若不同的应用程序之间涉及到通信，需要通过内核进行中转，因为需要用到内核的copy_from_user()和copy_to_user()等函数。**   
现在，再回到上面的框架图中。图中的ServiceManager, MediaPlayerService和MediaPlayer都位于用户空间，它们是不同的进程。前面说过，Binder机制的最终目的是实现"MediaPlayerService和MediaPlayer这两个不同进程之间的通信"。而这两个不同进程的通信必须要内核进行中转，对于Android而言，在内核中起中转作用便是Binder驱动。那么Binder驱动是如何进行数据中转的呢？这里概括的介绍一下，后面再详细说明。   
Android的通信是基于Client-Server架构的，进程间的通信无非就是Client向Server发起请求，Server响应Client的请求。这里以发起请求为例：当Client向Server发起请求(例如，MediaPlayer向MediaPlayerService发起请求)，Client会先将请求数据从用户空间拷贝到内核空间(将数据从MediaPlayer发给Binder驱动)；数据被拷贝到内核空间之后，再通过驱动程序，将内核空间中的数据拷贝到Server位于用户空间的缓存中(Binder驱动将数据发给MediaPlayerService)。这样，就成功的将Client进程中的请求数据传递到了Server进程中。

实际上，Binder驱动是整个Binder机制的核心。除了实现上面所说的数据传输之外，Binder驱动还是实现线程控制(通过中断等待队列实现线程的等待/唤醒)，以及UID/PID等安全机制的保证。



<a name="anchor1_3"></a>
## 1.3 ServiceManager存在的原因和意义

Binder是要实现Android的C-S架构的，即Client-Server架构。而ServiceManager，是以服务管理者的身份存在的。

ServiceManager也是运行在用户空间的一个独立进程。  
(01) 对于Binder驱动而言，**ServiceManager是一个守护进程，更是Android系统各个服务的管理者**。Android系统中的各个服务，都是添加到ServiceManager中进行管理的，而且每个服务都对应一个服务名。当Client获取某个服务时，则通过服务名来从ServiceManager中获取相应的服务。  
(02) 对于MediaPlayerService和MediaPlayer而言，**ServiceManager是一个Server服务端，是一个服务器**。当要将MediaPlayerService等服务添加到ServiceManager中进行管理时，ServiceManager是服务器，它会收到MediaPlayerService进程的添加服务请求。当MediaPlayer等客户端要获取MediaPlayerService等服务时，它会向ServiceManager发起获取服务请求。

当MediaPlayer和MediaPlayerService通信时，MediaPlayerService是服务端；而当MediaPlayerService则ServiceManager通信时，ServiceManager则是服务端。这样，就造就了ServiceManager的特殊性。于是，ServiceManager被赋予了特殊的句柄值0，通过这个特殊的句柄就能获取ServiceManager对象。这个后面会在代码中在进行详细介绍。





<a name="anchor1_4"></a>
## 1.4 为什么采用Binder机制，而不是其他的IPC通信方式

前面说过，Android是在Linux内核的基础上设计的。而在Linux中，已经拥有"管道/消息队列/共享内存/信号量/Socket等等"众多的IPC通信手段；但是，Google为什么单单选择了Binder，而不是其它的IPC机制呢？

这肯定是因为Binder具有无可比拟的优势。下面就从 "实用性(Client-Server架构)/传输效率/操作复杂度/安全性" 等几方面进行分析。

### 第一. Binder能够很好的实现Client-Server架构

对于Android系统，Google想提供一套基于Client-Server的通信方式。  
例如，将"电池信息/马达控制/wifi信息/多媒体服务"等等不同的服务都由不同的Server提供，当Client需要获取某Server的服务时，只需要Client向Server发送相应的请求，Server收到请求之后进行处理，处理完毕再将反馈内容发送给Client。

但是，目前Linux支持的"传统的管道/消息队列/共享内存/信号量/Socket等"IPC通信手段中，只有Socket是Client-Server的通信方式。但是，Socket主要用于网络间通信以及本机中进程间的低速通信，它的传输效率太低。


### 第二. Binder的传输效率和可操作性很好

前面已经说了，Socket传输效率很低，已经被排除。而消息队列和管道又采用存储-转发方式，使用它们进行IPC通信时，需要经过2次内存拷贝！效率太低！

为什么消息队列和管道的数据传输需要经过2次内存拷贝呢？ 首先，数据先从发送方的缓存区(即，Linux中的用户存储空间)拷贝到内核开辟的缓存区(即，Linux中的内核存储空间)中，是第1次拷贝。接着，再从内核缓存区拷贝到接收方的缓存区(也是Linux中的用户存储空间)，这是第2次拷贝。  
而采用Binder机制的话，则只需要经过1次内存拷贝即可！ 即，从发送方的缓存区拷贝到内核的缓存区，而接收方的缓存区与内核的缓存区是映射到同一块物理地址的，因此只需要1次拷贝即可。

至于共享内存呢，虽然使用它进行IPC通信时进行的内存拷贝次数是0。但是，共享内存操作复杂，也将它排除。


### 第三. Binder机制的安全性很高

传统IPC没有任何安全措施，完全依赖上层协议来确保。传统IPC的接收方无法获得对方进程可靠的UID/PID(用户ID/进程ID)，从而无法鉴别对方身份。而Binder机制则为每个进程分配了UID/PID来作为鉴别身份的标示，并且在Binder通信时会根据UID/PID进行有效性检测。




<a name="anchor1_5"></a>
## 1.5 Binder中各角色之间关系

先看看下面的关系图

[skywang-todo(关系图)]

在解释上面的图之前，先解释图中涉及到的几个非常重要的概念。

1. Binder实体

  Binder实体，是各个Server以及ServiceManager在内核中的存在形式。  
  Binder实体的本质上内核中binder_node结构体的对象，它的作用是在内核中保存Server和ServiceManager的信息(例如，Binder实体中保存了Server对象在用户空间的地址)。简言之，Binder实体是Server在Binder驱动中的存在形式，通过Binder实体就能够找到Server对象。  
  在上图中，Server和ServiceManager在Binder驱动中都对应的存在一个Binder实体。

2. Binder引用

   说到Binder实体，就不得不说"Binder引用"。所谓Binder引用，实际上是binder_ref结构体对象，它的作用是在引用"Binder实体"。换句话说，每一个Binder引用都是某一个Binder实体的引用，通过Binder引用就可以在Binder驱动中找到对应的Binder实体。如果将Server看作是Binder实体的话，那么Binder引用就相当于Client中保存的Server对象，通过该Client中保存的Server对象，Client就能想Server发送请求。  


   Binder实体和Binder引用都是内核(即，Binder驱动)中的数据结构。每一个Server在内核中就表现为一个Binder实体，而每一个Client则表现为一个Binder引用。这样，每个Binder引用都对应一个Binder实体，而每个Binder实体则可以多个Binder引用。

3. 远程服务

   Server都是以服务的形式注册到ServiceManager中进行管理的。如果将Server本身看作是"本地服务"的话，那么Client中的"远程服务"就是本地服务的代理，通过该远程服务Client就能和Server进行通信。

<br/>
理解上面3个概念之后，再通过各角色之间的通信来解析该关系图。

**ServiceManager守护进程**  
ServiceManager是用户空间的一个守护进程。当该应用程序启动时，它会和Binder驱动进行通信，告诉Binder驱动它是服务管理者；对Binder驱动而言，它则会新建ServiceManager对应的Binder实体，并将该Binder实体设为全局变量。为什么要将它设为全局变量呢？这点应该很容易理解--因为Client和Server都需要和ServiceManager进行通信，不将它设为全局变量的话，怎么找到ServiceManager呢！


**Server注册到ServiceManager中**  
Server首先会向Binder驱动发起注册请求，而Binder驱动在收到该请求之后就将该请求转发给ServiceManager进程。但是Binder驱动怎么才能知道该请求是要发给ServiceManager的呢？这是因为Server在发送请求的时候，会告诉Binder驱动这个请求是交给0号Binder引用对应的进程来进行处理的。而Binder驱动中指定了这个0号引用与ServiceManager关联的。  
在Binder驱动转发该请求之前，它其实还做了一件很重要的事：那就是当它知道该请求是由一个Server发送的时候，它会在ServiceManager的"保存Binder实体的红黑树"中查找该Server对应的Binder实体；找不到的话，就新建该Server对应的Binder实体，并将其添加到"ServiceManager的保存Binder实体的红黑树"中。简言之，Binder驱动创建了Server对应的Binder实体。  
当ServiceManager收到Binder驱动转发的注册请求之后，它就将该Server的相关信息注册到"Binder引用组成的单链表"中。这里所说的Server相关信息主要包括两部分：Server对应的服务名 + Server对应的Binder实体的一个Binder引用。


**Client获取远程服务**  
Client要和某个Server通信，需要先获取到该Server的远程服务。那么Client是如何获取到Server的远程服务的呢？  
Client首先会向Binder驱动发起获取服务的请求。Binder驱动在收到该请求之后也是该请求转发给ServiceManager进程。ServiceManager在收到Binder驱动转发的请求之后，会从"Binder引用组成的单链表"中找到要获取的Server的相关信息。至于ServiceManager是如何从单链表中找到Server的呢？答案是Client发送的请求数据中，会包括它要获取的Server的服务名；而ServiceManager正是根据这个服务名来找到Server的。  
接下来，ServiceManager通过Binder驱动将Server信息反馈给Client的。它反馈的信息是Server对应的Binder驱动的Binder引用。而Client在收到该Server的Binder引用相关信息之后，就根据该Binder引用创建一个Server的远程服务。Client调用该Server远程服务的接口，就相当于在调用Server的服务接口一样；因为Client调用该Server的远程服务接口时，该远程服务会对应的通过Binder驱动和真正的Server进行交互，从而执行相应的动作。


<a name="anchor2"></a>
# 2. Binder框架解析


## 2.1 Binder角色

在了解了Binder机制的四种角色和它的优缺点之后，接下来可以开始讲解Binder的框架和设计了。

在了解Binder设计的由来之前，先了解各个层次中的Binder角色的很有必要的。 

先看看下面一张图

[skywang-todo(角色图)]

前面说过，ServiceManager是服务管理者。下面就通过"Server将自己注册到ServiceManager中"以及"Client获取Server"这两个流程来理清图中各个角色之间的关系。

1. Server将自己注册到ServiceManager中

  由于Server和ServiceManager是用户空间的两个不同进程，它们之间的通信需要Binder驱动的协助。实际上，Server和ServiceManager在Binder驱动中都是以一个"Binder实体"的形式存在的。所谓Binder实体，实际上是binder_node结构体对象，它的作用是在Kernel中保存Server和ServiceManager的信息；在Binder实体中保存了Server对象在用户空间的地址。简言之，Binder实体是Server在Binder驱动中的存在形式，通过Binder实体就能够找到Server对象。      
   说到Binder实体，就不得不说"Binder引用"。所谓Binder引用，实际上是binder_ref结构体对象，它的作用是在引用"Binder实体"。换句话说，每一个Binder引用都是某一个Binder实体的引用。这样，通过Binder引用就可以在Binder驱动中找到对应的Binder实体。  
   Binder实体和Binder引用都是Binder驱动(即Kernel)中的数据结构。每一个Server在Kernel中就表现为一个Binder实体，而每一个Client则表现为一个Binder引用。这样，每个Binder引用都对应一个Binder实体，而每个Binder实体则可以多个Binder引用。

   理解了Binder实体和Binder引用的概念之后，再接着说Server是如何将自己注册到ServiceManager中的。  
   首先，Server向Binder驱动发送注册服务请求，该请求内容包含了两个非常重要的信息：Server对应的服务名+Server的本地Binder对象。这两个信息的作用稍候再介绍。当Binder驱动收到注册服务请求之后，将该请求转发给ServiceManager；



   
   简单来说，通过ServiceManager中的"Server服务的名称"能够找到"Binder引用的地址"，而通过"Binder引用的地址"又可以找到"它对应的Binder实体"；通过"Binder实体"就可以找到Server端在用户空间的地址，即可以找到Server对象。

  Server发起注册。
  在Server将自己保存到Binder实体中之后，Binder驱动再通知ServiceManager：将"Server服务的名称"连同"Binder引用的地址"一起保存到ServiceManager守护进程中。"Server服务的名称"的作用显而易见，它是为了方便查找和获取Server对象，通过该名称就可以找到对应的Server。而"Binder引用"呢？它是Binder驱动中的binder_ref结构体对象，一个Binder引用是一个Binder实体的引用，通过Binder引用就可以在Binder驱动中找到对应的Binder实体。 简单来说，通过ServiceManager中的"Server服务的名称"能够找到"Binder引用的地址"，而通过"Binder引用的地址"又可以找到"它对应的Binder实体"；通过"Binder实体"就可以找到Server端在用户空间的地址，即可以找到Server对象。  



由于Server和ServiceManager是用户空间的两个不同进程，它们之间的通信需要Binder驱动的协助。首先，Server向Binder驱动发送注册服务请求，该请求内容包含了两个非常重要的信息：Server对应的服务名+Server的本地Binder对象。这两个信息的作用稍候再介绍。当Binder驱动收到注册服务请求之后，将该请求转发给ServiceManager；


。当Client需要向Server发送请求时，需要先获取Server的接入点。如上图所示，接入点就是Server对应的远程代理。Client能够获取到该远程代理，获取的方式就是Client向ServiceManager发送获取服务的请求，而且发送请求时指定要获取的服务名。当ServiceManager收到该请求之后，就找到ServiceManager中保存。



前面说过，Binder设计要提供C-S架构。  
试想，如果C-S架构中的Client和Server属于同一进程：那么，当Client要向Server发送请求时，只需要在Client端先获取相应的Server端对象；然后，再通过Server对象调用Server的相应接口即可。但是，Binder机制中涉及到的Client和Server是位于不同的进程中的，这也就意味着，不可能直接获取到Server对象。那么怎么办呢？ 那就需要ServiceManager和Binder驱动的参与。总的来说，提供C-S架构，需要做到以下亮点：


## 第一，Server要提供接入点

[skywang-todo(远程代理图)]

前面说过，Server会先将自己注册到ServiceManager中。当Client需要向Server发送请求时，需要先获取Server的接入点。如上图所示，接入点就是Server对应的远程代理。Client能够获取到该远程代理，获取的方式就是Client向ServiceManager发送获取服务的请求，而且发送请求时指定要获取的服务名。当ServiceManager收到该请求之后，就找到ServiceManager中保存。

如MediaPlayerService






## 第一，Server要提供接入点

  Client要向Server发送请求，而又不能直接获取到Server对象；那么，Server必须要提供一个接入点，让Client能够访问到Server。  
  实际上，这个接入点是"Server在Binder驱动中的Binder引用的地址"。至于什么是Binder引用的地址，请继续往下看。

  前面说过，Server是通过注册到ServiceManager中进行管理的。下面就谈谈这个流程。  
  Server端要将自己注册到ServiceManager中，而Server端和ServiceManager又是不同的进程，它们之间只能通过Binder驱动进行交互；于是Server先将自己保存到Binder驱动中，Server在Binder驱动中是以一个Binder实体的形式存在的。所谓Binder实体，实际上是binder_node结构体对象，在Binder实体中保存了Server对象在用户空间的地址。简言之，Binder实体是Server在Binder驱动中的存在形式，通过Binder实体就能够找到Server对象。      
  在Server将自己保存到Binder实体中之后，Binder驱动再通知ServiceManager：将"Server服务的名称"连同"Binder引用的地址"一起保存到ServiceManager守护进程中。"Server服务的名称"的作用显而易见，它是为了方便查找和获取Server对象，通过该名称就可以找到对应的Server。而"Binder引用"呢？它是Binder驱动中的binder_ref结构体对象，一个Binder引用是一个Binder实体的引用，通过Binder引用就可以在Binder驱动中找到对应的Binder实体。 简单来说，通过ServiceManager中的"Server服务的名称"能够找到"Binder引用的地址"，而通过"Binder引用的地址"又可以找到"它对应的Binder实体"；通过"Binder实体"就可以找到Server端在用户空间的地址，即可以找到Server对象。  
  通过上面的两步，就成功的将Server注册到了ServiceManager中。也就成功的提供了Server端的进入点：Client通过"Server服务的名称"来向ServiceManager发起获取Server的请求即可获取到对应的Server。

  下面再来简单说说Client是如何获取到Server接入点的。  
  Client要获取Server对象，就发送一个获取服务的请求给Binder驱动，而且该请求数据中必须包含"要获取的Server服务的名称"。Binder驱动将该请求交给ServiceManager进行处理。ServiceManager在收到获取服务请求之后，根据请求数据中的"Server服务的名称"找到"Binder引用的地址"，然后将该"Binder引用的地址"返回给Binder驱动。Binder驱动收到ServiceManager的反馈之后，再将该"Binder引用的地址"返回给Client。这样，Client就成功获取到了"Server对应的Binder实体的Binder引用"。当Client需要向Server发送请求时，即可以通过它所保存的"Binder引用的地址"将请求发给Server。



### 第二，通信协议，即Server和Client之间的请求和反馈要遵循一定的格式。

下面来一点点讲解。









# Service Manager守护进程

先和Binder驱动通信；Binder驱动根据"MediaPlayerService的Binder引用的描述"获取到"MediaPlayerService进程"，进而唤醒"MediaPlayerService进程"并向该进程提交请求(发送事务到MediaPlayerService的待处理事务列表中)；"MediaPlayerService进程"被唤醒后，就对事务进行处理(从待处理事务列表中读取事务，然后发送给C++层的MediaPlayerService消息队列进行处理)；处理完毕，"MediaPlayerService进程"就将结果反馈给MediaPlayer进程。


