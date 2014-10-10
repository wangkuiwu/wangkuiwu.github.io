---
layout: post
title: "Android Binder机制(三) ServiceManager守护进程"
description: "android"
category: android
tags: [android]
date: 2014-09-03 09:01
---


> ServiceManager是用户空间的一个守护进程，它一直运行在后台。它的职责是管理Binder机制中的各个Server。当Server启动时，Server会将"Server对象的名字"连同"Server对象的信息"一起注册到ServiceManager中；而当Client需要获取Server接入点时，则通过"Server的名字"来从ServiceManager中找到对应的Server。  
> 本文的主要内容就是对ServiceManager进行介绍，通过它的启动流程来分析它是如何成为Server管理者的。

> 注意：本文是基于Android 4.4.2版本进行介绍的！

> **目录**  
> **1**. [ServiceManager流程图](#anchor_1st)  
> **2**. [ServiceManager流程详解](#anchor_2nd)  
> **2.1**. [main()](#anchor1)  
> **2.2**. [binder_open()](#anchor2)  
> **2.3**. [open("/dev/binder")](#anchor3)  
> **2.4**. [mmap()](#anchor4)  
> **2.5**. [binder_become_context_manager()](#anchor5)  
> **2.6**. [ioctl(, BINDER_SET_CONTEXT_MGR,)](#anchor6)  
> **2.7**. [binder_loop()](#anchor7)  
> **2.8**. [for(;;)](#anchor8)  
> **3**. [ServiceManager流程总结](#anchor_3rd)  


<a name="anchor_1st"></a>
# ServiceManager流程图


<a href="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/binder/ServiceManager.jpg"><img src="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/binder/ServiceManager.jpg" alt="" /></a>


上面是ServiceManager的时序图。它启动之后，会先打开"/dev/binder"文件("/dev/binder"是Binder驱动注册的设备节点)。打开文件之后，再告诉Binder驱动，它是Binder的上下文管理者。之后，就进入到了消息循环中。进入消息循环之后，会不断的从Binder的待处理事务队列中读取事务(Binder请求或反馈)，读出事务之后就进行解析，然后交给相应的进程进行处理。若没有事务，则进入等待状态，等待被唤醒。



<a name="anchor_2nd"></a>
# ServiceManager流程详解

<a name="anchor1"></a>
## 1. main()

ServiceManager是一个守护进程。它的main()函数源码如下：

    int main(int argc, char **argv)
    {
        struct binder_state *bs;
        void *svcmgr = BINDER_SERVICE_MANAGER;

        bs = binder_open(128*1024);

        if (binder_become_context_manager(bs)) {
            ALOGE("cannot become context manager (%s)\n", strerror(errno));
            return -1; 
        }   

        svcmgr_handle = svcmgr;
        binder_loop(bs, svcmgr_handler);
        return 0;
    }

说明：该代码在frameworks/native/cmds/servicemanager/service_manager.c中。main()主要进行了三项工作：  
(01) 通过binder_open()打开"/dev/binder"文件，即打开Binder设备文件。  
(02) 调用binder_become_context_manager()，通过ioctl()告诉Binder驱动程序自己是Binder上下文管理者。  
(03) 调用binder_loop()进入消息循环，等待Client的请求。如果没有Client请求，则进入中断等待状态；当有Client请求时，就被唤醒，然后读取并处理Client请求。  

> ServiceManager是如何启动的？  
> 这里简要介绍一下ServiceManager的启动方式。当Kernel启动加载完驱动之后，会启动Android的init程序，init程序会解析init.rc，进而启动init.rc中定义的守护进程。而ServiceManager则正是通过注册在init.rc中，而被启动的。 




<a name="anchor2"></a>
## 2. binder_open()

下面，对main()的逐个步骤进行详细分析。先看看binder_open()，代码如下：

    struct binder_state *binder_open(unsigned mapsize)
    {
        struct binder_state *bs;

        bs = malloc(sizeof(*bs));
        ...

        bs->fd = open("/dev/binder", O_RDWR);
        ...

        bs->mapsize = mapsize;
        bs->mapped = mmap(NULL, mapsize, PROT_READ, MAP_PRIVATE, bs->fd, 0); 
        ...

        return bs;
    }

说明： 该代码定义在frameworks/native/cmds/servicemanager/binder.c中。binder_open的作用是打开"/dev/binder"设备文件，然后调用mmap()将设备文件"/dev/binder"映射到进程空间的起始地址。  
(01) open("/dev/binder", O_RDWR)对应会调用驱动的open函数。  
(02) mmap(NULL, mapsize, PROT_READ, MAP_PRIVATE, bs->fd, 0)对应会调用驱动的mmap函数。第一个参数是映射内存的起始地址，NULL代表让系统自动选定地址；mapsize大小是128*1024B，即128K；PROT_READ表示映射区域是可读的；MAP_PRIVATE表示建立一个写入时拷贝的私有映射，即，当进程中对该内存区域进行写入时，是写入到映射的拷贝中；bs->fd是"/dev/binder"句柄；而0表示偏移。  
(03) binder_state结构体是来保存/dev/binder设备信息的。其中，fd是用来保存文件句柄，mmaped是映射内存的起始地址，mapsize映射内存大小。



<a name="anchor3"></a>
## 3. open("/dev/binder")

### 3.1 Binder驱动注册信息

下面看看open("/dev/binder", O_RDWR)到底做了些什么。先看看下面的代码：


    static const struct file_operations binder_fops = {
      .owner = THIS_MODULE,
      .poll = binder_poll,
      .unlocked_ioctl = binder_ioctl,
      .mmap = binder_mmap,
      .open = binder_open,
      .flush = binder_flush,
      .release = binder_release,
    };

    static struct miscdevice binder_miscdev = {
      .minor = MISC_DYNAMIC_MINOR,
      .name = "binder",
      .fops = &binder_fops
    };

    static int __init binder_init(void)
    {
        ...
        ret = misc_register(&binder_miscdev);
        ...
    }

    device_initcall(binder_init);

说明：上面是Kernel中Binder驱动代码，定义在drivers/staging/android/binder.c中。  
(01) device_initcall(binder_init)的作用是将binder_init()函数注册到Kernel的初始化函数列表中。当Kernel启动后，会按照一定的次序调用初始化函数列表，也就会执行binder_init()函数；执行binder_init()时便会加载Binder驱动。  
(02) binder_init()函数中会通过misc_register(&binder_miscdev)将Binder驱动注册到文件节点"/dev/binder"上。在Linux中，一切都是文件！将Binder驱动注册到文件节点上之后，就可以通过操作文件节点进而对Binder驱动进行操作。而该文件节点"/dev/binder"的设备信息是binder_miscdev这个结构体对象。  
(03) binder_miscdev变量是struct miscdevice类型。minor是次设备号，这个我们不需要关心；name是Binder驱动对应在/dev虚拟文件系统下的设备节点名称，也就是/dev/binder中的"binder"；fops是该设备节点的文件操作对象，它是我们需要重点关注的！fops指向binder_fops变量。  
(04) binder_fops变量是struct file_operations类型。owner是标明了该文件操作变量的拥有者，就是该驱动；poll则指定了poll函数指针，当我们对/dev/binder文件节点执行poll()操作时，实际上就是调用的binder_poll()函数；同理，mmap()对应binder_mmap()，open()对应binder_open()，ioctl()对应binder_ioctl()...



### 3.2 Binder驱动中的binder_open()函数源码

经过上面的介绍，我们可以知道open("/dev/binder", O_RDWR)实际上是调用Binder驱动中的binder_open()函数。  

    static HLIST_HEAD(binder_procs);
    ...

    static int binder_open(struct inode *nodp, struct file *filp)
    {
        struct binder_proc *proc;

        binder_debug(BINDER_DEBUG_OPEN_CLOSE, "binder_open: %d:%d\n",
               current->group_leader->pid, current->pid);

        // 为proc分配内存
        proc = kzalloc(sizeof(*proc), GFP_KERNEL);
        if (proc == NULL)
          return -ENOMEM;
        get_task_struct(current);
        // 将proc->tsk指向当前线程
        proc->tsk = current;
        // 初始化proc的待处理事务列表
        INIT_LIST_HEAD(&proc->todo);
        // 初始化proc的等待队列
        init_waitqueue_head(&proc->wait);
        // 设置proc的进程优先级为当前线程的优先级
        proc->default_priority = task_nice(current);

        binder_lock(__func__);

        binder_stats_created(BINDER_STAT_PROC);
        // 将该进程上下文信息proc保存到"全局哈希表binder_procs"中
        hlist_add_head(&proc->proc_node, &binder_procs);
        // 设置进程id
        proc->pid = current->group_leader->pid;
        INIT_LIST_HEAD(&proc->delivered_death);
        // 将proc添加到私有数据中。
        // 这样，mmap(),ioctl()等函数都可以通过私有数据获取到proc，即该进程的上下文信息
        filp->private_data = proc;

        binder_unlock(__func__);

        if (binder_debugfs_dir_entry_proc) {
          char strbuf[11];
          snprintf(strbuf, sizeof(strbuf), "%u", proc->pid);
          proc->debugfs_entry = debugfs_create_file(strbuf, S_IRUGO,
              binder_debugfs_dir_entry_proc, proc, &binder_proc_fops);
        }

        return 0;
    }

说明：binder_proc是记录进程上下文信息的结构体，它的详细介绍请参考[Android Binder机制(二) Binder中的数据结构][link_binder_02_datastruct]。该函数的作用如下。  
(01) 创建并初始化binder_proc结构体变量proc。binder_proc是描述Binder进程的上下文信息结构体。这里，就是将ServiceManager这个进程的信息都存储到proc中。   
(02) 将proc添加到全局哈希表binder_procs中。binder_procs不是我们关注的重点，也就不多说了。  
(03) 将proc设为filp的私有成员。这样，在mmap()，ioctl()等函数中，我们都可以根据filp的私有成员来获取proc信息。  



<a name="anchor4"></a>
## 4. mmap()

分析完了open()，接下来看看mmap()。mmap()对应会调用Binder驱动的binder_mmap()函数。

### 4.1 Binder驱动中的binder_mmap()源码

    static int binder_mmap(struct file *filp, struct vm_area_struct *vma)
    {
      int ret;
      struct vm_struct *area;
      struct binder_proc *proc = filp->private_data;
      const char *failure_string;
      struct binder_buffer *buffer;

      // 有效性检查：映射的内存不能大于4M
      if ((vma->vm_end - vma->vm_start) > SZ_4M)
          vma->vm_end = vma->vm_start + SZ_4M;

      ...

      vma->vm_flags = (vma->vm_flags | VM_DONTCOPY) & ~VM_MAYWRITE;

      mutex_lock(&binder_mmap_lock);

      // 获取空闲的内核空间地址
      area = get_vm_area(vma->vm_end - vma->vm_start, VM_IOREMAP);
      ...

      // 将内核空间地址赋值给proc->buffer，即保存到进程上下文中
      proc->buffer = area->addr;
      // 计算 "内核空间地址" 和 "进程虚拟地址" 的偏移
      proc->user_buffer_offset = vma->vm_start - (uintptr_t)proc->buffer;
      mutex_unlock(&binder_mmap_lock);

      // 为proc->pages分配内存
      proc->pages = kzalloc(sizeof(proc->pages[0]) * ((vma->vm_end - vma->vm_start) / PAGE_SIZE), GFP_KERNEL);
      ...

      // 内核空间的内存大小 = 进程虚拟地址区域(用户空间)的内存大小
      proc->buffer_size = vma->vm_end - vma->vm_start;

      vma->vm_ops = &binder_vm_ops;
      // 将 proc(进程上下文信息) 赋值给vma私有数据
      vma->vm_private_data = proc;

      // 通过调用binder_update_page_range()来分配物理页面。
      // 即，将物理内存映射到内核空间 以及 用户空间
      if (binder_update_page_range(proc, 1, proc->buffer, proc->buffer + PAGE_SIZE, vma)) {
          goto err_alloc_small_buf_failed;
      }
      buffer = proc->buffer;
      INIT_LIST_HEAD(&proc->buffers);
      // 将物理内存添加到proc->buffers链表中进行管理。
      list_add(&buffer->entry, &proc->buffers);
      buffer->free = 1;
      binder_insert_free_buffer(proc, buffer);
      proc->free_async_space = proc->buffer_size / 2;
      barrier();
      proc->files = get_files_struct(proc->tsk);
      // 将用户空间地址信息保存到proc中
      proc->vma = vma;
      proc->vma_vm_mm = vma->vm_mm;

      return 0;
      ...
    }

说明：mmap的作用是进行内存映射。当应用调用mmap()映射内存到进程虚拟地址时，该函数会进行两个操作：第一，将指定大小的"物理内存" 映射到 "用户空间"(即，进程的虚拟地址中)。 第二，将该"物理内存" 也映射到 "内核空间(即，内核的虚拟地址中)"。  
  简单来说，就是"将进程虚拟地址空间和内核虚拟地址空间映射同一个物理页面"。为什么要这么做呢？这就是Binder进程间通信机制的精髓所在了！在讲解之前，先回顾一下进程间通信的基础知识。

> 在32位Linux系统的内存地址划分中，0~3G为用户空间，3~4G为内核空间。应用程序都运行在用户空间，而kernel和驱动都运行在内核空间。应用程序之间若涉及到数据交换(例如，Client进程向Server进程发送请求)，即进程间通信，需要使用管道/消息队列/Socket/共享内存等IPC机制。共享内存控制比较复杂，而Socket常用于网络通信，这里将它们排除；剩下的就是管道/消息队列。下面对管道/消息队列的IPC等通信方式进行介绍。假如现在采用管道/消息队列从Client向Server发送请求，需要先将Client进程的数据拷贝到内核空间，然后再从内核空间拷贝到Server进程中。这其中，总共涉及到了2次内存拷贝！  
> 而Binder机制则只需要进行1次内存拷贝即可！

在Binder通信机制中，mmap()会将Server进程的虚拟地址和内核虚拟地址映射到同一个物理页面。那么当Client进程向Server进程发送请求时，只需要将Client的数据拷贝到内核空间即可！由于Server进程的地址和内核空间映射到同一个物理页面，因此，Client中的数据拷贝到内核空间时，也就相当于拷贝到了Server进程中。因此，Binder通信机制中，数据传输时，只需要1次内存拷贝！

<br/>
有了上面的理论基础，再来看mmap()是如何实现的。  
(01) proc = flip->private_data。该flip的私有数据是在binder_open()中设置的，这里通过该私有数据就获取binder_proc变量proc。  
(02) area = get_vm_area(vma->vm_end - vma->vm_start, VM_IOREMAP)。 它的作用是从内核虚拟地址中，获取指定大小的空闲地址，将空闲地址的起始地址赋值给area。 area是vm_struct类型，vm_struct是描述内核虚拟地址信息的结构体。此外，vm_area_struct则是描述进程虚拟地址信息的结构体。  
(03) 接着，给proc->buffer(内核空间地址)，proc->user_buffer_offset(内核空间地址和进程虚拟地址的偏移值)，proc->pages(内核空间所占物理页面的数目)，proc->buffer_size(内核地址空间的大小)赋值。  
(04) 然后，调用binder_update_page_range(proc, 1, proc->buffer, proc->buffer + PAGE_SIZE, vma)。它作用是分配物理内存，下面看看它的实现。  


### 4.2 Binder驱动中的binder_update_page_range()源码

    static int binder_update_page_range(struct binder_proc *proc, int allocate,
                      void *start, void *end,
                      struct vm_area_struct *vma)
    {
      void *page_addr;
      unsigned long user_page_addr;
      struct vm_struct tmp_area;
      struct page **page;

      ...

      // 分配物理页面，
      // 并将"内核空间"和"用户空间(进程的内存区域)"指向同一块物理内存。
      for (page_addr = start; page_addr < end; page_addr += PAGE_SIZE) {
          int ret;
          struct page **page_array_ptr;
          page = &proc->pages[(page_addr - proc->buffer) / PAGE_SIZE];

          // 分配物理页面
          *page = alloc_page(GFP_KERNEL | __GFP_ZERO);
          ...

          tmp_area.addr = page_addr;
          tmp_area.size = PAGE_SIZE + PAGE_SIZE /* guard page? */;
          page_array_ptr = page;
          // 将物理页面映射到内核空间中
          ret = map_vm_area(&tmp_area, PAGE_KERNEL, &page_array_ptr);
          ...

          user_page_addr =
              (uintptr_t)page_addr + proc->user_buffer_offset;
          // 将物理页面映射插入到进程的虚拟内存中
          ret = vm_insert_page(vma, user_page_addr, page[0]);
          ...
      }

      return 0;

      ...
    }

说明： binder_update_page_range()既可分配物理页面，也可以释放物理页面。当参数allocate=1时，会执行分配物理页面的操作；否则，会执行释放物理页面的操作。这里，allocate=1；因此，我们只关心分配物理页面的部分。  
在for循环中，每分配一个物理页面都会先通过map_vm_area()将该物理内存映射到内核虚拟地址中；然后再将该物理页面插入到进程的虚拟地址空间。


<br/>至此，binder_open(128*1024)算是介绍完了。从"用户空间的ServiceManager进程" 和 "Binder驱动"这两个方面分析它的作用。  
(01) **ServiceManager进程**：就是打开/dev/binder，同时映射物理内存到进程空间。  
(02) **Binder驱动**：新建并初始化该进程对应的binder_proc结构体，同时将内核虚拟地址和该进程的虚拟地址映射到同一物理内存中。



<a name="anchor5"></a>
## 5. binder_become_context_manager()

下面接着分析binder_become_context_manager(bs)。

    int binder_become_context_manager(struct binder_state *bs)
    {
        return ioctl(bs->fd, BINDER_SET_CONTEXT_MGR, 0);
    }

说明：根据前面介绍的Binder驱动初始化信息可知，ioctl()就是调用Binder驱动中的binder_ioctl()函数。 



<a name="anchor6"></a>
## 6. ioctl(, BINDER_SET_CONTEXT_MGR,)

### 6.1 Binder驱动中binder_ioctl()的BINDER_WRITE_READ相关部分的源码

    // 全局binder实体，准确点说是ServiceManager的binder实体
    static struct binder_node *binder_context_mgr_node;
    // ServiceManager守护进程的uid
    static uid_t binder_context_mgr_uid = -1;
    static int binder_stop_on_user_error;
    ...

    static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
    {
      int ret;
      struct binder_proc *proc = filp->private_data;
      struct binder_thread *thread;
      unsigned int size = _IOC_SIZE(cmd);
      void __user *ubuf = (void __user *)arg;

      // 中断等待函数。
      // 1. 当binder_stop_on_user_error < 2为true时；不会进入等待状态；直接跳过。
      // 2. 当binder_stop_on_user_error < 2为false时，进入等待状态。
      //    当有其他进程通过wake_up_interruptible来唤醒binder_user_error_wait队列，并且binder_stop_on_user_error < 2为true时；
      //    则继续执行；否则，再进入等待状态。
      ret = wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);
      ...

      binder_lock(__func__);
      // 在proc进程中查找该线程对应的binder_thread；若查找失败，则新建一个binder_thread，并添加到proc->threads中。
      thread = binder_get_thread(proc);
      ...

      switch (cmd) {
      ...

      case BINDER_SET_CONTEXT_MGR:
          if (binder_context_mgr_node != NULL) {
              ...
          }
          if (binder_context_mgr_uid != -1) {
              ...
          } else
              // 设置ServiceManager对应的uid
              binder_context_mgr_uid = current->cred->euid;

          // 新建binder实体，并将proc进程上下文信息保存到binder实体中；
          // 然后，将该binder实体赋值给全局变量binder_context_mgr_node。
          // 这个全局的binder实体，是ServiceManager对应的binder实体。
          binder_context_mgr_node = binder_new_node(proc, NULL, NULL);
          ...

          // 设置binder实体的引用计数等参数
          binder_context_mgr_node->local_weak_refs++;
          binder_context_mgr_node->local_strong_refs++;
          binder_context_mgr_node->has_strong_ref = 1;
          binder_context_mgr_node->has_weak_ref = 1;
          break;
      ...

      }
      ret = 0;

    err:
      // 去掉thread的BINDER_LOOPER_STATE_NEED_RETURN标记
      if (thread)
          thread->looper &= ~BINDER_LOOPER_STATE_NEED_RETURN;
      binder_unlock(__func__);
      wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);
      ...

      return ret;
    }


说明：binder_ioctl()的内容很多，上面仅仅列出与BINDER_SET_CONTEXT_MGR相关的代码。   
(01) proc = flip->private_data。该flip的私有数据是在binder_open()中设置的，这里通过该私有数据就获取binder_proc变量proc。  
(02) 接着调用wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2)。由于binder_stop_on_user_error是全局变量，它的初始值是0，因此binder_stop_on_user_error < 2为true，不进入中断等待，而是直接跳过该函数继续运行。  
(03) binder_get_thread()会在proc中查找当前线程对应的binder_thread结构体；由于之前还未创建该线程的binder_thread结构体，因此查找失败。进而创建一个binder_thread结构体变量，并将其添加到proc->threads红黑树中，然后返回该变量。  
(04) cmd的值是我们调用ioctl()传入的参数BINDER_SET_CONTEXT_MGR。在BINDER_SET_CONTEXT_MGR分支中，会设置binder_context_mgr_uid，binder_context_mgr_uid是一个全局变量，它代表ServiceManager对应的uid；接着，通过binder_new_node()新建一个Binder实体(即binder_node结构体对象)，并将该Binder实体赋值给全局变量binder_context_mgr_node，binder_context_mgr_node就是Serveice Manager对应的Binder实体；最后，设置binder实体的引用计数等参数。  
(05) 清除thread->looper的BINDER_LOOPER_STATE_NEED_RETURN标记。这个BINDER_LOOPER_STATE_NEED_RETURN标记，是在调用binder_get_thread()中创建binder_thread对象时添加的。  

关于binder_node结构体，在[Android Binder机制(二) Binder中的数据结构][link_binder_02_datastruct]中有消息的介绍。特别需要了解的是，对于每一个Server，Binder驱动都会为其分配一个binder_node对象。对于ServiceManager这个Binder上下文管理者而言，Binder驱动更是会将它的Binder实体保存到全局变量中。



### 6.2 Binder驱动中的binder_get_thread()源码

下面看看binder_get_thread()中做了什么。

    static struct binder_thread *binder_get_thread(struct binder_proc *proc)
    {
      struct binder_thread *thread = NULL;
      struct rb_node *parent = NULL;
      struct rb_node **p = &proc->threads.rb_node;

      // 在proc->threads这棵红黑树中，查找是否有线程的pid和current->pid相同。
      // 即，查找当前线程中是否创建过binder_thread信息
      while (*p) {
          parent = *p;
          thread = rb_entry(parent, struct binder_thread, rb_node);

          if (current->pid < thread->pid)
              p = &(*p)->rb_left;
          else if (current->pid > thread->pid)
              p = &(*p)->rb_right;
          else
              break;
      }
      // 若当前线程中没有创建过binder_thread信息；
      // 则创建binder_thread，并初始化；然后将其添加到binder_proc进程的proc->threads中
      if (*p == NULL) {
          thread = kzalloc(sizeof(*thread), GFP_KERNEL);
          if (thread == NULL)
              return NULL;
          binder_stats_created(BINDER_STAT_THREAD);
          // 将进程的上下文信息保存到thread中
          thread->proc = proc;
          thread->pid = current->pid;
          // 初始化thread的等待队列
          init_waitqueue_head(&thread->wait);
          // 初始化thread的待处理事件列表
          INIT_LIST_HEAD(&thread->todo);
          // 将该thread链接到proc->threads这棵红黑树中
          rb_link_node(&thread->rb_node, parent, p);
          rb_insert_color(&thread->rb_node, &proc->threads);
          thread->looper |= BINDER_LOOPER_STATE_NEED_RETURN;
          thread->return_error = BR_OK;
          thread->return_error2 = BR_OK;
      }
      return thread;
    }

说明：  
(01) 理解"红黑树"和"rb_entry"是理解while循环的前提。这里简单介绍下，proc->threads这棵红黑树是根据proc->thread->pid来排序的；而rb_entry(parent, struct binder_thread, rb_node)的作用根据binder_thread结构体对象中的已知成员的地址(binder_thread->rb_node的地址，也就是parent的值)来获取binder_thread结构体对象的地址。  
(02) 很显然，由于之前没有创建过当前线程对应的binder_thread对象，所以*p==null为true。那么，接下来就新建binder_thread对象，并对其进行初始化，然后再添加到红黑树proc->threads中。



### 6.3 Binder驱动中的binder_new_node()源码

下面看看binder_ioctl()中调用的binder_new_node()的代码。

    static struct binder_node *binder_new_node(struct binder_proc *proc,
                         void __user *ptr,
                         void __user *cookie)
    {
      struct rb_node **p = &proc->nodes.rb_node;
      struct rb_node *parent = NULL;
      struct binder_node *node;

      // 在proc->nodes这棵红黑树中，查找有要查找的binder实体(通过ptr成员来判断)
      while (*p) {
          parent = *p;
          node = rb_entry(parent, struct binder_node, rb_node);

          if (ptr < node->ptr)
              p = &(*p)->rb_left;
          else if (ptr > node->ptr)
              p = &(*p)->rb_right;
          else
              return NULL;
      }

      // 如果没有要找的binder实体，则新建该binder实体
      node = kzalloc(sizeof(*node), GFP_KERNEL);
      if (node == NULL)
          return NULL;
      binder_stats_created(BINDER_STAT_NODE);
      // 将node链接到红黑树proc->nodes中
      rb_link_node(&node->rb_node, parent, p);
      rb_insert_color(&node->rb_node, &proc->nodes);
      node->debug_id = ++binder_last_id;
      // 将进程上下文信息保存到node->proc中
      node->proc = proc;
      node->ptr = ptr;
      node->cookie = cookie;
      node->work.type = BINDER_WORK_NODE;
      INIT_LIST_HEAD(&node->work.entry);
      INIT_LIST_HEAD(&node->async_todo);
      return node;
    }

说明：跟binder_get_thread()类似，这里是先在proc->nodes这棵红黑树中查找是否有binder实体(即binder_node对象)存在。有的话，返回NULL，即不需要新建binder实体；没有的话，则新建并初始化binder_node对象，然后将其添加到proc->nodes红黑树中。


<br/>至此，binder_become_context_manager()就介绍完了。它的作用：  
(01) **ServiceManager进程**：告诉Kernel驱动，当前进程(即ServiceManager进程)是Binder上下文管理者。  
(02) **Binder驱动**：新建当前线程对应的binder_thread对象，并将其添加到进程上下文信息binder_proc的threads红黑树中；新建ServiceManager对应的binder实体，并将该binder实体保存到全局变量binder_context_mgr_node中。





<a name="anchor7"></a>
## 7. binder_loop()

我们继续回到main()函数，分析一下binder_loop(bs, svcmgr_handler)。

### 7.1 binder_loop()的源码

    void binder_loop(struct binder_state *bs, binder_handler func)
    {
        int res; 
        struct binder_write_read bwr;
        unsigned readbuf[32];
        
        bwr.write_size = 0;
        bwr.write_consumed = 0;
        bwr.write_buffer = 0;
        
        // 告诉Kernel，ServiceManager进程进入了消息循环状态。
        readbuf[0] = BC_ENTER_LOOPER;
        binder_write(bs, readbuf, sizeof(unsigned));
        
        for (;;) {
            bwr.read_size = sizeof(readbuf);
            bwr.read_consumed = 0;
            bwr.read_buffer = (unsigned) readbuf;

            // 向Kernel中发送消息(先写后读)。
            // 先将消息传递给Kernel，然后再从Kernel读取消息反馈
            res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);
        
            ...
        
            // 解析读取的消息反馈
            res = binder_parse(bs, 0, readbuf, bwr.read_consumed, func);
            ...
        }
    }

说明： 该代码定义在frameworks/native/cmds/servicemanager/binder.c中。  
  binder_loop()首先调用binder_write(,BC_ENTER_LOOPER,)告诉Kernel，ServiceManager进入了消息循环状态。紧接着，就通过ioctl(,BINDER_WRITE_READ,)进入消息循环，等待Client发送请求(例如，MediaPlayer进程调用addService将MediaPlayer注册到ServiceManager中进行管理)。如果没有消息，则进入中断等待状态；如果有消息，则进行消息处理！ 


### 7.2 binder_write()的源码

下面看看binder_loop()中的binder_write(,BC_ENTER_LOOPER,)。

    int binder_write(struct binder_state *bs, void *data, unsigned len)
    {
        struct binder_write_read bwr;
        int res;
        bwr.write_size = len;                // 数据长度
        bwr.write_consumed = 0;             
        bwr.write_buffer = (unsigned) data;  // 数据是BINDER_WRITE_READ
        bwr.read_size = 0;
        bwr.read_consumed = 0;
        bwr.read_buffer = 0;
        res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);
        if (res < 0) {
            fprintf(stderr,"binder_write: ioctl failed (%s)\n",
                    strerror(errno));
        }
        return res;
    }

说明：binder_write()单单只是向Kernel发送一个消息，而不会去读取消息反馈。这里的ioctl()又会调用到binder_ioctl()。  
这里涉及到了Binder通信中常用的数据结构体binder_write_read。bwr.write_size>0，表示通过ServiceManager有数据(即BC_ENTER_LOOPER指令)发送给Binder驱动，而发送的数据就保存在bwr.write_buffer中，bwr.write_consumed则表示已经被读取并处理的数据的大小。bwr.read_XXX则是用来保存Binder驱动即将反馈给ServiceManager的信息的。   
更多关于binder_write_read的介绍，请参考[Android Binder机制(二) Binder中的数据结构][link_binder_02_datastruct]。


### 7.3 Binder驱动中binder_ioctl()的BINDER_WRITE_READ相关部分的源码

下面我们看看Binder驱动部分的对应代码。


    static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
    {
      int ret;
      struct binder_proc *proc = filp->private_data;
      struct binder_thread *thread;
      unsigned int size = _IOC_SIZE(cmd);
      void __user *ubuf = (void __user *)arg;

      // 中断等待函数。
      // 1. 当binder_stop_on_user_error < 2为true时；不会进入等待状态；直接跳过。
      // 2. 当binder_stop_on_user_error < 2为false时，进入等待状态。
      //    当有其他进程通过wake_up_interruptible来唤醒binder_user_error_wait队列，并且binder_stop_on_user_error < 2为true时；
      //    则继续执行；否则，再进入等待状态。
      ret = wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);

      binder_lock(__func__);
      // 在proc进程中查找该线程对应的binder_thread；若查找失败，则新建一个binder_thread，并添加到proc->threads中。
      thread = binder_get_thread(proc);
      ...

      switch (cmd) {
      case BINDER_WRITE_READ: {
          struct binder_write_read bwr;
          ...

          // 将binder_write_read从"用户空间" 拷贝到 "内核空间"
          if (copy_from_user(&bwr, ubuf, sizeof(bwr))) {
              ...
          }

          // 如果write_size>0，则进行写操作
          if (bwr.write_size > 0) {
              ret = binder_thread_write(proc, thread, (void __user *)bwr.write_buffer, bwr.write_size, &bwr.write_consumed);
              ...
          }

          // 如果read_size>0，则进行读操作
          if (bwr.read_size > 0) {
              ret = binder_thread_read(proc, thread, (void __user *)bwr.read_buffer, bwr.read_size, &bwr.read_consumed, filp->f_flags   & O_NONBLOCK);
              ...
          }
          ...

          if (copy_to_user(ubuf, &bwr, sizeof(bwr))) {
              ret = -EFAULT;
              goto err;
          }
          break;
      }
      ...
      }
      ret = 0;

      ...
      return ret;
    }

说明：  
(01) wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2)中binder_stop_on_user_error < 2为true。因此，不进入中断等待状态而是直接跳过该函数。  
(02) thread = binder_get_thread(proc)。由于在上一次调用ioctl时，已经创建了该线程对应的binder_thread对象。因此，这次能在proc->threads红黑树中找到对应的binder_thread对象，然后，返回给thread。  
(03) copy_from_user()的作用是将用户空间的数据拷贝到内核空间。即，将ServiceManager中调用ioctl(bs->fd, BINDER_WRITE_READ, &bwr)时的bwr对象拷贝到Binder驱动中。  
(04) 在binder_write()中，设置的bwr.write_size>0；所以，调用binder_thread_write()进行写操作。  
(05) 在binder_write()中，设置的bwr.read_size为0；所以，不调用binder_thread_read()进行读操作。  
(06) 读写操作完毕之后，将bwr从内核空间再拷贝到用户空间。  


### 7.4 Binder驱动中binder_thread_write()的源码

    int binder_thread_write(struct binder_proc *proc, struct binder_thread *thread,
              void __user *buffer, int size, signed long *consumed)
    {
      uint32_t cmd;
      void __user *ptr = buffer + *consumed;
      void __user *end = buffer + size;

      // 读取binder_write_read.write_buffer中的内容。
      // 每次读取32bit(即4个字节)
      while (ptr < end && thread->return_error == BR_OK) {
          // 从用户空间读取32bit到内核中，并赋值给cmd。
          if (get_user(cmd, (uint32_t __user *)ptr))
              return -EFAULT;

          ptr += sizeof(uint32_t);
          ...
          switch (cmd) {
          ...
          case BC_ENTER_LOOPER:
              ...
              // 设置线程的状态为BINDER_LOOPER_STATE_ENTERED；
              // 即，进入了循环状态
              thread->looper |= BINDER_LOOPER_STATE_ENTERED;
              break;
          ...
          }
          // 更新bwr.write_consumed的值
          *consumed = ptr - buffer;
      }
      return 0;
    }

说明：binder_thread_write()从brw.write_buffer中读取4个字节作为cmd。这4个字节就是ServiceManager传递的指令BC_ENTER_LOOPER。  
在BC_ENTER_LOOPER对应的switch分支中，就是将BINDER_LOOPER_STATE_ENTERED加入到thread->looper中。即，告诉Binder驱动，ServiceManager进程进入了消息循环状态。



<a name="anchor8"></a>
## 8. for(;;)

继续往下走。回到binder_loop()中后，便进入了for(;;)消息循环中。进入循环后，首先调用ioctl(,BINDER_WRITE_READ,)；此时，对应的bwr内容如下：

        bwr.write_size = 0;
        bwr.write_consumed = 0;
        bwr.write_buffer = 0;
        bwr.read_size = sizeof(readbuf);
        bwr.read_consumed = 0;
        bwr.read_buffer = (unsigned) readbuf;

bwr.write_size=0，而bwr.read_size>0；表示只会从Binder驱动读取数据，而并不会向Binder驱动中写入数据。接着，调用ioctl()便再次进入到Binder驱动binder_ioctl()中。


### 8.1 Binder驱动中binder_ioctl()的BINDER_WRITE_READ相关部分的源码

    static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
    {
      ...
      switch (cmd) {
      case BINDER_WRITE_READ: {
          struct binder_write_read bwr;
          ...

          // 将binder_write_read从"用户空间" 拷贝到 "内核空间"
          if (copy_from_user(&bwr, ubuf, sizeof(bwr))) {
              ...
          }

          // 如果write_size>0，则进行写操作
          if (bwr.write_size > 0) {
              ret = binder_thread_write(proc, thread, (void __user *)bwr.write_buffer, bwr.write_size, &bwr.write_consumed);
              ...
          }

          // 如果read_size>0，则进行读操作
          if (bwr.read_size > 0) {
              ret = binder_thread_read(proc, thread, (void __user *)bwr.read_buffer, bwr.read_size, &bwr.read_consumed, filp->f_flags   & O_NONBLOCK);
              ...
          }
          ...

          if (copy_to_user(ubuf, &bwr, sizeof(bwr))) {
              ret = -EFAULT;
              goto err;
          }
          break;
      }
      ...
      }
      ret = 0;

      ...
      return ret;
    }

说明：由于此次bwr.write_size=0，而bwr.read_size不为0。因此，在通过copy_from_user()将数据从用户空间拷贝到内核空间之后，不进行写操作，而只进行读操作，即只执行binder_thread_read()。 在读操作执行完毕之后，再通过copy_to_user()，将数据返回给用户空间。  



### 8.2 Binder驱动中binder_thread_read()的源码

    static int binder_thread_read(struct binder_proc *proc,
                    struct binder_thread *thread,
                    void  __user *buffer, int size,
                    signed long *consumed, int non_block)
    {
      void __user *ptr = buffer + *consumed;
      void __user *end = buffer + size;

      int ret = 0;
      int wait_for_proc_work;

      // 如果*consumed=0，则写入BR_NOOP到用户传进来的bwr.read_buffer缓存区
      if (*consumed == 0) {
          if (put_user(BR_NOOP, (uint32_t __user *)ptr))
              return -EFAULT;
          // 修改指针位置
          ptr += sizeof(uint32_t);
      }

    retry:
      // 等待proc进程的事务标记。
      // 当线程的事务栈为空 并且 待处理事务列表为空时，该标记位true。
      wait_for_proc_work = thread->transaction_stack == NULL &&
                  list_empty(&thread->todo);

      ...

      // 设置线程为"等待状态"
      thread->looper |= BINDER_LOOPER_STATE_WAITING;
      if (wait_for_proc_work)
          proc->ready_threads++;

      ...
      if (wait_for_proc_work) {
          ...
          // 设置当前线程的优先级=proc->default_priority。
          // 即，当前线程要处理proc的事务，所以设置优先级和proc一样。
          binder_set_nice(proc->default_priority);
          if (non_block) {
              // 非阻塞式的读取，则通过binder_has_proc_work()读取proc的事务；
              // 若没有，则直接返回
              if (!binder_has_proc_work(proc, thread))
                  ret = -EAGAIN;
          } else
              // 阻塞式的读取，则阻塞等待事务的发生。
              ret = wait_event_interruptible_exclusive(proc->wait, binder_has_proc_work(proc, thread));
      } else {
          ...
      }
      ...
    }
  
说明：  
(01) 很显然，bwr.read_consumed=0。因此，*consumed=0，那么就将BR_NOOP拷贝到用户空间的bwr.read_buffer缓存区中。    
(02) 目前为止，并没有进程将事务添加到当前线程中；因此，线程的事务栈和待处理事务队列都是为空。于是得到wait_for_proc_work的值是true。  
(03) binder_set_nice()的作用是设置当前线程的优先级=proc->default_priority。  
(04) 根据上下文，可知non_block为false。因此调用wait_event_interruptible_exclusive(proc->wait, binder_has_proc_work(proc, thread))。 而目前ServiceManager进程中没有待处理事务，因此binder_has_proc_work(proc, thread)为false。从而当前线程进入中断等待状态，等待其它进程将ServiceManager唤醒。 


<br/>至此，ServiceManager进入了等待状态，binder_loop()就分析就暂告一段落。  
(01) **ServiceManager进程**：binder_loop()通过BC_ENTER_LOOPER告诉Kernel，ServiceManager进入了消息循环状态。接着，ServiceManager就进入等待状态，等待Client请求。  
(02) **Binder驱动**：已知ServiceManager进入了消息循环状态；在收到ServiceManager的BINDER_WRITE_READ消息之后，就去ServiceManager的从进程上下文binder_proc对象中读取是否有待处理事务，由于没有事务处理，则将ServiceManager线程设为中断等待状态。  



<a name="anchor_3rd"></a>
# ServiceManager流程总结

总结上面的分析，ServiceManager的main()进程完成了以下工作。 

1. 对于**ServiceManager进程**而言  
它打开了Binder设备文件，并且将内存映射到ServiceManager的进程空间。然后，它告诉Binder驱动自己是Binder上下文的管理者。最后，进入消息循环，等待Client请求。

2. 对于**Binder驱动**而言  
初始化了ServiceManager对应的进程上下文环境(即binder_proc变量)，并将内核虚拟地址和进程虚拟地址映射到同一物理内存中。然后，新建当前线程对应的binder_thread对象，并将其添加到进程上下文信息binder_proc->threads红黑树中。在得知ServiceManager是Binder上下文管理者后，建立ServiceManager对应的Binder实体，并将该Binder实体保存到全局变量中。最后，得知ServiceManager进入消息循环后，由于当前线程中没有事务可处理，则进入中断等待状态，等待其他进程将其唤醒。



[link_binder_02_datastruct]: /2014/09/02/Binder-Datastruct/

