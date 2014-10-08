---
layout: post
title: "Android Binder机制(十) getService详解02之 请求的处理"
description: "android"
category: android
tags: [android]
date: 2014-09-07 09:02
---


> 前面介绍了getService请求的发送部分，本文接着介绍请求的处理部分。下面看看ServiceManager被唤醒之后，是如何处理getService请求的

> 注意：本文是基于Android 4.4.2版本进行介绍的！



<a name="anchor1"></a>
# 1. Binder驱动中binder_thread_read()的源码

前面说到，MediaPlayer线程在执行binder_transaction()时，会将一个待处理事务添加到"ServiceManager的待处理事务队列"中；然后，再将ServiceManager进程唤醒。  
下面，我们就接着看看ServiceManager被唤醒之后做了些什么。

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
            // 这里，MediaPlayer的getService请求的目标是ServiceManager，因此target_node是Service Manager对应的节点；
            if (t->buffer->target_node) {
                // 事务目标对应的Binder实体(即，ServiceManager对应的Binder实体)
                struct binder_node *target_node = t->buffer->target_node;
                // Binder实体在用户空间的地址(ServiceManager的ptr为NULL)
                tr.target.ptr = target_node->ptr;
                // Binder实体在用户空间的其它数据(ServiceManager的cookie为NULL)
                tr.cookie =  target_node->cookie;
                t->saved_priority = task_nice(current);
                if (t->priority < target_node->min_priority &&
                    !(t->flags & TF_ONE_WAY))
                    binder_set_nice(t->priority);
                else if (!(t->flags & TF_ONE_WAY) ||
                     t->saved_priority > target_node->min_priority)
                    binder_set_nice(target_node->min_priority);
                cmd = BR_TRANSACTION;
            } else {
                ...
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
                // 将事务t添加到当前线程的事务栈transaction_stack中。
                // 这是因为，Binder驱动需要等待Service Manager的反馈。
                t->to_parent = thread->transaction_stack;
                t->to_thread = thread;
                thread->transaction_stack = t;
            } else {
                ...
            }
            break;
        }

    done:

        // 更新bwr.read_consumed的值
        *consumed = ptr - buffer;

        ...
        return 0;
    }

说明：ServiceManager进程被唤醒之后，binder_has_thread_work()为true，因为ServiceManager中有个待处理事务(即，MediaPlayer的getService求)。  
(01) 进入while循环后，首先取出待处理事务。  
(02) 事务的类型是BINDER_WORK_TRANSACTION，得到对应的binder_transaction*类型指针t之后，跳出switch语句。很显然，此时t不为NULL，因此继续往下执行。下面的工作的目的，是将t中的数据转移到tr中(tr是事务交互数据包结构体binder_transaction_data对应的指针)，然后将指令和tr数据都拷贝到用户空间，让ServiceManager读取后进行处理。此时的指令为BR_TRANSACTION！  
(03) 最后，更新*consumed的值，即更新bwr.read_consumed的值。


然后，binder_thread_read()会返回到binder_ioctl()中。binder_ioctl()在将数据bwr拷贝到用户空间之后会返回。这样，就又回到了ServiceManager守护进程中。
 


<a name="anchor2"></a>
# 2. binder_loop

    void binder_loop(struct binder_state *bs, binder_handler func)
    {
        ..

        for (;;) {
            ...
            res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);

            ...
            res = binder_parse(bs, 0, readbuf, bwr.read_consumed, func);
            ...
        }
    }   

说明：该代码在frameworks/native/cmds/servicemanager/binder.c中。Binder驱动共反馈了BR_NOOP和BR_TRANSACTION两个指令给Service Manager守护进程。BR_NOOP什么实质性的工作也不会做，我们直接分析BR_TRANSACTION的处理情况。


<a name="anchor3"></a>
# 3. binder_parse

    int binder_parse(struct binder_state *bs, struct binder_io *bio,
                     uint32_t *ptr, uint32_t size, binder_handler func)
    {
        int r = 1;
        uint32_t *end = ptr + (size / 4);

        while (ptr < end) {
            uint32_t cmd = *ptr++;

            switch(cmd) {
            case BR_NOOP:
                break;
            ...
            case BR_TRANSACTION: {
                struct binder_txn *txn = (void *) ptr;
                ...
                if (func) {
                    unsigned rdata[256/4];
                    struct binder_io msg;   // 用于保存"Binder驱动反馈的信息"
                    struct binder_io reply; // 用来保存"回复给Binder驱动的信息"
                    int res;

                    // 初始化reply
                    bio_init(&reply, rdata, sizeof(rdata), 4);
                    // 根据txt(Binder驱动反馈的信息)初始化msg
                    bio_init_from_txn(&msg, txn);
                    // 消息处理
                    res = func(bs, txn, &msg, &reply);
                    // 反馈消息给Binder驱动。
                    binder_send_reply(bs, &reply, txn->data, res);
                }
                ptr += sizeof(*txn) / sizeof(uint32_t);
                break;
            }
            ...
            }
        }

        return r;
    }

说明：这里只关注BR_TRANSACTION分支。 首先，用bio_init()初始化reply。然后通过bio_init_from_txn()初始化msg。接着，是通过func函数指针对数据进行处理，func指向svcmgr_handler。处理完毕，再通过binder_send_reply()填写反馈信息给Binder驱动。  
这里的大部分内容在[Android Binder机制(六) addService详解02之 请求的处理][link_binder_05_addService02]中都介绍过，这里重点关注svcmgr_handler()处理getService请求的流程。


<a name="anchor4"></a>
# 4. svcmgr_handler

    int svcmgr_handler(struct binder_state *bs,
                       struct binder_txn *txn,
                       struct binder_io *msg,
                       struct binder_io *reply)
    {
        ...
        // 数据有效性检测(数据头)
        strict_policy = bio_get_uint32(msg);
        s = bio_get_string16(msg, &len);
        if ((len != (sizeof(svcmgr_id) / 2)) ||
            memcmp(svcmgr_id, s, sizeof(svcmgr_id))) {
            ...
        }

        switch(txn->code) {
            case SVC_MGR_GET_SERVICE:
            case SVC_MGR_CHECK_SERVICE:
                s = bio_get_string16(msg, &len);
                ptr = do_find_service(bs, s, len, txn->sender_euid);
                if (!ptr)
                    break;
                bio_put_ref(reply, ptr);
                return 0;
            ...
        }

        bio_put_uint32(reply, 0);
        return 0;
    }

说明：该代码在frameworks/native/cmds/servicemanager/service_manager.c中。svcmgr_handler()首先读取出getService请求的消息头，进行有效性检测。然后，取出请求的编码；这里请求编码对应是SVC_MGR_CHECK_SERVICE。接着，便进入对应的switch分支。  
(01) 通过bio_get_string16()获取请求的IBinder对象的名称，即s="media.player"。  
(02) 然后，通过do_find_service()查找名称为s的IBinder对象。  


<a name="anchor5"></a>
# 5. do_find_service

    void *do_find_service(struct binder_state *bs, uint16_t *s, unsigned len, unsigned uid)
    {
        struct svcinfo *si;
        si = find_svc(s, len);

        if (si && si->ptr) {
            ...
            return si->ptr;
        } else {
            return 0;
        }
    }

    struct svcinfo *find_svc(uint16_t *s16, unsigned len)
    {
        struct svcinfo *si;

        for (si = svclist; si; si = si->next) {
            if ((len == si->len) &&
                !memcmp(s16, si->name, len * sizeof(uint16_t))) {
                return si;
            }
        }
        return 0;
    }

说明：  
(01) do_find_service()会调用find_svc()进行查找。在find_svc()中，会在svclist链表中查找是否有名称等于"media.player"的svcinfo对象。很显然，在[Android Binder机制(六) addService详解02之 请求的处理][link_binder_05_addService02]中，已经将MediaPlayerService注册到svclist中，而MediaPlayerService的名称就是"media.player"。  
(02) find_svc()找到svcinfo对象后返回到do_find_service()中。此时，if (si && si->ptr)为true，返回si->ptr。这里的si->ptr就是MediaPlayerService在Binder驱动中的Binder引用的描述。根据该引用描述，就能找到对应的MediaPlayerService对象。


随后，在成功获取Binder引用的描述之后，svcmgr_handler()会调用bio_put_ref()将该引用信息写入到binder_object中。


<a name="anchor6"></a>
# 6. bio_put_ref()

    void bio_put_ref(struct binder_io *bio, void *ptr)
    {
        struct binder_object *obj;
        
        if (ptr) 
            obj = bio_alloc_obj(bio);
        else
            obj = bio_alloc(bio, sizeof(*obj));
            
        if (!obj)
            return;
        
        obj->flags = 0x7f | FLAT_BINDER_FLAG_ACCEPTS_FDS;
        // 类型
        obj->type = BINDER_TYPE_HANDLE;
        // 句柄地址
        obj->pointer = ptr;
        obj->cookie = 0;
    }

说明：bio_put_ref()会将获取到的Binder引用描述打包到结构体binder_object中。而binder_object是与flat_binder_object对应的结构体，Binder驱动在收到个数据之后，就能对flat_binder_object进行解析处理。


在bio_put_ref()将数据打包到reply中之后，svcmgr_handle会调用binder_send_reply()将数据和指令整合到一起。


<a name="anchor7"></a>
# 7. binder_send_reply()

    void binder_send_reply(struct binder_state *bs,
                           struct binder_io *reply,
                           void *buffer_to_free,
                           int status)
    {
        struct {
            uint32_t cmd_free;
            void *buffer;
            uint32_t cmd_reply;
            struct binder_txn txn;
        } __attribute__((packed)) data;
        
        data.cmd_free = BC_FREE_BUFFER;
        data.buffer = buffer_to_free;
        data.cmd_reply = BC_REPLY;
        data.txn.target = 0;
        data.txn.cookie = 0;
        data.txn.code = 0;
        if (status) {
            ...
        } else {
            data.txn.flags = 0;
            data.txn.data_size = reply->data - reply->data0;
            data.txn.offs_size = ((char*) reply->offs) - ((char*) reply->offs0);
            data.txn.data = reply->data0;
            data.txn.offs = reply->offs0;
        }
        binder_write(bs, &data, sizeof(data));
    }

说明：binder_send_reply()在[Android Binder机制(六) addService详解02之 请求的处理][link_binder_05_addService02]中已经介绍过。它共打包了两个指令：BC_FREE_BUFFER和BC_REPLY。在函数最后，它调用binder_write()和Binder驱动交互。


<a name="anchor8"></a>
# 8. binder_write()

    int binder_write(struct binder_state *bs, void *data, unsigned len)
    {
        struct binder_write_read bwr;
        int res;
        bwr.write_size = len;
        bwr.write_consumed = 0;
        bwr.write_buffer = (unsigned) data;
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

说明：binder_write()会将数据封装到binder_write_read的变量bwr中；其中，bwr.read_size=0，而bwr.write_size>0。接着，便通过ioctl(,BINDER_WRITE_READ,)和Binder驱动交互，将数据反馈给Binder驱动。


再次回到Binder驱动的binder_ioctl()对应的BINDER_WRITE_READ分支中。此时，由于bwr.read_size=0，而bwr.write_size>0；因此，Binder驱动只调用binder_thread_write进行写操作，而不会进行读。


<a name="anchor9"></a>
# 9. Binder驱动中binder_thread_write()的源码

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
            case BC_FREE_BUFFER: 
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

说明：binder_thread_write()会逐个读出"Service Manager反馈的指令"。  
(01) 第一个指令是BC_FREE_BUFFER。binder_thread_write()进入BC_FREE_BUFFER对应的分支后后，执行的动作主要是释放"保存MediaPlayer请求数据的缓冲"。  
(02) 第二个指令是BC_REPLY。binder_thread_write()进入BC_REPLY之后，会将数据拷贝到内核空间，然后调用binder_transaction()进行处理



<a name="anchor10"></a>
# 10. Binder驱动中binder_transaction()的源码

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
            // 事务栈
            in_reply_to = thread->transaction_stack;
            ...
            // 设置优先级
            binder_set_nice(in_reply_to->saved_priority);
            ...
            thread->transaction_stack = in_reply_to->to_parent;
            // 发起请求的线程，即MediaPlayer所在线程。
            // from的值，是MediaPlayer发起请求时在binder_transaction()中赋值的。
            target_thread = in_reply_to->from;
            ...
            // MediaPlayer对应的进程
            target_proc = target_thread->proc;
        } else {
            ...
        }
        if (target_thread) {
            e->to_thread = target_thread->pid;
            target_list = &target_thread->todo;
            target_wait = &target_thread->wait;
        } else {
            ...
        }
        e->to_proc = target_proc->pid;

        // 分配一个待处理的事务t，t是binder事务(binder_transaction对象)
        t = kzalloc(sizeof(*t), GFP_KERNEL);
        if (t == NULL) {
            return_error = BR_FAILED_REPLY;
            goto err_alloc_t_failed;
        }

        // 分配一个待完成的工作tcomplete，tcomplete是binder_work对象。
        tcomplete = kzalloc(sizeof(*tcomplete), GFP_KERNEL);
        if (tcomplete == NULL) {
            return_error = BR_FAILED_REPLY;
            goto err_alloc_tcomplete_failed;
        }
        binder_stats_created(BINDER_STAT_TRANSACTION_COMPLETE);

        t->debug_id = ++binder_last_id;
        e->debug_id = t->debug_id;

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

        // 分配空间
        t->buffer = binder_alloc_buf(target_proc, tr->data_size,
            tr->offsets_size, !reply && (t->flags & TF_ONE_WAY));
        if (t->buffer == NULL) {
            return_error = BR_FAILED_REPLY;
            goto err_binder_alloc_buf_failed;
        }
        t->buffer->allow_user_free = 0;
        t->buffer->debug_id = t->debug_id;
        // 保存事务
        t->buffer->transaction = t;
        // target_node为NULL
        t->buffer->target_node = target_node;
        trace_binder_transaction_alloc_buf(t->buffer);
        if (target_node)
            binder_inc_node(target_node, 1, 0, NULL);

        offp = (size_t *)(t->buffer->data + ALIGN(tr->data_size, sizeof(void *)));

        // 将"用户传入的数据"保存到事务中
        if (copy_from_user(t->buffer->data, tr->data.ptr.buffer, tr->data_size)) {
            ...
        }
        // 将"用户传入的数据偏移地址"保存到事务中
        if (copy_from_user(offp, tr->data.ptr.offsets, tr->offsets_size)) {
            ...
        }

        ...
        off_end = (void *)offp + tr->offsets_size;
        // 将flat_binder_object对象读取出来，
        // 这里就是Service Manager中反馈的MediaPlayerService对象。
        for (; offp < off_end; offp++) {
            struct flat_binder_object *fp;
            ...
            fp = (struct flat_binder_object *)(t->buffer->data + *offp);

            switch (fp->type) {
                ...
                case BINDER_TYPE_HANDLE:
                case BINDER_TYPE_WEAK_HANDLE: {
                    // 根据handle获取对应的Binder引用，即得到MediaPlayerService的Binder引用
                    struct binder_ref *ref = binder_get_ref(proc, fp->handle);
                    if (ref == NULL) {
                        ...
                    }
                    // ref->node->proc是MediaPlayerService的进程上下文环境，
                    // 而target_proc是MediaPlayer的进程上下文环境
                    if (ref->node->proc == target_proc) {
                        ...
                    } else {
                        struct binder_ref *new_ref;
                        // 在MediaPlayer进程中引用"MediaPlayerService"。
                        // 表现为，执行binder_get_ref_for_node()会，会先在MediaPlayer进程中查找是否存在MediaPlayerService对应的Binder引用；
                        // 很显然是不存在的。于是，并新建MediaPlayerService对应的Binder引用，并将其添加到MediaPlayer的Binder引用红黑树中。
                        new_ref = binder_get_ref_for_node(target_proc, ref->node);
                        if (new_ref == NULL) {
                            ...
                        }
                        // 将new_ref的引用描述复制给fp->handle。
                        fp->handle = new_ref->desc;
                        binder_inc_ref(new_ref, fp->type == BINDER_TYPE_HANDLE, NULL);
                        ...
                    }
                } break;

            }
        }

        if (reply) {
            binder_pop_transaction(target_thread, in_reply_to);
        } else if (!(t->flags & TF_ONE_WAY)) {
            ...
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

说明：reply=1，这里只关注reply部分。  
(01) 此反馈最终是要回复给MediaPlayer的。因此，target_thread被赋值为MediaPlayer所在的线程，target_proc则是MediaPlayer对应的进程，target_node为null。  
(02) 这里，先看看for循环里面的内容，取出BR_REPLY指令所发送的数据，然后获取数据中的flat_binder_object变量fp。因为fp->type为BINDER_TYPE_HANDLE，因此进入BINDER_TYPE_HANDLE对应的分支。接着，通过binder_get_ref()获取MediaPlayerService对应的Binder引用；很明显，能够正常获取到MediaPlayerService的Binder引用。因为在MediaPlayerService调用addService请求时，已经创建了它的Binder引用。 binder_get_ref_for_node()的作用是在MediaPlayer进程上下文中添加"MediaPlayerService对应的Binder引用"。这样，后面就可以根据该Binder引用一步步的获取MediaPlayerService对象。 最后，将Binder引用的描述赋值给fp->handle。  
(03) 此时，Service Manager已经处理了getService请求。便调用binder_pop_transaction(target_thread, in_reply_to)将事务从"target_thread的事务栈"中删除，即从MediaPlayer线程的事务栈中删除该事务。  
(04) 新建的"待处理事务t"的type为设为BINDER_WORK_TRANSACTION后，会被添加到MediaPlayer的待处理事务队列中。  
(05) 此时，Service Manager已经处理了getService请求，而Binder驱动在等待它的回复。于是，将一个BINDER_WORK_TRANSACTION_COMPLETE类型的"待完成工作tcomplete"(作为回复)添加到当前线程(即，Service Manager线程)的待处理事务队列中。  
(06) 最后，调用wake_up_interruptible()唤醒MediaPlayer。MediaPlayer被唤醒后，会对事务BINDER_WORK_TRANSACTION进行处理。


OK，到现在为止，还有两个待处理事务：(01) ServiceManager待处理事务列表中有个BINDER_WORK_TRANSACTION_COMPLETE类型的事务 (02) MediaPlayer待处理事务列表中有个BINDER_WORK_TRANSACTION事务。

关于BINDER_WORK_TRANSACTION_COMPLETE事务，它是用来告诉ServiceManager，ServiceManager的反馈信息已经处理完毕。下一篇文章，就说说MediaPlayer被唤醒后，执行BINDER_WORK_TRANSACTION的流程。



[link_binder_01_introduce]: /2014/09/01/Binder-Introduce/
[link_binder_02_datastruct]: /2014/09/02/Binder-Datastruct/
[link_binder_03_ServiceManagerDeamon]: /2014/09/03/Binder-ServiceManager-Daemon/
[link_binder_04_defaultServiceManager]: /2014/09/04/Binder-defaultServiceManager/
[link_binder_05_addService01]: /2014/09/05/BinderCommunication-AddService01/
[link_binder_05_addService02]: /2014/09/05/BinderCommunication-AddService02/
[link_binder_05_addService03]: /2014/09/05/BinderCommunication-AddService03/
