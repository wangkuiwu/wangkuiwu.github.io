---
layout: post
title: "Android Binder机制(二) Binder中的数据结构"
description: "android"
category: android
tags: [android]
date: 2014-09-02 09:02
---


> 本文是一篇参考文献，当读者阅读后面的文章时，可以以此文章为参考。本文列举的数据结构，涵盖了Kernel层，Service Manager守护进程，C++层以及Android Framework层等几个方面。

> **目录**  
> **1**. [Android消息机制的架构](#anchor1)  
> **2**. [C++层](#anchor1)  

> 注意：本文是基于Android 4.4.2版本进行介绍的！

TAG:SKYWANG-TODO


<a name="anchor0"></a>
# Service Manager守护进程



<a name="anchor1"></a>
# 1. Kernel层

<a name="anchor1_1"></a>
## 1.1 binder_proc

binder_proc是描述Binder进程的上下文信息结构体。

    struct binder_proc {
      struct hlist_node proc_node;    // 根据proc_node，可以获取该进程在"全局哈希表binder_procs(统计了所有的binder proc进程)"中的位置
      struct rb_root threads;         // binder_proc进程内用于处理用户请求的线程(关联binder_thread->rb_node)
      struct rb_root nodes;           // binder_proc进程内的binder实体(关联binder_node->rb_node)
      struct rb_root refs_by_desc;    // binder_proc进程内的binder引用，该引用以句柄来排序(关联binder_ref->rb_node_desc)
      struct rb_root refs_by_node;    // binder_proc进程内的binder引用，该引用以它对应的binder实体的地址来排序(关联binder_ref->rb_node)
      int pid;                        // 进程id
      struct vm_area_struct *vma;     // 进程的内核虚拟内存
      struct mm_struct *vma_vm_mm;
      struct task_struct *tsk;        // 进程控制结构体(每一个进程都由task_struct 数据结构来定义)。
      struct files_struct *files;     // 保存了进程打开的所有文件表数据
      struct hlist_node deferred_work_node;
      int deferred_work;
      void *buffer;                   // 该进程映射的物理内存在内核空间中的起始位置
      ptrdiff_t user_buffer_offset;   // 内核虚拟地址与进程虚拟地址之间的差值

      // 内存管理的相关变量
      struct list_head buffers;         // 和binder_buffer->entry关联到同一链表，从而对Binder内存进行管理
      struct rb_root free_buffers;      // 空闲内存，和binder_buffer->rb_node关联。
      struct rb_root allocated_buffers; // 已分配内存，和binder_buffer->rb_node关联。
      size_t free_async_space;

      struct page **pages;            // 映射内存的page页数组，page是描述物理内存的结构体
      size_t buffer_size;             // 映射内存的大小
      uint32_t buffer_free;
      struct list_head todo;          // 该进程的待处理事件列表。
      wait_queue_head_t wait;         // 等待队列。
      struct binder_stats stats;
      struct list_head delivered_death;
      int max_threads;                // 最大线程数。定义threads中可包含的最大进程数。
      int requested_threads;
      int requested_threads_started;
      int ready_threads;
      long default_priority;          // 默认优先级。
      struct dentry *debugfs_entry;
    };

说明：binder_proc定义在drivers/staging/android/binder.c中，可见它是binder.c的私有结构体。它是用来记录进程的上下文信息的，每一个进程打开/dev/binder文件时，都会创建binder_proc结构体变量；从而将进程的相关信息保存在binder_proc中。  由于在结构体中已经有相关成员的注释，这里只对部分比较重要的成员进行说明。  
(01) proc_node, 它的作用是通过proc_node，将该binder_proc添加到"全局哈希表binder_procs(它记录了所有的binder_proc)"。不管binder_procs在Kernel驱动中暂时没有太大用处，所以不用太过关注该成员。  
(02) threads，它是包含该进程内用于处理用户请求的所有线程的红黑树。threads成员和binder_thread->rb_node关联到一棵红黑树，从而将binder_proc和binder_thread关联起来。  
(03) nodes，它是包行该进程内的所有Binder实体所组成的红黑树。nodes成员和binder_node->rb_node关联到一棵红黑树，从而将binder_proc和binder_node关联起来。  
(04) refs_by_desc，它是包行该进程内的所有Binder引用所组成的红黑树。refs_by_desc成员和binder_ref->rb_node_desc关联到一棵红黑树，从而将binder_proc和binder_ref关联起来。  
(05) refs_by_node，它是包行该进程内的所有Binder引用所组成的红黑树。refs_by_node成员和binder_ref->rb_node_node关联到一棵红黑树，从而将binder_proc和binder_ref关联起来。  
(06) buffer，它是该进程内核虚拟内存的起始地址。而user_buffer_offset，则是该内核虚拟地址和进程虚拟地址之间的差值。在Binder驱动中，将进程的内核虚拟地址和进程虚拟地址映射到同一物理页面，该user_buffer_offset则是它们之间的差值；这样，已知其中一个，就可以根据差值算出另外一个。  
(07) todo是该进程的待处理事务列表，而wait则是等待队列。它们的作用是实现进程的等待/唤醒。例如，当Server进程的wait等待队列为空时，Server就进入中断等待状态；当某Client向Server发送请求时，就将该请求添加到Server的todo待处理事务列表中，并尝试唤醒Server等待队列上的线程。如果，此时Server的待处理事务列表不为空，则Server被唤醒后；唤醒后，则取出待处理事务进行处理，处理完毕，则将结果返回给Client。  


<a name="anchor1_2"></a>
## 1.2 binder_buffer

binder_buffer是描述Binder进程所管理的每段内存的结构体。

    struct binder_buffer {
        struct list_head entry;    // 和binder_proc->buffers关联到同一链表，从而使Binder进程对内存进行管理。
        struct rb_node rb_node;    // 和binder_proc->free_buffers或binder_proc->allocated_buffers关联到同一红黑树，从而对已有内存和空闲内存进行管理。

        unsigned free:1;                // 空闲与否的标记
        unsigned allow_user_free:1;
        unsigned async_transaction:1;
        unsigned debug_id:29;

        struct binder_transaction *transaction;

        struct binder_node *target_node;
        size_t data_size;
        size_t offsets_size;
        uint8_t data[0];
    };


说明：binder_buffer是Binder驱动的私有结构体，它定义在drivers/staging/android/binder.c中。Binder驱动将Binder进程的内存分成一段一段进行管理，而这每一段则是使用binder_buffer来描述的。



<a name="anchor1_3"></a>
## 1.3 binder_thread

binder_thread是描述Binder线程的结构体。

    struct binder_thread {
        struct binder_proc *proc;   // 线程所属的Binder进程
        struct rb_node rb_node;     // 红黑树节点，关联到红黑树binder_proc->threads中。
        int pid;                    // 进程id
        int looper;                 // 线程状态。可以取BINDER_LOOPER_STATE_REGISTERED等值
        struct binder_transaction *transaction_stack;   // 正在处理的事务栈
        struct list_head todo;                          // 待处理的事务链表
        uint32_t return_error; /* Write failed, return error code in read buf */
        uint32_t return_error2; /* Write failed, return error code in read */

        wait_queue_head_t wait;                         // 等待队列
        struct binder_stats stats;                      // 保存一些统计信息
    };

说明：binder_thread是Binder驱动的私有结构体，它定义在drivers/staging/android/binder.c中。binder_thread是描述Binder线程相关信息的结构体。  
(01) proc，它是binder_proc(进程上下文信息)结构体对象。目的是保存该线程所属的Binder进程。  
(02) rb_node，它是红黑树节点。通过将rb_node binder关联到红黑树proc->threads中，从而将该线程添加到进程的threads红黑树中进行管理。



<a name="anchor1_4"></a>
## 1.4 binder_node

binder_node是描述Binder实体的结构体。

    struct binder_node {
        int debug_id;
        struct binder_work work;
        union {
            struct rb_node rb_node;         // 如果这个Binder实体还在使用，则将该节点链接到proc->nodes中。
            struct hlist_node dead_node;    // 如果这个Binder实体所属的进程已经销毁，而这个Binder实体又被其它进程所引用，则这个Binder实体通过dead_node进入到一个哈希表中去存放
        };
        struct binder_proc *proc;           // 该binder实体所属的Binder进程
        struct hlist_head refs;             // 该Binder实体的所有引用所组成的链表
        int internal_strong_refs;
        int local_weak_refs;
        int local_strong_refs;
        void __user *ptr;                   // Binder实体在用户空间的地址(为IBinder对象的引用)
        void __user *cookie;                // Binder实体在用户空间的其他数据(为IBinder对象自身)
        unsigned has_strong_ref:1;
        unsigned pending_strong_ref:1;
        unsigned has_weak_ref:1;
        unsigned pending_weak_ref:1;
        unsigned has_async_transaction:1;
        unsigned accept_fds:1;
        unsigned min_priority:8;
        struct list_head async_todo;
    };

说明：binder_node是Binder驱动的私有结构体，它定义在drivers/staging/android/binder.c中。binder_node是描述Binder实体相关信息的结构体。  
(01) rb_node和dead_node属于一个union。如果该Binder实体还在使用，则通过rb-node将该节点链接到proc->nodes红黑树中；否则，则将Binder实体通过dead_node链接到全局哈希表binder_dead_nodes中。  
(02) proc，它是binder_proc(进程上下文信息)结构体对象。目的是保存该Binder对象所属的Binder进程。  
(03) refs，它是该Binder实体的所有引用所组成的链表。




<a name="anchor1_5"></a>
## 1.5 binder_ref

binder_ref是描述Binder引用的结构体。


    struct binder_ref {
        int debug_id;
        struct rb_node rb_node_desc;    // 关联到binder_proc->refs_by_desc红黑树中
        struct rb_node rb_node_node;    // 关联到binder_proc->refs_by_node红黑树中
        struct hlist_node node_entry;   // 关联到binder_node->refs哈希表中
        struct binder_proc *proc;       // 该Binder引用所属的Binder进程
        struct binder_node *node;       // 该Binder引用对应的Binder实体
        uint32_t desc;                  // 描述
        int strong;                     
        int weak;
        struct binder_ref_death *death;
    };

说明：binder_ref是Binder驱动的私有结构体，它定义在drivers/staging/android/binder.c中。它是用来描述Binder引用的相关信息的。  



<a name="anchor1_6"></a>
## 1.6 binder_write_read

binder_write_read是描述Binder读写信息的结构体。

    struct binder_write_read {
        signed long write_size;
        signed long write_consumed;
        unsigned long   write_buffer;
        signed long read_size;
        signed long read_consumed;
        unsigned long   read_buffer;
    };

说明：binder_write_read是Binder驱动和Android的C++层的通信结构体，它记录了Binder读写内容的相关信息。在Kernel中，它定义在drivers/staging/android/binder.h中；它对应在Android中引用在external/kernel-headers/original/linux/binder.h中。  
(01) write_size，是写内容的总大小；write_consumed，是已写内容的大小；write_buffer，是写的内容的虚拟地址。  
(02) read_size，是读内容的总大小；read_consumed，是已读内容的大小；read_buffer，是读的内容的虚拟地址。



<a name="anchor1_7"></a>
## 1.7 flat_binder_object

    struct flat_binder_object {
        unsigned long       type;   // binder类型：BINDER_TYPE_BINDER或BINDER_TYPE_HANDLE等类型
        unsigned long       flags;  // 标记

        union {
            void        *binder;    // 当传递的是Binder实体时使用binder域，它指向Binder实体在应用程序中的地址。
            signed long handle;     // 当传递的是Binder引用时使用handle域，它存放Binder在进程中的引用号。
        };

        /* extra data associated with local object */
        void            *cookie;
    };





TODO
例如，Client(位于用户空间)向Server(位于用户空间)发送请求，Client会将请求的发送内容都写入到write_*中。具体的，将内容的虚拟地址写入到write_buffer中，将内容大小写入到write_size中，而设置write_consumed的大小为0。Client发送请求后，一般都会要求Server进行回复，因此它会设置read_size非零，而设置read_buffer为读取缓存的地址，read_consumed的大小为0。  设置完毕之后，就将请求发送给Kernel，Kernel收到请求后，就从write_buffer中读取请求内容，读取的请求内容大小为write_size；读取数据之后，设置write_consumed的大小为所读数据的大小。Kernel处理完请求之后，将反馈数据填写到read_buffer中，进而将请求反馈给Client。  





<a name="anchor2"></a>
# 2. Service Manager守护进程中的数据结构

<a name="anchor2_1"></a>
## 2.1 binder_state

    struct binder_state
    {
        int fd;           // 文件节点"/dev/binder"的句柄
        void *mapped;     // 映射内存的起始地址
        unsigned mapsize; // 映射内存的大小
    };  

说明：binder_state是Service Manager用来描述打开的"/dev/binder"的信息结构体。


<a name="anchor2_2"></a>
## 2.2 binder_object

binder_object是与flat_binder_object对应的结构体。

    struct binder_object
    {
        uint32_t type;  // 类型
        uint32_t flags;
        void *pointer;
        void *cookie;
    };



<a name="anchor2_3"></a>
## 2.3 binder_txn

binder_txn与binder_transaction_data对应的结构体。

    struct binder_txn
    {
        void *target;
        void *cookie;
        uint32_t code;
        uint32_t flags;

        uint32_t sender_pid;
        uint32_t sender_euid;

        uint32_t data_size;
        uint32_t offs_size;
        void *data;
        void *offs;
    };



<a name="anchor2_4"></a>
## 2.4 svcinfo

    struct svcinfo
    {
        struct svcinfo *next;         // 下一个"服务的信息"
        void *ptr;                    // 服务在Binder驱动中的Binder引用的描述
        struct binder_death death;
        int allow_isolated;
        unsigned len;                 // 服务的名称长度
        uint16_t name[0];             // 服务的名称
    };      

说明：svcinfo定义在frameworks/native/cmds/servicemanager/service_manager.c中。它是Service Manager守护进程的私有结构体。  
svcinfo是保存"注册到Service Manager中的服务"的相关信息的结构体。它是一个单链表，在Service Manager守护进程中的svclist是保存注册到Service Manager中的服务的链表，它就是struct info类型。svcinfo中的next是指向下一个服务的节点，而ptr是该服务在Binder驱动中Binder引用的描述。name则是服务的名称。





<a name="anchor3"></a>
# 3. C++层数据结构

<a name="anchor2_1"></a>
## 3.1 Parcel

    class Parcel {
    public:
        ...

        // 获取数据(返回mData)
        const uint8_t*      data() const;
        // 获取数据大小(返回mDataSize)
        size_t              dataSize() const;
        // 获取数据指针的当前位置(返回mDataPos)
        size_t              dataPosition() const;

    private:
        ...
        
        status_t            mError;                                   
        uint8_t*            mData;            // 数据
        size_t              mDataSize;        // 数据大小
        size_t              mDataCapacity;    // 数据容量
        mutable size_t      mDataPos;         // 数据指针的当前位置
        size_t*             mObjects;         // 对象在mData中的偏移地址
        size_t              mObjectsSize;     // 对象个数
        size_t              mObjectsCapacity; // 对象的容量
            
        ...
    }


