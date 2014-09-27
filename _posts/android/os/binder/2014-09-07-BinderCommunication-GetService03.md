---
layout: post
title: "Android Binder机制(十一) getService详解03之 请求的反馈"
description: "android"
category: android
tags: [android]
date: 2014-09-07 09:03
---


> 本文会介绍Android的消息处理机制。  

> **目录**  
> **1**. [Android消息机制的架构](#anchor1)  

> 注意：本文是基于Android 4.4.2版本进行介绍的！



<a name="anchor1"></a>
# MediaPlayerService的main()函数


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


之后的流程应该都比较熟悉了，首先返回到binder_ioctl()中，接着将数据拷贝到用户空间。接下来的工作就交给MediaPlayer进程进行处理了。  
从Binder驱动返回后，首先回到talkWithDriver()中，接着便返回到waitForResponse()中。在waitForResponse()会反馈数据进行解析。  


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

说明： data就是Binder返回来的数据地址，而objectsCount则为0。先通过freeDataNoInit()将原始的数据清空，然后在给mData赋值，这样就将数据保存到了Parcel中。  


