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

> 注意：本文是基于Android 4.4.2版本进行介绍的！

> **目录**  
> **1**. [Binder架构解析](#anchor1)  
> **1.1**. [Binder模型](#anchor1_1)  
> **1.2**. [Binder驱动存在的原因和意义](#anchor1_2)  
> **1.3**. [ServiceManager存在的原因和意义](#anchor1_3)  
> **1.4**. [为什么采用Binder机制，而不是其他的IPC通信方式](#anchor1_4)  
> **1.5**. [Binder中各角色之间关系](#anchor1_5)  
> **2**. [Binder设计解析](#anchor2)  
> **2.1**. [Binder设计](#anchor2_1)  
> **2.1.1**. [内核空间的Binder设计](#anchor2_1_1)  
> **2.1.2**. [用户空间的Binder设计](#anchor2_1_2)  
> **2.2**. [Binder通信](#anchor2_2)  
> **2.2.1**. [Binder通信模型](#anchor2_2_1)  
> **2.2.2**. [Binder通信数据](#anchor2_2_2)  



<a name="anchor1"></a>
# 1. Binder架构解析 

<a name="anchor1_1"></a>
## 1.1 Binder模型

<a href="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/binder/binder_frame.jpg"><img src="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/binder/binder_frame.jpg" alt="" /></a>

上图中涉及到Binder模型的4类角色：**Binder驱动**，**ServiceManager**，**Server**和**Client**。 因为后面章节讲解Binder时，都是以MediaPlayerService和MediaPlayer为代表进行讲解的；这里就使用MediaPlayerService代表了Server，而MediaPlayer则代表了Client。

Binder机制的目的**是实现IPC(Inter-Process Communication)，即实现进程间通信**。在上图中，由于MediaPlayerService是Server的代表，而MediaPlayer是Client的代表；因此，对于上图而言，Binder机制则表现为"实现MediaPlayerService和MediaPlayer之间的通信"。




<a name="anchor1_2"></a>
## 1.2 Binder驱动存在的原因和意义

在回答"Binder机制中Binder驱动存在的原因和意义"之前，先介绍几个基本的概念。

### 1. Linux系统中的内存划分

Android是基于Linux内核而打造的操作系统。  
以32位Linux系统而言，它的内存最大是4G。在这4G内存中，0~3G为用户空间，3~4G为内核空间。应用程序都运行在用户空间，而Kernel和驱动都运行在内核空间。用户空间和内核空间若涉及到通信(即，数据交互)，两者不能简单地使用指针传递数据，而必须在"内核"中通过copy_from_user(),copy_to_user(),get_user()或put_user()等函数传递数据。copy_from_user()和get_user()是将内核空间的数据拷贝到用户空间，而copy_to_user()和put_user()则是将用户空间的数据拷贝到内核空间。


### 2. 进程的基本概念

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

当MediaPlayer和MediaPlayerService通信时，MediaPlayerService是服务端；而当MediaPlayerService则ServiceManager通信时，ServiceManager则是服务端。这样，就造就了ServiceManager的特殊性。于是，在Binder驱动中，将句柄0指定为ServiceManager对应的句柄，通过这个特殊的句柄就能获取ServiceManager对象。这部分的知识后面会详细介绍。




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

<a href="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/binder/binder_4relationship.jpg"><img src="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/binder/binder_4relationship.jpg" alt="" /></a>

在解释上面的图之前，先解释图中涉及到的几个非常重要的概念。

**1. Binder实体**

  Binder实体，是各个Server以及ServiceManager在内核中的存在形式。  
  Binder实体实际上是内核中binder_node结构体的对象，它的作用是在内核中保存Server和ServiceManager的信息(例如，Binder实体中保存了Server对象在用户空间的地址)。简言之，Binder实体是Server在Binder驱动中的存在形式，内核通过Binder实体可以找到用户空间的Server对象。  
  在上图中，Server和ServiceManager在Binder驱动中都对应的存在一个Binder实体。

**2. Binder引用**

   说到Binder实体，就不得不说"Binder引用"。所谓Binder引用，实际上是内核中binder_ref结构体的对象，它的作用是在表示"Binder实体"的引用。换句话说，每一个Binder引用都是某一个Binder实体的引用，通过Binder引用可以在内核中找到它对应的Binder实体。  
   如果将Server看作是Binder实体的话，那么Client就好比Binder引用。Client要和Server通信，它就是通过保存一个Server对象的Binder引用，再通过该Binder引用在内核中找到对应的Binder实体，进而找到Server对象，然后将通信内容发送给Server对象。


   Binder实体和Binder引用都是内核(即，Binder驱动)中的数据结构。每一个Server在内核中就表现为一个Binder实体，而每一个Client则表现为一个Binder引用。这样，每个Binder引用都对应一个Binder实体，而每个Binder实体则可以多个Binder引用。

**3. 远程服务**

   Server都是以服务的形式注册到ServiceManager中进行管理的。如果将Server本身看作是"本地服务"的话，那么Client中的"远程服务"就是本地服务的代理。如果你对代理模式比较熟悉的话，就很容易理解了，远程服务就是本地服务的一个代理，通过该远程服务Client就能和Server进行通信。

<br/>
理解上面3个概念之后，下面再通过几个典型的通信示例来解析该关系图。

**ServiceManager守护进程**  
  ServiceManager是用户空间的一个守护进程。当该应用程序启动时，它会和Binder驱动进行通信，告诉Binder驱动它是服务管理者；对Binder驱动而言，它则会新建ServiceManager对应的Binder实体，并将该Binder实体设为全局变量。为什么要将它设为全局变量呢？这点应该很容易理解--因为Client和Server都需要和ServiceManager进行通信，不将它设为全局变量的话，怎么找到ServiceManager呢！


**Server注册到ServiceManager中**  
  Server首先会向Binder驱动发起注册请求，而Binder驱动在收到该请求之后就将该请求转发给ServiceManager进程。但是Binder驱动怎么才能知道该请求是要转发给ServiceManager的呢？这是因为Server在发送请求的时候，会告诉Binder驱动这个请求是交给0号Binder引用对应的进程来进行处理的。而Binder驱动中指定了0号引用是与ServiceManager对应的。  
  在Binder驱动转发该请求之前，它其实还做了两件很重要的事：(01) 当它知道该请求是由一个Server发送的时候，它会新建该Server对应的Binder实体。 (02) 它在ServiceManager的"保存Binder引用的红黑树"中查找是否存在该Server的Binder引用；找不到的话，就新建该Server对应的Binder引用，并将其添加到"ServiceManager的保存Binder引用的红黑树"中。简言之，Binder驱动会创建Server对应的Binder实体，并在ServiceManager的红黑树中添加该Binder实体的Binder引用。  
  当ServiceManager收到Binder驱动转发的注册请求之后，它就将该Server的相关信息注册到"Binder引用组成的单链表"中。这里所说的Server相关信息主要包括两部分：Server对应的服务名 + Server对应的Binder实体的一个Binder引用。


**Client获取远程服务**  
  Client要和某个Server通信，需要先获取到该Server的远程服务。那么Client是如何获取到Server的远程服务的呢？  
  Client首先会向Binder驱动发起获取服务的请求。Binder驱动在收到该请求之后也是该请求转发给ServiceManager进程。ServiceManager在收到Binder驱动转发的请求之后，会从"Binder引用组成的单链表"中找到要获取的Server的相关信息。至于ServiceManager是如何从单链表中找到需要的Server的呢？答案是Client发送的请求数据中，会包括它要获取的Server的服务名；而ServiceManager正是根据这个服务名来找到Server的。  
  接下来，ServiceManager通过Binder驱动将Server信息反馈给Client的。它反馈的信息是Server对应的Binder实体的Binder引用信息。而Client在收到该Server的Binder引用信息之后，就根据该Binder引用信息创建一个Server对应的远程服务。这个远程服务就是Server的代理，Client通过调用该远程服务的接口，就相当于在调用Server的服务接口一样；因为Client调用该Server的远程服务接口时，该远程服务会对应的通过Binder驱动和真正的Server进行交互，从而执行相应的动作。






<a name="anchor2"></a>
# 2. Binder设计解析

有了上面Binder模型的理论基础，接下来就可以逐步来讲解Binder的设计了。实际上，在设计C-S架构时，要考虑以下两个非常重要的因素。

**第一，Server要提供接入点**

  如果C-S架构中的Client和Server属于同一进程的话，那么Client和Server之间的通信将非常容易。只需要在Client端先获取相应的Server端对象；然后，再通过Server对象调用Server的相应接口即可。但是，Binder机制中涉及到的Client和Server是位于不同的进程中的，这也就意味着，不可能直接获取到Server对象。那么怎么办呢？ 那就需要Server提供一个接入点给Client。  
  这个接入点就是**"Server的远程服务代理"**！  
  Client能够获取到Server的远程服务，它就相当于Server的代理。Client要和Server通信时，它只需要调用该远程服务的相应接口即可，其他的工作都交给远程服务来处理。远程服务收到Client请求之后，会和Binder驱动通信；因为远程服务中有Server在Binder驱动中的Binder引用信息，因此远程服务就能轻易的找到对应的Server，进而将Client的请求内容发送Server。



**第二，通信协议**

  Binder机制中，涉及到大量的"内核的Binder驱动 和 用户空间的引用程序"之间的通信。需要指定对应的通信协议，确保通信的安全和正常。关于这部分，稍候再详细展开。



有了上面的两个中心思想之后，再来对Binder驱动的设计和协议进行逐步展开。

<a name="anchor2_1"></a>
## 2.1 Binder设计

讲解Binder设计时，分为"内核空间"和"用户空间"这两部分进程讲解。内核空间就是Binder驱动中的Binder设计，而用户空间则是Android的C++层中的Binder设计。


<a name="anchor2_1_1"></a>
### 2.1.1 内核空间的Binder设计

内核空间的Binder设计涉及到3个非常重要的结构体：binder_proc，binder_node和binder_ref。由于本文的重点是介绍Binder机制的理论知识，因此，在这里我并不打算展开这3个结构体对它们进行详细介绍。当然，后面会再撰文对这些类进行详细说明。这里只需要了解个大概即可。

binder_proc是描述进程上下文信息的，每一个用户空间的进程都对应一个binder_proc结构体。  
binder_node是Binder实体对应的结构体，它是Server在Binder驱动中的体现。  
binder_ref是Binder引用对应的结构体，它是Client在Binder驱动中的体现。

<a href="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/binder/binder_kernel_ds01.jpg"><img src="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/binder/binder_kernel_ds01.jpg" alt="" /></a>

如上图所示，binder_proc中包含了3棵红黑树。  
(01) Binder实体红黑树是保存"binder_proc对应的进程"所包含的Binder实体的，而Binder实体是与Server的服务对应的。可以将Binder实体红黑树理解为Server进程中包行的Server服务的红黑树。  
(02) 图中有两棵Binder引用红黑树，这两棵树所包含的Binder引用都是一样的。不同的是，红黑树的排序基准不同，一个是以Binder实体来排序，而另一个则是以Binder引用描述(Binder引用描述实际上就是一个32位的整型数)来排序。以Binder引用描述的红黑树是为了方便进行快速查找。



<a href="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/binder/binder_kernel_ds02.jpg"><img src="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/binder/binder_kernel_ds02.jpg" alt="" /></a>

上图是描述Binder驱动中Binder实体结构体的。如图所示，Binder实体中有一个Binder引用的哈希表，专门来存放该Binder实体的Binder引用。这也如我们之前所说，每个Binder实体则可以多个Binder引用，而每个Binder引用则都只对应一个Binder实体。



<a name="anchor2_1_2"></a>
### 2.1.2 用户空间的Binder设计

<a href="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/binder/binder_user_ds01.jpg"><img src="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/binder/binder_user_ds01.jpg" alt="" /></a>

上面是用户空间中Binder模型图，该图仅仅只描述出Server的相关类图，并没有Client部分。不过不要紧，通过这个Server的模型图，就能理清用户空间的Binder框架。

  前面说过，Server是以服务的形式注册到ServiceManager中，而Server在Client中则是以远程服务的形式存在的。因此，这个图的主干就是理清楚本地服务和远程服务这两者之间的关系。  
  "本地服务"就是Server提供的服务本身，而"远程服务"就是服务的代理；"服务接口"则是抽象出了它们的通用接口。这3个角色都是通用的，对于不同的服务而言，它们的名称都不相同。例如，对于MediaPlayerService服务而言，本地服务就是MediaPlayerService自身，远程服务是BpMediaPlayerService，而服务接口是IMediaPlayerService。当Client需要向MediaPlayerService发送请求时，它需要先获取到服务的代理(即，远程服务对象)，也就是BpMediaPlayerService实例，然后通过该实例和MediaPlayerService进行通信。

  图中的ProcessState和IPCThreadState都是采用单例模式实现的，它们的实例都是全局的，而且只有唯一一个。

(01) 当Server启动之后，它会先将自己注册到ServiceManager中。注册时，Binder驱动会创建Server对应的Binder实体，并将"Server对应的本地服务对象的地址"保存到Binder实体中。注册成功之后，Server就进入消息循环，等待Client的请求。  
(02) 当Client需要和Server通信时，会先获取到Server接入点，即获取到远程服务对象；而且Client要获取的远程服务对象是"服务接口"类型的。Client向ServiceManager发送获取服务的请求时，会通过IPCThreadState和Binder驱动进行通信；当ServiceManager反馈之后，IPCThreadState会将ServiceManager反馈的"Server的Binder引用信息"保存BpBinder中(具体来说，BpBinder的mHandle成员保存的就是Server的Binder引用信息)。然后，会根据该BpBinder对象创建对应的远程服务。这样，Client就获取到了远程服务对象，而且远程服务对象的成员中保存了Server的Binder引用信息。   
(03) 当Client获取到远程服务对象之后，它就可以轻松的和Server进行通信了。当它需要向Server发送请求时，它会调用远程服务接口；远程服务能够获取到BpBinder对象，而BpBinder则通过IPCThreadState和Binder驱动进行通信。由于BpBinder中保存了Server在Binder驱动中的Binder引用；因此，IPCThreadState和Binder驱动通信时，是知道该请求是需要传给哪个Server的。Binder驱动通过Binder引用找到对应的Binder实体，然后将Binder实体中保存的"Server对应的本地服务对象的地址"返回给用户空间。当IPC收到Binder驱动反馈的内容之后，它从内容中找到"Server对应的本地服务对象"，然后调用该对象的onTransact()。不同的本地服务都可以实现自己的onTransact()；这样，不同的服务就可以按照自己的需求来处理请求。




<a name="anchor2_2"></a>
## 2.2 Binder通信

Binder通信协议是基于Command-Reply的方式的。

<a name="anchor2_2_1"></a>
### 2.2.1 Binder通信模型

下面是Client和Server的交互模型图。

<a href="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/binder/binder_communication.jpg"><img src="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/binder/binder_communication.jpg" alt="" /></a>

图中的原理很简单。  
(01) Server进程启动之后，会进入中断等待状态，等待Client的请求。  
(02) 当Client需要和Server通信时，会将请求发送给Binder驱动。  
(03) Binder驱动收到请求之后，会唤醒Server进程。  
(04) 接着，Binder驱动还会反馈信息给Client，告诉Client：它发送给Binder驱动的请求，Binder驱动已经收到。  
(05) Client将请求发送成功之后，就进入等待状态。等待Server的回复。  
(06) Binder驱动唤醒Server之后，就将请求转发给Server进程。  
(07) Server进程解析出请求内容，并将回复内容发送给Binder驱动。  
(08) Binder驱动收到回复之后，唤醒Client进程。  
(09) 接着，Binder驱动还会反馈信息给Server，告诉Server：它发送给Binder驱动的回复，Binder驱动已经收到。  
(10) Server将回复发送成功之后，再次进入等待状态，等待Client的请求。  
(11) 最后，Binder驱动将回复转发给Client。



<a name="anchor2_2_2"></a>
### 2.2.2 Binder通信数据

<a href="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/binder/binder_data.jpg"><img src="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/binder/binder_data.jpg" alt="" /></a>

上面是用户空间和内核空间进行交互时，数据的打包方式。例如，当Client向Server发送请求时，Client会将数据打包成上诉格式，然后通过ioctl()发送给Binder驱动。根据数据的层次，从外到里分为3层进行说明。

**第一层**：这是用户空间的进程调用ioctl(fd,BINDER_WRITE_READ,&bwr)时传递给Binder驱动的信息。fd是Binder驱动的文件句柄，BINDER_WRITE_READ是ioctl()的一个标识，而bwr是传递的数据，它对应是途中的binder_write_read结构体的指针。binder_write_read中以write_开头的是保存请求数据的，而read_开头的是保存反馈数据的。其中，write_size是请求数据的大小，write_buffer是请求数据的内容，而write_consumed是用来记录请求数据中已经被Binder驱动处理过的数据的大小。

**第二层**：这层的数据是"事务指令"+"binder_transaction_data结构体"组成的。图中给出的事务指令是BC_TRANSACTION，表示该事务是请求；如果是回复，则是BR_开头的，例如BR_TRANSACTION。binder_transaction_data是描述事务交互数据的结构体；例如，target是指定事务目标，用来表示这个事务是交给谁进行处理的；code是事务编码，用来表示这是一个什么样的事务(例如，注册服务事务/获取服务事务等待)；data是保存事务中具体数据的内存地址。

**第三层**：这层是有效数据。如果该请求是传递给ServiceManager进行处理的，则有效数据是：消息头+"Server的相关信息"。消息头是用来进行有效性检查的，而"Server的相关信息"则是请求要处理的信息。


