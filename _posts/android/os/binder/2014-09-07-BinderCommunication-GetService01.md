---
layout: post
title: "Android Binder机制(九) getService详解01之 请求的发送"
description: "android"
category: android
tags: [android]
date: 2014-09-07 09:01
---


> 前面，以MediaPlayerService为例，介绍了Server服务是如何通过addService请求添加到ServiceManager中的。本文，将以MediaPlayer获取MediaPlayerService服务为例，介绍Client是如何通过getService请求从ServiceManager中获取到Server接入点的。  
> 在本文的getService请求中，MediaPlayer是Client，它要获取的Server接入点是MediaPlayerService。和addService一样，在分析getService时，会将文章分为请求的发送，请求的处理，和请求的反馈这3部分来进行介绍。

> **目录**  
> **1**. [Android消息机制的架构](#anchor1)  

> 注意：本文是基于Android 4.4.2版本进行介绍的！


<a name="anchor1"></a>
# getService概述

先看看getService的时序图

<a href="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/binder/getService.jpg"><img src="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/binder/getService.jpg" alt="" /></a>

 仔细观察，你会发现getService的时序图和"addService的时序图(在[Android Binder机制(五) addService详解01之 请求的发送][link_binder_05_addService01]中)"几乎是一样的。  
 先是MediaPlayer进程将getService以BC_TRANSACTION事务的方式发给Binder驱动。Binder驱动收到之后，对内容进行解析；然后唤醒ServiceManager，同时反馈一个BR_TRANSACTION_COMPLETE给MediaPlayer。反馈的BR_TRANSACTION_COMPLETE是告诉MediaPlayer，它的getService请求已经被Binder驱动成功收到。接着，MediaPlayer就进入等待状态，等待ServiceManager的反馈。 ServiceManager被唤醒之后，读取Binder驱动传递给它的BR_TRANSACTION事务。在得知是获取MediaPlayerService的请求之后，就从缓冲中取出MediaPlayerService的相关信息；然后和BC_REPLY指令一起反馈给Binder驱动。Binder驱动收到ServiceManager的反馈之后，将内容进一步反馈给MediaPlayer，并将MediaPlayer唤醒。MediaPlayer被唤醒之后，从Binder驱动反馈的BR_REPLY中解析出MediaPlayerService的相关信息；这样，MediaPlayer就成功获取到了MediaPlayerService的接入点。



<a name="anchor2"></a>
# getService的代码解析

<a name="anchor2_1"></a>
## 1. MediaPlayer的getService入口

    sp<IMediaPlayerService> IMediaDeathNotifier::sMediaPlayerService;
    ...

    const sp<IMediaPlayerService>& IMediaDeathNotifier::getMediaPlayerService()
    {
        ...
        if (sMediaPlayerService == 0) {
            sp<IServiceManager> sm = defaultServiceManager();
            sp<IBinder> binder;             
            do {
                binder = sm->getService(String16("media.player"));
                ...
                usleep(500000); // 0.5 s    
            } while (true);                 

            ...
            sMediaPlayerService = interface_cast<IMediaPlayerService>(binder);                 
        }     
        ...
        return sMediaPlayerService;         
    }

说明：该代码在frameworks/av/media/libmedia/IMediaDeathNotifier.cpp中。  
(01) sMediaPlayerService是sp<IMediaPlayerService>成员，初始化为null。因此if(sMediaPlayerService==0)为true。  
(02) 调用defaultServiceManager()获取IServiceManager对象，该对象实际上是BpServiceManager类的实例。defaultServiceManager()的详细流程请参考[Android Binder机制(四) defaultServiceManager()的实现][link_binder_04_defaultServiceManager]。  
(03) 接着就是调用sm->getService(String16("media.player"))获取MediaPlayerService对象。


<a name="anchor2_2"></a>
## 2. BpServiceManager::getService()

    virtual sp<IBinder> getService(const String16& name) const
    {
        unsigned n;
        for (n = 0; n < 5; n++){
            sp<IBinder> svc = checkService(name);
            if (svc != NULL) return svc;
            sleep(1);
        }
        return NULL;
    }

    virtual sp<IBinder> checkService( const String16& name) const
    {
        Parcel data, reply;             
        data.writeInterfaceToken(IServiceManager::getInterfaceDescriptor());               
        data.writeString16(name);       
        remote()->transact(CHECK_SERVICE_TRANSACTION, data, &reply);
        return reply.readStrongBinder();
    }     

说明：该代码在frameworks/native/libs/binder/IServiceManager.cpp中。  
(01) getService()是通过调用checkService()来获取IBinder对象的。如果获取失败，它会调用sleep()休眠1ms之后再次尝试；若尝试5次都失败，则返回null。之所以要尝试5次，是由于可能此时MediaPlayerService服务还没有准备好。  
(02) 下面看看checkService()，它和"[Android Binder机制(五) addService详解01之 请求的发送][link_binder_05_addService01]中的addService()"很多内容都相似。 checkService()会先调用writeInterfaceToken()写入一个消息头："4字节的整型数" + "字符串android.os.IServiceManager"。然后，再调用writeString16(name)将服务名"media.player"写入到data中。 最后，调用remote()->transact()进行事务交互，其中remote()返回的是BpBinder对象。




<a name="anchor2_3"></a>
## 3. BpBinder::transact()

    status_t BpBinder::transact(            
        uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
    {
        // mAlive的初始值为1
        if (mAlive) {
            status_t status = IPCThreadState::self()->transact(
                mHandle, code, data, reply, flags);
            if (status == DEAD_OBJECT) mAlive = 0;
            return status;
        }

        return DEAD_OBJECT;
    }

说明：该代码在frameworks/native/libs/binder/BpBinder.cpp中。它会调用IPCThreadState::transact()。



<a name="anchor2_4"></a>
## 4. IPCThreadState::transact()

    status_t IPCThreadState::transact(int32_t handle,
                                      uint32_t code, const Parcel& data,
                                      Parcel* reply, uint32_t flags)
    {
        status_t err = data.errorCheck();
        
        flags |= TF_ACCEPT_FDS;

        ...
        if (err == NO_ERROR) {
            ...
            err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, NULL);
        }
        
        ...
        
        if ((flags & TF_ONE_WAY) == 0) {
            if (reply) {
                err = waitForResponse(reply);
            } else {
                ...
            }
        } else {
            ...
        }

        return err;
    }

说明：该代码在frameworks/native/libs/binder/IPCThreadState.cpp中。它会先通过writeTransactionData()将要发送的指令和数据打包到binder_transaction_data中，然后调用waitForResponse()和Binder驱动进行通信。



<a name="anchor2_5"></a>
## 5. IPCThreadState::writeTransactionData()

    status_t IPCThreadState::writeTransactionData(int32_t cmd, uint32_t binderFlags,
        int32_t handle, uint32_t code, const Parcel& data, status_t* statusBuffer)
    {
        binder_transaction_data tr;

        tr.target.handle = handle;
        tr.code = code;
        tr.flags = binderFlags;
        tr.cookie = 0;
        tr.sender_pid = 0;
        tr.sender_euid = 0;
                
        const status_t err = data.errorCheck();
        if (err == NO_ERROR) {
            tr.data_size = data.ipcDataSize();
            tr.data.ptr.buffer = data.ipcData();
            tr.offsets_size = data.ipcObjectsCount()*sizeof(size_t);
            tr.data.ptr.offsets = data.ipcObjects();
        } else if (statusBuffer) {
            ..
        } else {
            ...
        }

        mOut.writeInt32(cmd);
        mOut.write(&tr, sizeof(tr));

        return NO_ERROR;
    }

说明：该函数会读取Parcel中的数据，然后将其打包到tr中，tr是binder_transaction_data结构体的对象。之后，将"指令"+"数据"写入到mOut中。指令(cmd)=BC_TRANSACTION，数据就是tr。



<a name="anchor2_6"></a>
## 6. IPCThreadState::waitForResponse()

    status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
    {
        int32_t cmd;
        int32_t err;

        while (1) {
            // 先通过talkWithDriver()和Binder驱动交互
            if ((err=talkWithDriver()) < NO_ERROR) break;
            ...
            if (mIn.dataAvail() == 0) continue;
            
            // 然后读取返回结果，再根据结果进行处理
            cmd = mIn.readInt32();
            
            switch (cmd) {
            case BR_TRANSACTION_COMPLETE:
                ...
            case BR_DEAD_REPLY:
                ...
            case BR_FAILED_REPLY:
                ...
            case BR_ACQUIRE_RESULT:
                ...
            case BR_REPLY:
                ...
            default:
                err = executeCommand(cmd);
                if (err != NO_ERROR) goto finish;
                break;
            }
        }

    finish:
        ...

        return err;
    }

说明：waitForResponse()会先调用talkWithDriver()和Binder驱动交互，然后根据反馈结果来进行处理。



<a name="anchor2_7"></a>
## 7. IPCThreadState::talkWithDriver()

    status_t IPCThreadState::talkWithDriver(bool doReceive)
    {
        ...
        binder_write_read bwr;

        // Is the read buffer empty?
        const bool needRead = mIn.dataPosition() >= mIn.dataSize();
        const size_t outAvail = (!doReceive || needRead) ? mOut.dataSize() : 0;

        bwr.write_size = outAvail;
        bwr.write_buffer = (long unsigned int)mOut.data();

        // This is what we'll read.
        if (doReceive && needRead) {
            bwr.read_size = mIn.dataCapacity();
            bwr.read_buffer = (long unsigned int)mIn.data();
        } else {
            bwr.read_size = 0;
            bwr.read_buffer = 0;
        }

        ...
        if ((bwr.write_size == 0) && (bwr.read_size == 0)) return NO_ERROR;

        bwr.write_consumed = 0;
        bwr.read_consumed = 0;
        status_t err;
        do {
            ...
            if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0)
                err = NO_ERROR;
            else
                ...
            ...
        } while (err == -EINTR);

        ...

        if (err >= NO_ERROR) {
            // 清空已写的数据
            if (bwr.write_consumed > 0) {
                if (bwr.write_consumed < (ssize_t)mOut.dataSize())
                    mOut.remove(0, bwr.write_consumed);
                else
                    mOut.setDataSize(0);
            }
            // 设置已读数据
            if (bwr.read_consumed > 0) {
                mIn.setDataSize(bwr.read_consumed);
                mIn.setDataPosition(0);
            }
            ...
            return NO_ERROR;
        }

        return err;
    }

说明：talkWithDriver()会先初始化bwr(binder_write_read类型的变量)，然后将bwr通过ioctl()发送给Binder驱动。初始化之后的bwr各个成员的值如下：  

    bwr.write_size = outAvail;                          // mOut中数据大小，大于0
    bwr.write_buffer = (long unsigned int)mOut.data();  // mOut中数据的地址
    bwr.write_consumed = 0;
    bwr.read_size = mIn.dataCapacity();                 // 256
    bwr.read_buffer = (long unsigned int)mIn.data();    // mIn.mData，实际上为空
    bwr.read_consumed = 0;

bwr初始化完成之后，调用ioctl(,BINDER_WRITE_READ,)和Binder驱动进行交互。



<a name="anchor2_8"></a>
## 8. Binder驱动中binder_ioctl()的BINDER_WRITE_READ相关部分的源码

    static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
    {
      int ret;
      struct binder_proc *proc = filp->private_data;
      struct binder_thread *thread;
      unsigned int size = _IOC_SIZE(cmd);
      void __user *ubuf = (void __user *)arg;

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

说明：首先，会将binder_write_read从用户空间拷贝到内核空间之后。拷贝之后，读取出来的bwr.write_size和bwr.read_size都>0，因此先写后读。即，先执行binder_thread_write()，然后执行binder_thread_read()。




<a name="anchor2_9"></a>
## 9. Binder驱动中binder_thread_write()的源码

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
            case BC_TRANSACTION:
            case BC_REPLY: {
                struct binder_transaction_data tr;

                if (copy_from_user(&tr, ptr, sizeof(tr)))
                    return -EFAULT;
                ptr += sizeof(tr);
                binder_transaction(proc, thread, &tr, cmd == BC_REPLY);
                break;
            }
          ...
          }
          // 更新bwr.write_consumed的值
          *consumed = ptr - buffer;
      }
      return 0;
    }

说明：MediaPlayer发送的指令是BC_TRANSACTION，这里只关心与BC_TRANSACTION相关的部分。在通过copy_from_user()将数据拷贝从用户空间拷贝到内核空间之后，就调用binder_transaction()进行处理。



<a name="anchor2_10"></a>
## 10. Binder驱动中binder_transaction()的源码

    static void binder_transaction(struct binder_proc *proc,
                       struct binder_thread *thread,
                       struct binder_transaction_data *tr, int reply)
    {
        struct binder_transaction *t;
        struct binder_work *tcomplete;
        size_t *offp, *off_end;
        struct binder_proc *target_proc;
        struct binder_thread *target_thread = NULL;
        struct binder_node *target_node = NULL;
        struct list_head *target_list;
        wait_queue_head_t *target_wait;
        struct binder_transaction *in_reply_to = NULL;
        struct binder_transaction_log_entry *e;
        uint32_t return_error;

        ...

        if (reply) {
            ...
        } else {
            if (tr->target.handle) {
                ...
            } else {
                // 该getService是从ServiceManager中获取MediaPlayer；
                // 因此事务目标对象是ServiceManager的binder实体。
                target_node = binder_context_mgr_node;
                ...
            }
            ...
            // 设置处理事务的目标进程
            target_proc = target_node->proc;
            ...
        }

        if (target_thread) {
            ...
        } else {
            target_list = &target_proc->todo;
            target_wait = &target_proc->wait;
        }
        ...

        // 分配一个待处理的事务t，t是binder事务(binder_transaction对象)
        t = kzalloc(sizeof(*t), GFP_KERNEL);
        ...

        // 分配一个待完成的工作tcomplete，tcomplete是binder_work对象。
        tcomplete = kzalloc(sizeof(*tcomplete), GFP_KERNEL);
        ...

        t->debug_id = ++binder_last_id;
        ...

        // 设置from，表示该事务是MediaPlayer线程发起的
        if (!reply && !(tr->flags & TF_ONE_WAY))
            t->from = thread;
        else
            t->from = NULL;
        // 下面的一些赋值是初始化事务t
        t->sender_euid = proc->tsk->cred->euid;
        // 事务将交给target_proc进程进行处理
        t->to_proc = target_proc;
        // 事务将交给target_thread线程进行处理
        t->to_thread = target_thread;
        // 事务编码
        t->code = tr->code;
        // 事务标志
        t->flags = tr->flags;
        // 事务优先级
        t->priority = task_nice(current);

        ...

        // 分配空间
        t->buffer = binder_alloc_buf(target_proc, tr->data_size,
            tr->offsets_size, !reply && (t->flags & TF_ONE_WAY));
        ...
        t->buffer->allow_user_free = 0;
        t->buffer->debug_id = t->debug_id;
        // 保存事务
        t->buffer->transaction = t;
        // 保存事务的目标对象(即处理该事务的binder对象)
        t->buffer->target_node = target_node;
        trace_binder_transaction_alloc_buf(t->buffer);
        if (target_node)
            binder_inc_node(target_node, 1, 0, NULL);

        offp = (size_t *)(t->buffer->data + ALIGN(tr->data_size, sizeof(void *)));

        // 将"用户空间的数据"拷贝到内核中
        // tr->data.ptr.buffer就是用户空间数据的起始地址，tr->data_size就是数据大小
        if (copy_from_user(t->buffer->data, tr->data.ptr.buffer, tr->data_size)) {
            ...
        }
        // 将"用户空间的数据中所含对象的偏移地址"拷贝到内核中
        // MediaPlayer中不包含对象, offp=null
        if (copy_from_user(offp, tr->data.ptr.offsets, tr->offsets_size)) {
            ...
        }
        ...
        // MediaPlayer中不包含对象, off_end为null
        off_end = (void *)offp + tr->offsets_size;
        // MediaPlayer中不包含对象, offp=off_end
        for (; offp < off_end; offp++) {
            ...
        }
        if (reply) {
            ..
        } else if (!(t->flags & TF_ONE_WAY)) {
            BUG_ON(t->buffer->async_transaction != 0);
            t->need_reply = 1;
            t->from_parent = thread->transaction_stack;
            // 将当前事务添加到当前线程的事务栈中
            thread->transaction_stack = t;
        } else {
            ...
        }
        // 设置事务的类型为BINDER_WORK_TRANSACTION
        t->work.type = BINDER_WORK_TRANSACTION;
        // 将事务添加到target_list队列中，即target_list的待处理事务中
        list_add_tail(&t->work.entry, target_list);
        // 设置待完成工作的类型为BINDER_WORK_TRANSACTION_COMPLETE
        tcomplete->type = BINDER_WORK_TRANSACTION_COMPLETE;
        // 将待完成工作添加到thread->todo队列中，即当前线程的待完成工作中。
        list_add_tail(&tcomplete->entry, &thread->todo);
        // 唤醒目标进程
        if (target_wait)
            wake_up_interruptible(target_wait);
        return;

        ...
    }

说明：参数reply=0，表明这是个请求事务，而不是反馈。binder_transaction新建会新建"一个待处理事务t"和"待完成的工作tcomplete"，并根据请求的数据对它们进行初始化。  
(01) MediaPlayer的getService请求是提交给ServiceManager进行处理的，因此，"待处理事务t"会被添加到ServiceManager的待处理事务队列中。此时的target_thread是ServiceManager对应的线程，而target_proc则是ServiceManager对应的进程上下文环境。  
(02) 此时，Binder驱动已经收到了MediaPlayer的getService请求；于是，将一个BINDER_WORK_TRANSACTION_COMPLETE类型的"待完成工作tcomplete"添加到当前线程(即，MediaPlayer线程)的待处理事务队列中。目的是告诉MediaPlayer，Binder驱动已经收到它的getService请求了。  
(03) 最后，调用wake_up_interruptible(target_wait)将Service Manager唤醒。

接下来，还是先分析完MediaPlayer线程，再看ServiceManager被唤醒后做了些什么。

binder_transaction()执行完毕之后，就会返回到binder_thread_write()中。binder_thread_write()更新bwr.write_consumed的值后，就返回到binder_ioctl()继续执行"读"动作。即执行binder_thread_read()。



<a name="anchor2_11"></a>
## 11. Binder驱动中binder_thread_read()的源码

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
            ptr += sizeof(uint32_t);
        }

    retry:
        // 等待proc进程的事务标记。
        // 当线程的事务栈为空 并且 待处理事务队列为空时，该标记位true。
        wait_for_proc_work = thread->transaction_stack == NULL &&
                    list_empty(&thread->todo);

        ...

        if (wait_for_proc_work) {
            ...
        } else {
            if (non_block) {
                ...
            } else
                ret = wait_event_interruptible(thread->wait, binder_has_thread_work(thread));
        }

        ...

        while (1) {
            uint32_t cmd;
            struct binder_transaction_data tr;
            struct binder_work *w;
            struct binder_transaction *t = NULL;

            // 如果当前线程的"待完成工作"不为空，则取出待完成工作。
            if (!list_empty(&thread->todo))
                w = list_first_entry(&thread->todo, struct binder_work, entry);
            else if (!list_empty(&proc->todo) && wait_for_proc_work)
                ...
            else {
                if (ptr - buffer == 4 && !(thread->looper & BINDER_LOOPER_STATE_NEED_RETURN)) /* no data added */
                    goto retry;
                break;
            }

            ...

            switch (w->type) {
            ...
            case BINDER_WORK_TRANSACTION_COMPLETE: {
                cmd = BR_TRANSACTION_COMPLETE;
                // 将BR_TRANSACTION_COMPLETE写入到用户缓冲空间中
                if (put_user(cmd, (uint32_t __user *)ptr))
                    return -EFAULT;
                ptr += sizeof(uint32_t);

                ...
                // 待完成事务已经处理完毕，将其从待完成事务队列中删除。
                list_del(&w->entry);
                kfree(w);
                binder_stats_deleted(BINDER_STAT_TRANSACTION_COMPLETE);
            } break;
            ...
            }

            if (!t)
                continue;

            ...
        }

        ...
        // 更新bwr.read_consumed的值
        *consumed = ptr - buffer;

        ...
        return 0;
    }

说明：  
(01) bwr.read_consumed=0，即if (*consumed == 0)为true。因此，会将BR_NOOP写入到bwr.read_buffer中。  
(02) thread->transaction_stack不为空，thread->todo也不为空。因为，前面在binder_transaction()中有将一个BINDER_WORK_TRANSACTION_COMPLETE类型的待完成工作添加到thread的待完成工作队列中。因此，wait_for_proc_work为false。  
(03) binder_has_thread_work(thread)为true。因此，在调用wait_event_interruptible()时，不会进入等待状态，而是继续运行。  
(04) 进入while循环后，通过list_first_entry()取出待完成工作w。w的类型w->type=BINDER_WORK_TRANSACTION_COMPLETE，进入到对应的switch分支。随后，将BR_TRANSACTION_COMPLETE写入到bwr.read_buffer中。此时，待处理工作已经完成，将其从当前线程的待处理工作队列中删除。  
(05) 最后，更新bwr.read_consumed的值。  

经过binder_thread_read()处理之后，bwr.read_buffer中包含了两个指令：BR_NOOP和BR_TRANSACTION_COMPLETE。

再回到binder_ioctl()中，在将bwr拷贝到用户空间之后，binder_ioctl()的工作就完成了。于是就返回到talkWithDriver()中。




<a name="anchor2_12"></a>
## 12. IPCThreadState::talkWithDriver()

    status_t IPCThreadState::talkWithDriver(bool doReceive)
    {
        ...
        do {
            ...
            if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0)
                err = NO_ERROR;
            else
                ...
            ...
        } while (err == -EINTR);

        ...

        if (err >= NO_ERROR) {
            // 清空已写的数据
            if (bwr.write_consumed > 0) {
                if (bwr.write_consumed < (ssize_t)mOut.dataSize())
                    ...
                else
                    mOut.setDataSize(0);
            }
            // 设置已读数据
            if (bwr.read_consumed > 0) {
                mIn.setDataSize(bwr.read_consumed);
                mIn.setDataPosition(0);
            }
            ...
            return NO_ERROR;
        }

        return err;
    }

说明：  
(01) 从Binder驱动返回后，bwr.write_consumed>0，因此调用mOut.setDataSize(0)将mOut中的数据清空。这意味着，MediaPlayer的请求Binder驱动已经收到，并且已经将请求数据读取完毕。  
(02) bwr.read_consumed也>0，因此会执行if(bwr.read_consumed>0)中的代码，更新mIn中的mDataSize和mDataPos。这意味着，Binder驱动反馈给MediaPlayer的数据不为空。接下来，MediaPlayer线程肯定会读取Binder驱动反馈的数据(BR_NOOP和BR_TRANSACTION_COMPLETE)。在读取完这些数据之后，MediaPlayer线程会再次调用ioctl(,BINDER_WRITE_READ,)进行读动作；而当执行到binder_thread_read()时，由于此时MediaPlayer线程的待处理工作队列为空，因此MediaPlayer线程会进入中断等待状态。待ServiceManager守护进程处理完MediaPlayer的请求之后，就会将MediaPlayer唤醒。

<br/>
至此，getService请求的发送部分就介绍完了。下面，就看看ServiceManager被唤醒后是如何获取MediaPlayerService进入点，然后再将该进入点反馈给MediaPlayer的。




[link_binder_01_introduce]: /2014/09/01/Binder-Introduce/
[link_binder_02_datastruct]: /2014/09/02/Binder-Datastruct/
[link_binder_03_ServiceManagerDeamon]: /2014/09/03/Binder-ServiceManager-Daemon/
[link_binder_04_defaultServiceManager]: /2014/09/04/Binder-defaultServiceManager/
[link_binder_05_addService01]: /2014/09/05/BinderCommunication-AddService01/
[link_binder_05_addService02]: /2014/09/05/BinderCommunication-AddService02/
[link_binder_05_addService03]: /2014/09/05/BinderCommunication-AddService03/
