---
layout: post
title: "Android Binder机制(七) addService详解03之 请求的反馈"
description: "android"
category: android
tags: [android]
date: 2014-09-05 09:03
---


> 前面两篇文章分别介绍了addService中"请求的发送"和"请求的处理"这两部分，本文将介绍addService请求的最后一部分--请求的反馈。  
> ServiceManager在处理完addService请求之后，添加了一个待处理事务到MediaPlayerService的事务列表中，并将MediaPlayerService唤醒。我们从上次MediaPlayerService休眠的地方开始，看看它被唤醒之后干了些什么。


> 注意：本文是基于Android 4.4.2版本进行介绍的！



<a name="anchor1"></a>
# 1. Binder驱动中binder_thread_read()的源码

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
              // 阻塞式的读取，则阻塞等待事务的发生。
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

            // t->buffer->target_node是NULL
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
                tr.sender_pid = 0;
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
            // 设置回复信息
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

说明：MediaPlayerService进程被Service Manager唤醒，同时它的待处理事务队列中有ServiceManager添加的事务；此时，binder_has_thread_work()为true。因此，MediaPlayerService会继续往下执行。  
(01) 进入while循环后，首先取出待处理事务。  
(02) 事务的类型是BINDER_WORK_TRANSACTION，得到对应的binder_transaction*类型指针t之后，跳出switch语句。时t不为NULL，因此继续往下执行。下面的工作的目的，是将t中的数据转移到tr中(tr是事务交互数据包结构体binder_transaction_data对应的指针)，然后将指令和tr数据都拷贝到用户空间，让MediaPlayerService读取后进行处理。此时的指令是BR_REPLY。

binder_thread_read()执行完毕之后，共反馈了两个指令到用户空间：BR_NOOP和BR_REPLY

现在回到MediaPlayerService位于用户空间的进程。它会逐个解析Binder驱动反馈的指令。  
对于BR_NOOP，MediaPlayerService不会做任何实质性的动作。  
对于BR_REPLY，看看MediaPlayerService的处理流程。


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

说明：ipcSetDataReference()是根据参数的值重新初始化Parcel的数据和对象。  
(01) freeDataNoInit()的目的是释放原有的内存。为接下来保存Binder驱动反馈的数据做准备。  
(02) 在[Android Binder机制(六) addService详解02之 请求的处理][link_binder_05_addService02]中，ServiceManager反馈数据时，我们知道它对应的BR_REPLY的数据实际上是空的！因此，这里的mDataSize和mObjectsSize都是0。

实际上，Binder驱动反馈给MediaPlayerService的指令就是告诉它addService已经成功处理完毕！

在MediaPlayerService解析完Binder驱动反馈的数据之后，它会层层向上返回。这样，MediaPlayerService::instantiate()也就正式执行完了！  
MediaPlayerService::instantiate()执行完毕，但是MediaPlayerService进程似乎还没有进入消息循环中等到Client的请求！那么，它是何时进入消息循环的呢？回到MediaPlayerService进程的main()函数入口中，它后面是通过startThreadPool()进入消息循环的。这部分的内容，我们下一章再来介绍。

    int main(int argc, char** argv)
    {
        ...

        if (doLog && (childPid = fork()) != 0) {
            ...
        } else {
            ...
            MediaPlayerService::instantiate();
            ...
            ProcessState::self()->startThreadPool();
            IPCThreadState::self()->joinThreadPool();
        }
    }


[link_binder_01_introduce]: /2014/09/01/Binder-Introduce/
[link_binder_02_datastruct]: /2014/09/02/Binder-Datastruct/
[link_binder_03_ServiceManagerDeamon]: /2014/09/03/Binder-ServiceManager-Daemon/
[link_binder_04_defaultServiceManager]: /2014/09/04/Binder-defaultServiceManager/
[link_binder_05_addService01]: /2014/09/05/BinderCommunication-AddService01/
[link_binder_05_addService02]: /2014/09/05/BinderCommunication-AddService02/
