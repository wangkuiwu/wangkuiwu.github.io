---
layout: post
title: "Android Binder机制(六) addService详解02之 请求的处理"
description: "android"
category: android
tags: [android]
date: 2014-09-05 09:02
---


> [前面一文][link_binder_05_addService01]介绍了addService的请求发送部分，Binder驱动在处理addService请求时，将一个待处理事务添加到ServiceManager中，然后将ServiceManager唤醒。在[Android Binder机制(三) ServiceManager守护进程][link_binder_03_ServiceManagerDeamon]的末尾，我们说过ServiceManager启动之后，由于没有事务可处理，就进入了等待状态。这里，从ServiceManager被唤醒后开始讲解。

> **目录**  
> **1**. [Android消息机制的架构](#anchor1)  

> 注意：本文是基于Android 4.4.2版本进行介绍的！



<a name="anchor1"></a>
# 1. Binder驱动中binder_thread_read()的源码

下面，就接着[Android Binder机制(三) ServiceManager守护进程][link_binder_03_ServiceManagerDeamon]中的休眠部分进行讲解，看看Service Manager被唤醒后，会干些什么。


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

            // t->buffer->target_node是目标节点。
            // 这里，addService请求的目标是ServiceManager，因此target_node是ServiceManager对应的节点；
            // 它的值在事务交互时(binder_transaction中)，被赋值为ServiceManager对应的Binder实体。  
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
                // 该事务会发送给Service Manager守护进程进行处理。
                // Service Manager处理之后，还需要给Binder驱动回复处理结果。
                // 这里设置Binder驱动回复信息。
                t->to_parent = thread->transaction_stack;
                // to_thread表示Service Manager反馈后，将反馈结果交给当前thread进行处理
                t->to_thread = thread;
                // transaction_stack交易栈保存当前事务。用于之处反馈是针对哪个事务的。
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

说明：ServiceManager进程在调用wait_event_interruptible_exclusive(proc->wait, binder_has_proc_work(proc, thread))进入等待之后，被MediaPlayerService进程唤醒。唤醒之后，binder_has_thread_work()为true，因为ServiceManager的待处理事务队列中有个待处理事务(即，MediaPlayerService添加服务的请求)。  
(01) 进入while循环后，首先取出待处理事务。  
(02) 事务的类型是BINDER_WORK_TRANSACTION，得到对应的binder_transaction*类型指针t之后，跳出switch语句。很显然，此时t不为NULL，因此继续往下执行。下面的工作的目的，是将t中的数据转移到tr中(tr是事务交互数据包结构体binder_transaction_data对应的指针)，然后将指令和tr数据都拷贝到用户空间，让ServiceManager读取后进行处理。  
下面列举比较重要的几个部分进行说明。


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

这里着重强调一下地址的赋值方式，因为它涉及到Binder机制的数据拷贝原理！   
t->buffer是在binder_transaction()中，通过binder_alloc_buf()分配的内核空间地址。现在要将数据返回给Service Manager守护进程，需要将内核空间的数据拷贝到用户空间。如果你还记得的话，前面在[Android Binder机制(三) ServiceManager守护进程][link_binder_03_ServiceManagerDeamon]的mmap()中，我们将内核虚拟地址和进程虚拟地址映射到同一个物理存储区；现在，已知内核虚拟地址(即t->buffer->data)。那么，只需要将t->buffer->data加上proc->user_buffer_offset(内核虚拟地址和进程虚拟地址的偏移)即可得到在用户空间的地址。  

在tr赋值完毕之后，就将完整数据拷贝到用户空间。此时，该事务已经在Binder驱动中被处理，于是将事务从Service Manager的待处理事务队列中删除。Binder驱动随后会将该事务发送给Service Manager守护进程，Service Manager守护进程在处理完事务之后，需要反馈结果给Binder驱动。因此，接下来会设置t->to_thread和t->transaction_stack等成员。最后，修改*consumed的值，即bwr.read_consumed的值，表示待读取内容的大小。  
执行完binder_thread_read()之后，回到binder_ioctl()中，执行copy_to_user()将数据拷贝到用户空间。接下来，就回到了Service Manager的守护进程当中，即回到binder_loop()中。


<a name="anchor2"></a>
# 2. binder_loop()

    void binder_loop(struct binder_state *bs, binder_handler func)
    {
        struct binder_write_read bwr;
        unsigned readbuf[32];
        ...
        
        for (;;) {
            bwr.read_size = sizeof(readbuf);
            bwr.read_consumed = 0;
            bwr.read_buffer = (unsigned) readbuf;

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

说明：binder_loop()会将ioctl()反馈的数据发送给binder_parse()进行解析。



<a name="anchor3"></a>
# 3. binder_parse()

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

说明：此处里的cmd就是bwr.read_buffer指针。而在Binder驱动的binder_thread_read()中，反馈的第一个指令是BR_NOOP；因此这里的cmd=BR_NOOP，不执行任何动作，继续取出下一个指令cmd=BR_TRANSACTION。在BR_TRANSACTION中，会先取出消息，在对消息处理之后，再将反馈信息发送给Binder驱动。下面是BR_TRANSACTION的详细内容。  
(01) 首先，将ptr转换成struct binder_txn结构体指针。struct binder_txn是与binder_transaction_datad对应的结构体，在[Android Binder机制(二) Binder中的数据结构][link_binder_02_datastruct]中有它的详细介绍。  
(02) 此处的func是函数指针svcmgr_handler，不为空；因此，先调用bio_init()初始化reply，再调用bio_init_from_txn()来初始化msg。  
(03) 初始化完毕之后，就调用svcmgr_handler()对消息进行处理。  
(04) 消息处理完毕，就通过binder_send_reply()将处理结果反馈给Binder驱动。  



<a name="anchor4"></a>
# 4. bio_init()

    void bio_init(struct binder_io *bio, void *data,
                  uint32_t maxdata, uint32_t maxoffs)
    {               
        uint32_t n = maxoffs * sizeof(uint32_t);
                
        if (n > maxdata) {
            bio->flags = BIO_F_OVERFLOW;
            bio->data_avail = 0;
            bio->offs_avail = 0;            
            return;
        }       
                    
        bio->data = bio->data0 = (char *) data + n;
        bio->offs = bio->offs0 = data;
        bio->data_avail = maxdata - n;
        bio->offs_avail = maxoffs;
        bio->flags = 0;
    }

说明：bio_init()就是对struct binder_io的各个成员赋值。


<a name="anchor5"></a>
# 5. bio_init_from_txn()

    void bio_init_from_txn(struct binder_io *bio, struct binder_txn *txn)
    {           
        bio->data = bio->data0 = txn->data;    // 数据起始地址
        bio->offs = bio->offs0 = txn->offs;    // 数据中对象的偏移数组的起始地址
        bio->data_avail = txn->data_size;      // 数据大小
        bio->offs_avail = txn->offs_size / 4;  // 对象个数
        bio->flags = BIO_F_SHARED;
    }

说明：bio_init_from_txn()就是根据已有的数据txn初始化struct binder_io的各个成员。 



<a name="anchor6"></a>
# 6. svcmgr_handler()

    int svcmgr_handler(struct binder_state *bs,
                       struct binder_txn *txn,
                       struct binder_io *msg,
                       struct binder_io *reply)
    {
        struct svcinfo *si;
        uint16_t *s;
        unsigned len;
        void *ptr;  
        uint32_t strict_policy;
        int allow_isolated;
                
        if (txn->target != svcmgr_handle)
            return -1;

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
                ...

            case SVC_MGR_ADD_SERVICE:
                s = bio_get_string16(msg, &len);
                ptr = bio_get_ref(msg);
                allow_isolated = bio_get_uint32(msg) ? 1 : 0;
                if (do_add_service(bs, s, len, ptr, txn->sender_euid, allow_isolated))
                    return -1;
                break;
            case SVC_MGR_LIST_SERVICES:
                ...
        }

        bio_put_uint32(reply, 0);
        return 0;
    }

说明：  
(01) txt->target对应tr.target.ptr，而tr.target.ptr是Binder驱动的在binder_thread_read()中赋值的，它指向Service Manager的Binder实体在用户空间的句柄，是NULL。而svcmgr_handle=BINDER_SERVICE_MANAGER=((void*) 0)。显然，txt->target=svcmgr_handler。  
(02) 接下来，先通过bio_get_uint32(msg)和bio_get_string16(msg, &len)进行有效性检测。通过bio_get_uint32()从msg中取出32位的整型数，就是MediaPlayerService请求数据中的STRICT_MODE_PENALTY_GATHER。然后，通过bio_get_string16(msg, &len)获取数据中字符串，也就是"android.os.IServiceManager"。接着，将该字符串和svcmgr_id进行比较(依次比较长度和内容)；很显然，这里是相当的。  
(03) 在通过有效性检测之后，就根据相应的事务编码进行处理。这里txt->code的值是SVC_MGR_ADD_SERVICE。先通过bio_get_string16()获取MediaPlayerService的名称，也就是s="media.player"，然后就通过bio_get_ref()获取MediaPlayerService对象的引用。  


<a name="anchor7"></a>
# 7. svcmgr_handler()

    void *bio_get_ref(struct binder_io *bio)
    {   
        struct binder_object *obj;
        
        obj = _bio_get_obj(bio);
        if (!obj)
            return 0;

        if (obj->type == BINDER_TYPE_HANDLE)
            return obj->pointer;
        
        return 0;
    }       

说明：binder_object是与flat_binder_object对应的结构体，关于它的详细介绍可以参考[Android Binder机制(二) Binder中的数据结构][link_binder_02_datastruct]。  
(01) _bio_get_obj(bio)的代码就不展开了，它是根据bio创建binder_object对象。实际上，obj就是MediaPlayerService打包成的flat_binder_object对象。  
(02) obj->type的值是BINDER_TYPE_HANDLE。原来MediaPlayerService对应的type是BINDER_TYPE_BINDER，但在Binder驱动的binder_transaction()中，将type修改成了BINDER_TYPE_HANDLE。因此，返回obj->pointer，而obj->pointer实际上是flat_binder_object中的handle，而该handle在Binder驱动中被赋值为"MediaPlayerService对应的Binder引用的描述，即binder_ref->desc"。根据该引用描述，可以在Binder驱动中找到MediaPlayerService对应的Binder实体以及MediaPlayerService对应的进程上下文信息，进而可以给MediaPlayerService发送消息。  



<a name="anchor8"></a>
# 8. svcmgr_handler()

接下来，回到svcmgr_handler()中，继续执行do_add_service()。


    int do_add_service(struct binder_state *bs,
                       uint16_t *s, unsigned len,
                       void *ptr, unsigned uid, int allow_isolated)
    {
        struct svcinfo *si;
        ...
                
        if (!svc_can_register(uid, s)) {
            ...
        }

        si = find_svc(s, len);
        if (si) {
            ...
        } else {
            si = malloc(sizeof(*si) + (len + 1) * sizeof(uint16_t));
            if (!si) { 
                ...
            }
            si->ptr = ptr;
            si->len = len;
            memcpy(si->name, s, (len + 1) * sizeof(uint16_t));
            si->name[len] = '\0';
            si->death.func = svcinfo_death;
            si->death.ptr = si;
            si->allow_isolated = allow_isolated;
            si->next = svclist;
            svclist = si;
        }
        
        binder_acquire(bs, ptr);
        binder_link_to_death(bs, ptr, &si->death);
        return 0;
    }

说明：do_add_service()是将该MediaPlayerService
(01) 先看看参数。bs是struct binder_state类型，它在保存了打开"/dev/binder"文件的相关信息。s是IBinder对象的名称，即"media.player"。len是s的长度。ptr是MediaPlayerService在Binder驱动中的引用描述。uid是MediaPlayerService的uid。allow_isolated是flase。  
(02) svc_can_register()是检测能否将uid线程的信息注册到Service Manager中。这里，返回true。  
(03) find_svc(s, len)是在Service Manager的服务队列svclist中，查找是否有名称为s的服务。由于之前没有将MediaPlayerService注册到Service Manager中，这里返回的si=null；接下来，就将MediaPlayerService的信息保存到si中，然后再将si注册到svclist中。  
这样，MediaPlayerService就注册到Service Manager中了。



<a name="anchor9"></a>
# 9. svcmgr_handler()

接下来，回到svcmgr_handler()中，调用bio_put_uint32(reply, 0)。这里就不对bio_put_uint32()的代码进行展开了，bio_put_uint32(reply, val)的作用是将val写入到reply中。但是，当val=0时，不会写入任何数据；也就是说bio_put_uint32(reply, 0)不会写入任何数据到reply中！

    int svcmgr_handler(struct binder_state *bs,
                       struct binder_txn *txn,
                       struct binder_io *msg,
                       struct binder_io *reply)
    {
        ...

        switch(txn->code) {

            case SVC_MGR_ADD_SERVICE:
                s = bio_get_string16(msg, &len);
                ptr = bio_get_ref(msg);
                allow_isolated = bio_get_uint32(msg) ? 1 : 0;
                if (do_add_service(bs, s, len, ptr, txn->sender_euid, allow_isolated))
                    return -1;
                break;
                ...
        }

        bio_put_uint32(reply, 0);
        return 0;
    }


接着，回到binder_parse()中，调用binder_send_reply()写入到即将发送Binder的缓冲区中。


<a name="anchor10"></a>
# 10. svcmgr_handler()

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

说明：  
(01) 先看看参数。bs是struct binder_state，它保存了打开"/dev/binder"文件的相关信息。reply没有任何数据。buffer_to_free是对应binder_transaction_data中保存请求数据的buffer缓冲区，它是在Binder驱动的binder_transaction()中分配的。status_t=0。  
(02) 该函数中的私有结构体struct是用来描述返回给Binder驱动的数据。我们知道，Binder机制的交互数据的格式是"指令+数据"。这里，返回的指令有两个BC_FREE_BUFFER和BC_REPLY，BC_FREE_BUFFER是告诉Binder驱动，请求处理完毕，让Binder驱动释放数据缓冲；而BC_REPLY是告诉Binder驱动，这是回复，回复的内容是data.txt.data，实际上，这里的回复内容是空！  
(03) 最后，调用binder_write()将数据打包。



<a name="anchor11"></a>
# 11. binder_write()的源码


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

说明：binder_write()单单只是向Binder驱动发送一个消息，而不会去读取消息反馈。



<a name="anchor12"></a>
## 12. Binder驱动中binder_ioctl()的BINDER_WRITE_READ相关部分的源码

下面我们看看Binder驱动部分的对应代码。


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

说明：bwr.write_size>0，而bwr.read_size=0；因此，只会执行写动作，而不会进行读取动作。下面看看binder_thread_write()到底写了些什么。



<a name="anchor13"></a>
## 13. Binder驱动中binder_thread_write()的源码

    int binder_thread_write(struct binder_proc *proc, struct binder_thread *thread,
                void __user *buffer, int size, signed long *consumed)
    {
        uint32_t cmd; 
        void __user *ptr = buffer + *consumed;
        void __user *end = buffer + size;

        // 读取binder_write_read.write_buffer中的内容。
        // 每次读取32bit(即4个字节)
        while (ptr < end && thread->return_error == BR_OK) {
            if (get_user(cmd, (uint32_t __user *)ptr))
                return -EFAULT;
            ptr += sizeof(uint32_t);

            ...
            switch (cmd) {

            case BC_FREE_BUFFER: {
                void __user *data_ptr;
                struct binder_buffer *buffer;

                // 获取要释放的内存地址
                if (get_user(data_ptr, (void * __user *)ptr))
                    return -EFAULT;
                ptr += sizeof(void *);

                // 根据用户空间地址，得到进程空间地址；
                // 再根据进程空间地址，在proc->allocated_buffers红黑树中进行查找该地址对应的binder_buffer对象。
                buffer = binder_buffer_lookup(proc, data_ptr);
                ...
                // 释放内存
                trace_binder_transaction_buffer_release(buffer);
                binder_transaction_buffer_release(proc, buffer, NULL);
                binder_free_buf(proc, buffer);
                break;
            }
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

说明：在Service Manager中，反馈给Binder驱动的指令有两个，分别是BC_FREE_BUFFER和BC_REPLY。  
(01) binder_write_read()先读出BC_FREE_BUFFER指令，然后释放内存。代码中给出了相应的注释，这里就不再详细说明了。  
(02) 接着，读出BC_REPLY指令，将数据拷贝到内核空间之后，便执行binder_transaction()对数据进行处理。


<a name="anchor14"></a>
## 14. Binder驱动中binder_transaction()的源码

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
            // 发起请求的线程，即MediaPlayerService所在线程。
            // from的值，是MediaPlayerService发起请求时在binder_transaction()中赋值的。
            target_thread = in_reply_to->from;
            ...
            // MediaPlayerService对应的进程
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
        for (; offp < off_end; offp++) {
            ...
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

说明：  
(01) reply=1。这里只关注reply部分。target_thread被赋值为MediaPlayerService所在的线程，而target_proc则是MediaPlayerService对应的进程。  
(02) 接着，会新建一个待处理事务t和待完成的工作tcomplete，并对它们进行初始化。这部分前面已经介绍过了；这里就不再重复说明了。从Service Manager反馈的信息中，仅仅包含了数据0，而没有flat_binder_object对象；因此，off_end=offp，不会执行for循环。  
(03) 此时，MediaPlayerService已经成功的添加到了Server Manager守护进程中，接下来便调用binder_pop_transaction(target_thread, in_reply_to)将事务从"target_thread的事务栈"中删除，即从MediaPlayerService线程的事务栈中删除该事务。  
(04) 之后，便是设置事务的类型为BINDER_WORK_TRANSACTION，然后将其添加到target_list队列中。即，将事务添加到MediaPlayerService的待处理事务队列中。  
(05) 设置待完成工作的类型为BINDER_WORK_TRANSACTION_COMPLETE，然后将其添加到thread->todo中。即，将其添加到当前线程(Service Manager守护进程的线程)的待处理事务队列中。  
(06) 最后，调用wake_up_interruptible()唤醒MediaPlayerService进程。 


此时，Binder驱动就将addService的反馈内容以待处理事务t的方式添加到MediaPlayerService的待处理事务队列当中，并将MediaPlayerService进程唤醒了。而对于待完成工作tcomplete，肯定是告诉ServiceManager进程，它的反馈已经被Binder驱动收到。

下面，还是先说完ServiceManager的流程，然后再来看MediaPlayerService被唤醒后做了什么。


ServiceManager执行完binder_transaction()后，回到binder_thread_write()中；此时，数据已经处理完毕，便返回到binder_ioctl()中。binder_ioctl()将数据拷贝到用户空间后，Binder驱动的工作就结束了。  
于是，又回到ServiceManager守护进程中，binder_write()执行完ioctl()后，返回到binder_send_reply()中，binder_send_reply()则进一步返回到binder_parse()。binder_parse()已经解析完请求数据，于是进一步返回到binder_loop()中。而binder_loop()会再次开始循环，调用ioctl(,BINDER_WRITE_READ,)到Binder驱动执行读操作。  
当ServiceManager再次进入到Binder驱动，并通过binder_ioctl()调用到binder_thread_read()时。由于此时的ServiceManager线程中有一个类型为BINDER_WORK_TRANSACTION_COMPLETE的待处理事务；于是，便取出该事务进行执行。执行完毕之后，将该事务从Service Manager的待处理事务队列中删除，并反馈cmd=BR_TRANSACTION_COMPLETE信息给ServiceManager守护进程。ServiceManager守护进程收到Binder驱动的反馈后，解析出BR_TRANSACTION_COMPLETE，该指令什么也不做；它的目的是让ServiceManager知道，此次addService的反馈已经顺利完成！  
于是，ServiceManager继续它的循环；当它再次调用ioctl()，进而进入到Binder驱动中读取请求时；由于此时的待处理事务队列为空，因此，ServiceManager会再次进入中断等待状态，等待Client的请求。


<br/>
至此，MediaPlayerService进程的addService的请求处理部分就讲解完了。在继续了解请求的反馈之前，先回顾一下本部分的内容。

<a href="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/binder/addService02_deal.jpg"><img src="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/binder/addService02_deal.jpg" alt="" /></a>

MediaPlayerService将addService请求发送到Binder驱动，Binder驱动将addService转换成一个待处理事务并添加到ServiceManager的事务队列中，并将ServiceManager唤醒。ServiceManager被唤醒后，取出该处理；接着，Binder驱动将BR_TRANSACTION发送到ServiceManager守护进程中。ServiceManager通过BR_TRANSACTION解析出addService请求；在从请求数据中解析出MediaPlayerService的相关信息后，并将这些信息存储在一个链表中。接着，ServiceManager守护进程反馈BC_FREE_BUFFER和BC_REPLY给Binder驱动。Binder驱动收到BC_FREE_BUFFER后，释放保存事务数据的内存；在收到BC_REPLY之后，得知ServiceManager已经处理完addService请求。于是，将一个待处理事务添加到MediaPlayerService的事务队列中；然后将MediaPlayerService唤醒。目的是告诉MediaPlayerService，它已经处理完了addService请求。  最后，Binder驱动还需要反馈一个BR_TRANSACTION_COMPLETE给ServiceManager进程，目的是告诉ServiceManager，Binder驱动已经收到了它的回复。


[link_binder_01_introduce]: /2014/09/01/Binder-Introduce/
[link_binder_02_datastruct]: /2014/09/02/Binder-Datastruct/
[link_binder_03_ServiceManagerDeamon]: /2014/09/03/Binder-ServiceManager-Daemon/
[link_binder_04_defaultServiceManager]: /2014/09/04/Binder-defaultServiceManager/
[link_binder_05_addService01]: /2014/09/05/BinderCommunication-AddService01/

