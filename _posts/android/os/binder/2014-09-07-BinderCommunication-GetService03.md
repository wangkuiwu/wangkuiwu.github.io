---
layout: post
title: "Android Binder机制(十一) getService详解03之 请求的反馈"
description: "android"
category: android
tags: [android]
date: 2014-09-07 09:03
---


> 前面两篇文章分别介绍了getService中"请求的发送"和"请求的处理"这两部分，本文将介绍getService请求的最后一部分--请求的反馈。下面就说说MediaPlayer收到请求反馈之后的处理流程。

> 注意：本文是基于Android 4.4.2版本进行介绍的！



<a name="anchor1"></a>
# 1. Binder驱动中binder_thread_read()的源码

从MediaPlayer开始唤醒开始说起。

    static int binder_thread_read(struct binder_proc *proc,
                    struct binder_thread *thread,
                    void  __user *buffer, int size,
                    signed long *consumed, int non_block)
    {
        ...
        if (wait_for_proc_work) {
          ...
          if (non_block) {
              ...
          } else
              ret = wait_event_interruptible_exclusive(proc->wait, binder_has_proc_work(proc, thread));
        } else {
          ...
        }
        ...

        while (1) {
            struct binder_transaction_data tr;
            struct binder_work *w;
            struct binder_transaction *t = NULL;

            // 如果当前线程的"待完成工作"不为空，则取出待完成工作。
            if (!list_empty(&thread->todo))
                w = list_first_entry(&thread->todo, struct binder_work, entry);
            else if (!list_empty(&proc->todo) && wait_for_proc_work)
                ...
            else {
                ...
            }

            ...

            switch (w->type) {
                case BINDER_WORK_TRANSACTION: {
                    t = container_of(w, struct binder_transaction, work);
                } break;
                ...
            }

            if (!t)
                continue;

            // t->buffer->target_node是目标节点。
            // 这里，MediaPlayer的getService请求的目标是Service Manager，因此target_node是Service Manager对应的节点；
            if (t->buffer->target_node) {
                ...
            } else {
                tr.target.ptr = NULL;
                tr.cookie = NULL;
                cmd = BR_REPLY;
            }
            // 交易码
            tr.code = t->code;
            tr.flags = t->flags;
            tr.sender_euid = t->sender_euid;

            if (t->from) {
                struct task_struct *sender = t->from->proc->tsk;
                tr.sender_pid = task_tgid_nr_ns(sender,
                                current->nsproxy->pid_ns);
            } else {
                ...
            }

            // 数据大小
            tr.data_size = t->buffer->data_size;
            // 数据中对象的偏移数组的大小(即对象的个数)
            tr.offsets_size = t->buffer->offsets_size;
            // 数据
            tr.data.ptr.buffer = (void *)t->buffer->data +
                        proc->user_buffer_offset;
            // 数据中对象的偏移数组
            tr.data.ptr.offsets = tr.data.ptr.buffer +
                        ALIGN(t->buffer->data_size,
                            sizeof(void *));

            // 将cmd指令写入到ptr，即传递到用户空间
            if (put_user(cmd, (uint32_t __user *)ptr))
                return -EFAULT;
            // 将tr数据拷贝到用户空间
            ptr += sizeof(uint32_t);
            if (copy_to_user(ptr, &tr, sizeof(tr)))
                return -EFAULT;
            ptr += sizeof(tr);

            ...
            // 删除已处理的事务
            list_del(&t->work.entry);
            t->buffer->allow_user_free = 1;
            if (cmd == BR_TRANSACTION && !(t->flags & TF_ONE_WAY)) {
                ...
            } else {
                t->buffer->transaction = NULL;
                kfree(t);
            }
            break;
        }

    done:

        // 更新bwr.read_consumed的值
        *consumed = ptr - buffer;

        ...
        return 0;
    }


说明：MediaPlayer进程被唤醒之后，binder_has_thread_work()为true，因为MediaPlayer进程中有个BINDER_WORK_TRANSACTION类型的待处理事务。  
(01) 进入while循环后，首先取出待处理事务。  
(02) 事务的类型是BINDER_WORK_TRANSACTION，得到对应的binder_transaction*类型指针t之后，跳出switch语句。很显然，此时t不为NULL，因此继续往下执行。下面的工作的目的，是将t中的数据转移到tr中(tr是事务交互数据包结构体binder_transaction_data对应的指针)，然后将指令和tr数据都拷贝到用户空间，让MediaPlayer读取后进行处理。此时的指令为BR_REPLY！  
(03) 最后，更新*consumed的值，即更新bwr.read_consumed的值。

binder_thread_read()执行完毕之后，共反馈了两个指令到用户空间：BR_NOOP和BR_REPLY。

之后的流程应该都比较熟悉了，首先返回到binder_ioctl()中，接着将ServiceManager反馈的数据拷贝到用户空间。接下来的工作就交给MediaPlayer进程进行处理了。  
从Binder驱动返回后，首先回到talkWithDriver()中，接着便返回到waitForResponse()中。在waitForResponse()会反馈数据进行解析。  


<a name="anchor2"></a>
# 2. IPCThreadState::waitForResponse

    status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
    {
        int32_t cmd;
        int32_t err;

        while (1) {
            if ((err=talkWithDriver()) < NO_ERROR) break;
            ...

            cmd = mIn.readInt32();
            
            switch (cmd) {
                ...
            case BR_REPLY:
                {
                    binder_transaction_data tr;
                    err = mIn.read(&tr, sizeof(tr));
                    ...

                    if (reply) {
                        if ((tr.flags & TF_STATUS_CODE) == 0) {
                            reply->ipcSetDataReference(
                                reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                                tr.data_size,
                                reinterpret_cast<const size_t*>(tr.data.ptr.offsets),
                                tr.offsets_size/sizeof(size_t),
                                freeBuffer, this);
                        } else {
                            ...
                        }
                    } else {
                        ...
                    }
                }
                goto finish;
                ...
            }
        }

    finish:
        ...

        return err;
    }

说明：在BR_REPLY分支中，先读取出数据，并保存到tr中。由于reply不为null，并且tr.flags & TF_STATUS_CODE为0；因此，会执行reply->ipcSetDataReference()。


<a name="anchor3"></a>
# 3. Parcel::ipcSetDataReference

    void Parcel::ipcSetDataReference(const uint8_t* data, size_t dataSize,
        const size_t* objects, size_t objectsCount, release_func relFunc, void* relCookie)     
    {
        freeDataNoInit();                   
        mError = NO_ERROR;
        mData = const_cast<uint8_t*>(data); 
        mDataSize = mDataCapacity = dataSize;
        mDataPos = 0;
        mObjects = const_cast<size_t*>(objects);
        mObjectsSize = mObjectsCapacity = objectsCount;
        mNextObjectHint = 0;
        mOwner = relFunc;
        mOwnerCookie = relCookie;           
        scanForFds();
    }

说明： data就是ServiceManager返回来的数据。数据中包含一个flat_binder_object对象(对应ServiceManager中的binder_object)，因此objectsCount则为1。先通过freeDataNoInit()将原始的数据清空，然后再给mData和mObjects赋值，这样就将数据保存到了Parcel中。  
为什么objectsCount的值是1呢？请返回查看一下Service Manager在执行binder_send_reply()即可知，这里就不再多说。

waitForResponse()执行完BR_REPLY之后，便返回到IPCThreadState::transact()中；然后层层返回，直到退回到checkService()。  


<a name="anchor4"></a>
# 4. BpServiceManager::checkService()

    virtual sp<IBinder> checkService( const String16& name) const
    {
        Parcel data, reply;             
        data.writeInterfaceToken(IServiceManager::getInterfaceDescriptor());               
        data.writeString16(name);       
        remote()->transact(CHECK_SERVICE_TRANSACTION, data, &reply);
        return reply.readStrongBinder();
    }     

说明：到目前为止，通过transact()来获取MediaPlayerService的事务已经执行完毕！MediaPlayerService的接入点已经保存在replay中。接下来的工作就是调用reply.readStrongBinder()来从replay中解析出所需要的数据，即MediaPlayerService在Biner驱动中的Binder引用描述，也就是C++层的句柄。


<a name="anchor5"></a>
# 5. Parcel::readStrongBinder

    sp<IBinder> Parcel::readStrongBinder() const
    {
        sp<IBinder> val; 
        unflatten_binder(ProcessState::self(), *this, &val);
        return val; 
    }

说明：readStrongBinder()会调用unflatten_binder()来解析Parcel中的数据。


<a name="anchor6"></a>
# 6. Parcel::unflatten_binder

    status_t unflatten_binder(const sp<ProcessState>& proc,
        const Parcel& in, sp<IBinder>* out)
    {
        const flat_binder_object* flat = in.readObject(false);
        
        if (flat) {
            switch (flat->type) {
                case BINDER_TYPE_BINDER:
                    ...
                case BINDER_TYPE_HANDLE:
                    *out = proc->getStrongProxyForHandle(flat->handle);
                    return finish_unflatten_binder(
                        static_cast<BpBinder*>(out->get()), *flat, in);
            }        
        }
        return BAD_TYPE;
    }

说明：readObject()的作用是从Parcel中读取出它所保存的flat_binder_object类型的对象。该对象的类型是BINDER_TYPE_HANDLE，因此会指向BINDER_TYPE_HANDLE对应的switch分支。  
(01) 这里的proc是ProcessState对象，执行proc->getStrongProxyForHandle()会将句柄(MediaPlayerService的Binder引用描述)保存到ProcessState的链表中，然后再创建并返回该句柄的BpBinder对象(即Binder的代理)。在[Android Binder机制(四) defaultServiceManager()的实现][link_binder_04_defaultServiceManager]中有getStrongProxyForHandle()的详细说明，下面只给出getStrongProxyForHandle()代码。  
(02) finish_unflatten_binder()中只有return NO_ERROR。


    sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle)
    {
        sp<IBinder> result;

        AutoMutex _l(mLock);

        // 在矢量数组中查找"句柄值为handle的handle_entry对象"；
        // 找到的话，则直接返回；找不到的话，则新建handle对应的handle_entry。
        handle_entry* e = lookupHandleLocked(handle);

        if (e != NULL) {
            IBinder* b = e->binder;
            if (b == NULL || !e->refs->attemptIncWeak(this)) {
                // 当handle==0(即是Service Manager的句柄)时，尝试去ping Binder驱动。
                if (handle == 0) {
                    Parcel data;
                    status_t status = IPCThreadState::self()->transact(
                            0, IBinder::PING_TRANSACTION, data, NULL, 0);
                    if (status == DEAD_OBJECT)
                       return NULL;
                }

                // 新建BpBinder代理
                b = new BpBinder(handle);
                e->binder = b;
                if (b) e->refs = b->getWeakRefs();
                result = b;
            } else {
                ...
            }
        }

        return result;
    }

这样，getService()的内容就全部执行完毕。getService()的返回结果IBinder=BpBinder对象，该对象包含了"MediaPlayerService(在Binder驱动)中的Binder引用的描述"，该描述在C++层而言就是个整型句柄。之后，若MediaPlayer要向MediaPlayerService发送请求，就根据该IBinder对象和"MediaPlayerService"进行通信。


上面只是执行完了getService()，它返回了IBinder对象。但是，getMediaPlayerService()并没有执行完毕。下面继续回到getMediaPlayerService()中。


<a name="anchor7"></a>
# 7. IMediaDeathNotifier::getMediaPlayerService()

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

说明：在成功获取MediaPlayerService对应的IBinder对象(binder)之后，可以通过interface_cast<IMediaPlayerService>(binder)获取它的代理。  
是不是对interface_cast()很熟悉！不错，在[Android Binder机制(四) defaultServiceManager()的实现][link_binder_04_defaultServiceManager]中就是通过该宏获取IServiceManager的代理的。


<a name="anchor8"></a>
# 8. IMediaDeathNotifier::getMediaPlayerService()

    template<typename INTERFACE>
    inline sp<INTERFACE> interface_cast(const sp<IBinder>& obj)
    {
        return INTERFACE::asInterface(obj);
    }   

下面直接给出IMediaPlayerService::asInterface()的代码。

        android::sp<IMediaPlayerService> IMediaPlayerService::asInterface(
                const android::sp<android::IBinder>& obj)
        {
            android::sp<IMediaPlayerService> intr;
            if (obj != NULL) {
                intr = static_cast<IMediaPlayerService*>(
                    obj->queryLocalInterface(
                            IMediaPlayerService::descriptor).get());
                if (intr == NULL) {
                    intr = new BpServiceManager(obj);
                }
            }
            return intr;
        }

说明：asInterface()会调用new BpMediaPlayerService()新建BpServiceManager对象，并返回给对象。


这样，MediaPlayer进程的getService请求就全部介绍完毕了。



[link_binder_01_introduce]: /2014/09/01/Binder-Introduce/
[link_binder_02_datastruct]: /2014/09/02/Binder-Datastruct/
[link_binder_03_ServiceManagerDeamon]: /2014/09/03/Binder-ServiceManager-Daemon/
[link_binder_04_defaultServiceManager]: /2014/09/04/Binder-defaultServiceManager/
[link_binder_05_addService01]: /2014/09/05/BinderCommunication-AddService01/

