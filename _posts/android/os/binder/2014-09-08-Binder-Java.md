---
layout: post
title: "Android Binder机制(十二) Binder机制的Java调用流程"
description: "android"
category: android
tags: [android]
date: 2014-09-08 09:01
---


> 前面几篇文章，是基于Binder驱动和C/C++层对Binder机制进行了介绍。本文将从Java引用开始，逐步的分析Client是如何与Server进行交互的。本文的例子还是选取MediaPlayer。

> **目录**  
> **1**. [MediaPlayer的使用示例](#anchor1)  
> **2**. [MediaPlayer示例分析](#anchor2)  




<a name="anchor1"></a>
# 1. MediaPlayer的使用示例

下面是一个调用MediaPlayer播放音乐(test.mp3)的示例代码。

    public class MainActivity extends Activity {
        @Override
        public void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);

            setContentView(R.layout.main);
            testMediaPlayer();
        }   

        private void testMediaPlayer() {
            try {
                MediaPlayer mp = new MediaPlayer();
                mp.setDataSource("/sdcard/test.mp3");
                mp.setAudioStreamType(AudioManager.STREAM_MUSIC);
                mp.prepare();
                mp.start();
            } catch (Exception e) {
                e.printStackTrace();
            }   
        }   
    }

说明：源码很简单，只需要关注testMediaPlayer()部分即可。首先，新建一个MediaPlayer对象。接着，设置数据源和音频流类型。最后，调用prepare()进行准备之后，再通过start()进行播放。

下面，我们就对该过程进行分析，看看该Binder机制是如何参与其中的。


<a name="anchor2"></a>
# 2. MediaPlayer示例分析

我们将MediaPlayer示例就看作一个MediaPlayer进程，接下来，就看看这个MediaPlayer进程是如何通过Binder机制来和MediaPlayerService通信的。重点需要关注的是mp.setDataSource()的实现。

<a name="anchor2_1"></a>
## 2.1 MediaPlayer的构造函数

先看看MediaPlayer的构造函数。


    public MediaPlayer() {

        ...
        native_setup(new WeakReference<MediaPlayer>(this));
    }

    private static native final void native_init();
    private native final void native_setup(Object mediaplayer_this);


说明：该代码在frameworks/base/media/java/android/media/MediaPlayer.java中。在MediaPlayer中，会调用本地方法native_setup()；而且在native_setup()之前，会调用静态方法native_init()进行初始化。下面就看看native_init()和native_setup()各自的代码。


<a name="anchor2_2"></a>
## 2.2 native_init()和native_setup()的注册信息

    static JNINativeMethod gMethods[] = {
        ...
        {"_setDataSource",       "(Ljava/io/FileDescriptor;JJ)V",    (void *)android_media_MediaPlayer_setDataSourceFD},
        ...
        {"_start",              "()V",                              (void *)android_media_MediaPlayer_start},
        ...
        {"native_init",         "()V",                              (void *)android_media_MediaPlayer_native_init},
        {"native_setup",        "(Ljava/lang/Object;)V",            (void *)android_media_MediaPlayer_native_setup},
        ...
    };

    static int register_android_media_MediaPlayer(JNIEnv *env)
    {
        return AndroidRuntime::registerNativeMethods(env,
                    "android/media/MediaPlayer", gMethods, NELEM(gMethods));
    }

    jint JNI_OnLoad(JavaVM* vm, void* reserved)
    {
        ...
        if (register_android_media_MediaPlayer(env) < 0) {
            ...
        }
        ...
    }

说明：该代码在frameworks/base/media/jni/android_media_MediaPlayer.cpp中。gMethods是JNI本地方法的注册表；在Dalvik虚拟机启动之后，会调用JNI_OnLoad()；进而将上面的方法注册到系统中。  
这里，我们只需要了解native_init()与android_media_MediaPlayer_native_init()对应，而native_setup()和android_media_MediaPlayer_native_setup()对应即可。




<a name="anchor2_3"></a>
## 2.3 native_init()的实现

由于native_init()与android_media_MediaPlayer_native_init()对应，下面就看看native_init()的实现。

    static void
    android_media_MediaPlayer_native_init(JNIEnv *env)
    {
        jclass clazz;

        clazz = env->FindClass("android/media/MediaPlayer");
        ..

        fields.context = env->GetFieldID(clazz, "mNativeContext", "I");
        ...
    }


说明：该代码在frameworks/base/media/jni/android_media_MediaPlayer.cpp中。  
env->FindClass("android/media/MediaPlayer")会加载Java层的MediaPlayer类；进而将fields.context初始化为MediaPlayer类中的mNativeContext成员。



<a name="anchor2_4"></a>
## 2.4 native_setup()的实现

由于native_setup()和android_media_MediaPlayer_native_setup()对应，下面就看看native_setup()的实现。

    static void
    android_media_MediaPlayer_native_setup(JNIEnv *env, jobject thiz, jobject weak_this)
    {
        sp<MediaPlayer> mp = new MediaPlayer();
        ...

        setMediaPlayer(env, thiz, mp);
    }

说明：该函数会新建一个MediaPlayer对象，然后调用setMediaPlayer()来保存该MediaPlayer对象。下面看看setMediaPlayer()是如何保存MediaPlayer对象的。



<a name="anchor2_5"></a>
## 2.5 setMediaPlayer()

    static sp<MediaPlayer> setMediaPlayer(JNIEnv* env, jobject thiz, const sp<MediaPlayer>& player)
    {
        sp<MediaPlayer> old = (MediaPlayer*)env->GetIntField(thiz, fields.context);
        ...
        env->SetIntField(thiz, fields.context, (int)player.get());
        return old;
    }

说明：通过SetIntField()会将MediaPlayer对象保存到fields.context中。而在前面的，我们将fields.context初始化为MediaPlayer类(Java层)中的mNativeContext成员。这也就意味着，设置了Java层的MediaPlayer中的mNativeContext成员的值为C++层MediaPlayer对象。



<br/>
至此，就分析完了MediaPlayer的构造函数。下面继续看Java示例代码中的mp.setDataSource("/sdcard/test.mp3")。


<a name="anchor2_6"></a>
## 2.6 setDataSource()

    public void setDataSource(String path)
            throws IOException, IllegalArgumentException, SecurityException, IllegalStateException {
        setDataSource(path, null, null);
    }

    private void setDataSource(String path, String[] keys, String[] values)
            throws IOException, IllegalArgumentException, SecurityException, IllegalStateException {

        ...
        final File file = new File(path);
        if (file.exists()) {
            FileInputStream is = new FileInputStream(file);
            FileDescriptor fd = is.getFD();
            setDataSource(fd);
            is.close();
        } else {
            ...
        }
        ...
    }

    public void setDataSource(FileDescriptor fd)
            throws IOException, IllegalArgumentException, IllegalStateException {
        setDataSource(fd, 0, 0x7ffffffffffffffL);
    }

    public void setDataSource(FileDescriptor fd, long offset, long length)
            throws IOException, IllegalArgumentException, IllegalStateException {
        ...
        _setDataSource(fd, offset, length);
    }

    private native void _setDataSource(FileDescriptor fd, long offset, long length)
            throws IOException, IllegalArgumentException, IllegalStateException;

说明：该代码在frameworks/base/media/java/android/media/MediaPlayer.java中。setDataSource()最终会调用到本地方法_setDataSource()。在前面的gMethods本地方法注册表中，将_setDataSource()和android_media_MediaPlayer_setDataSourceFD()匹配。下面，看看_setDataSource()的实现。


<a name="anchor2_7"></a>
## 2.7 android_media_MediaPlayer_setDataSourceFD()

    static void
    android_media_MediaPlayer_setDataSourceFD(JNIEnv *env, jobject thiz, jobject fileDescriptor, jlong offset, jlong length)
    {
        sp<MediaPlayer> mp = getMediaPlayer(env, thiz);
        ...

        int fd = jniGetFDFromFileDescriptor(env, fileDescriptor);
        ...
        process_media_player_call( env, thiz, mp->setDataSource(fd, offset, length), "java/io/IOException", "setDataSourceFD failed." );
    }

说明：该代码在frameworks/base/media/jni/android_media_MediaPlayer.cpp中。该函数会先通过getMediaPlayer()获取MediaPlayer对象，然后在执行mp->setDataSource()时会调用MediaPlayer的setDataSource()方法。


<a name="anchor2_8"></a>
## 2.8 getMediaPlayer()

    static sp<MediaPlayer> getMediaPlayer(JNIEnv* env, jobject thiz)
    {
        ...
        MediaPlayer* const p = (MediaPlayer*)env->GetIntField(thiz, fields.context);
        return sp<MediaPlayer>(p);
    }

说明：前面在native_setup()中，将fields.context设置为MediaPlayer对象。这里就是返回fields.context中保存的MediaPlayer对象。


<a name="anchor2_9"></a>
## 2.9 MediaPlayer::setDataSource

    status_t MediaPlayer::setDataSource(
            const char *url, const KeyedVector<String8, String8> *headers)
    {
        ...
        status_t err = BAD_VALUE;
        if (url != NULL) {
            // 通过getMediaPlayerService()的代理BpMediaPlayerService。
            const sp<IMediaPlayerService>& service(getMediaPlayerService());
            if (service != 0) {
                // 通过BpMediaPlayerService创建一个IMediaPlayer客户端
                sp<IMediaPlayer> player(service->create(this, mAudioSessionId));
                ...
                // 保存player
                err = attachNewPlayer(player);
            }
        }
        return err;
    }

说明：该代码在frameworks/av/media/libmedia/mediaplayer.cpp中。  
(01) 它会新建一个service对象，而service是通过getMediaPlayerService()获取到的。getMediaPlayerService()已经在"[介绍getService请求][link_binder_07_getService01]"时，详细分析过了。它会返回IMediaPlayerService的代理，即BpMediaPlayerService对象。  
(02) 接着，会调用service->create()返回一个IMediaPlayer对象。下面看看这个MediaPlayer进程是如何通过BpMediaPlayerService这个远程代理来获取IMediaPlayer对象的。


<a name="anchor2_10"></a>
## 2.10 BpMediaPlayerService::create()


    class BpMediaPlayerService: public BpInterface<IMediaPlayerService>
    {
        ...
        virtual sp<IMediaPlayer> create(
                const sp<IMediaPlayerClient>& client, int audioSessionId) {
            Parcel data, reply;
            data.writeInterfaceToken(IMediaPlayerService::getInterfaceDescriptor());
            data.writeStrongBinder(client->asBinder());
            data.writeInt32(audioSessionId);

            remote()->transact(CREATE, data, &reply);
            return interface_cast<IMediaPlayer>(reply.readStrongBinder());
        }
        ...
    }

说明：该代码在frameworks/av/media/libmedia/IMediaPlayerService.cpp中。  
这里无非是CREATE请求数据打包之后发送给Binder驱动，再由Binder驱动转发给MediaPlayerService进程。数据的发送和解析，在前面介绍"addService"和"getService"时已经多次介绍过了；这里就不再展开说明了。

Binder驱动在收到MediaPlayer的数据之后，会将添加一个事务到MediaPlayerService的待处理事务列表中，然后唤醒MediaPlayerService。下面就从MediaPlayerService被唤醒之后开始说明。



<a name="anchor2_11"></a>
## 2.11 Binder驱动中binder_thread_read()的源码

又回到了熟悉的binder_thread_read()中。

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
            // 这里，MediaPlayer的CREATE请求的目标是MediaPlayerService，因此target_node是MediaPlayerService对应的节点；
            if (t->buffer->target_node) {
                // 事务目标对应的Binder实体(即，MediaPlayerService对应的Binder实体)
                struct binder_node *target_node = t->buffer->target_node;
                // Binder实体在用户空间的地址。
                // MediaPlayerService的ptr为本地Binder的弱引用，即BBinder的弱引用
                tr.target.ptr = target_node->ptr;
                // Binder实体在用户空间的其它数据
                // MediaPlayerService的cookie为本地Binder本身，即BBinder对象
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
            // 交易码，即CREATE
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

            // 将cmd(即BR_TRANSACTION)指令写入到ptr，即传递到用户空间
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

说明：MediaPlayerService被唤醒之后，binder_has_thread_work()为true。因为MediaPlayerService的待处理事务队列中有个待处理事务(即，MediaPlayer添加的CREATE请求)。  
(01) 进入while循环后，首先取出待处理事务。  
(02) 事务的类型是BINDER_WORK_TRANSACTION，得到对应的binder_transaction*类型指针t之后，跳出switch语句。很显然，此时t不为NULL，因此继续往下执行。下面的工作的目的，是将t中的数据转移到tr中(tr是事务交互数据包结构体binder_transaction_data对应的指针)，然后将指令和tr数据都拷贝到用户空间，让MediaPlayerService读取后进行处理。  

这里，共添加了两个指令到bwr.read_consumed中：BR_NOOP和BR_TRANSACTION。其中，BR_TRANSACTION指令对应的数据中包含了CREATE请求的数据。

接下来，binder_thread_read()返回到binder_ioctl()中；binder_ioctl()将数据拷贝到用户空间之后，便返回到用户空间继续执行。  
而在[Android Binder机制(八) MediaPlayerService服务的消息循环][link_binder_06_threadpool]中介绍过，MediaPlayerService是通过调用IPCThreadState::joinThreadPool()进入消息循环的，而joinThreadPool()又会通过getAndExecuteCommand()调用到talkWithDriver()来和Binder驱动交互的。因此，Binder驱动返回到用户空间之后，会进入talkWithDriver()。


<a name="anchor2_12"></a>
## 2.12 IPCThreadState::talkWithDriver()

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

说明：ioctl()返回之后，talkWithDriver()会清除已经发送的数据。然后，便返回到getAndExecuteCommand()中。

<a name="anchor2_13"></a>
## 13. IPCThreadState::getAndExecuteCommand()

    status_t IPCThreadState::getAndExecuteCommand()
    {
        status_t result;
        int32_t cmd;

        // 和Binder驱动交互
        result = talkWithDriver();
        if (result >= NO_ERROR) {
            ...
            // 读取mIn中的数据
            cmd = mIn.readInt32();
            ...

            // 调用executeCommand()对数据进行处理。
            result = executeCommand(cmd);
            ...
        }

        return result;
    }

说明：getAndExecuteCommand()会取出Binder反馈的指令，然后再调用executeCommand()根据指令进行解析。前面说过，Binder驱动共反馈了BR_NOOP和BR_TRANSACTION两个指令。而BR_NOOP指令什么也不会做。因此，我们直接分析BR_TRANSACTION指令。


<a name="anchor2_14"></a>
## 14. IPCThreadState::executeCommand()

    status_t IPCThreadState::executeCommand(int32_t cmd) 
    {
        BBinder* obj; 
        RefBase::weakref_type* refs;
        status_t result = NO_ERROR;
        
        switch (cmd) {
            ...
            case BR_TRANSACTION:
            {
                binder_transaction_data tr;
                result = mIn.read(&tr, sizeof(tr));
                ...

                Parcel buffer;
                buffer.ipcSetDataReference(
                    reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                    tr.data_size,
                    reinterpret_cast<const size_t*>(tr.data.ptr.offsets),
                    tr.offsets_size/sizeof(size_t), freeBuffer, this);

                ...

                Parcel reply;
                ...
                if (tr.target.ptr) {
                    sp<BBinder> b((BBinder*)tr.cookie);
                    const status_t error = b->transact(tr.code, buffer, &reply, tr.flags);
                    if (error < NO_ERROR) reply.setError(error);

                } else {
                    ...
                }

                if ((tr.flags & TF_ONE_WAY) == 0) {
                    sendReply(reply, 0);
                } else {
                    ...
                }
                ...

            }
            break;

            ...
        }

        ...
        return result;
    }

说明：进入BR_TRANSACTION分支后，首先通过mIn.read()读取事务数据。接着，调用ipcSetDataReference()将事务数据解析出来。很显然，tr.target.ptr不为空，它的值是"MediaPlayerService的BBinder的弱引用"。然后，就将tr.cookie转换为BBinder*对象b；而b实际上是MediaPlayerService的本地Binder实例，即BnMediaPlayerService的实例。最终，通过b->transact()进行事务处理。

下面看看BBinder的transact()代码。


    status_t BBinder::transact(
        uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
    {
        data.setDataPosition(0);

        status_t err = NO_ERROR;
        switch (code) {
            case PING_TRANSACTION:
                reply->writeInt32(pingBinder());
                break;
            default:
                err = onTransact(code, data, reply, flags);
                break;
        }

        if (reply != NULL) {
            reply->setDataPosition(0);
        }

        return err;
    }

该代码在frameworks/native/libs/binder/Binder.cpp中。此时的code是CREATE，因此，它会调用onTransact()对事务进行处理。而BnMediaPlayerService重写了onTransact()方法；因此会调用到BnMediaPlayerService的onTransact()方法。在Binder机制中也是根据这种方式来实现不同Server的对各自的的请求进行区分处理的：Server的本地Binder实现类，通过覆盖onTransact()方法来处理事务。

下面看看BnMediaPlayerService的onTransact()方法。


<a name="anchor2_15"></a>
## 15. BnMediaPlayerService::onTransact()

    status_t BnMediaPlayerService::onTransact(
        uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
    {
        switch (code) {
            case CREATE: {
                ...
                sp<IMediaPlayerClient> client =
                    interface_cast<IMediaPlayerClient>(data.readStrongBinder());
                int audioSessionId = data.readInt32();
                sp<IMediaPlayer> player = create(client, audioSessionId);
                reply->writeStrongBinder(player->asBinder());
                return NO_ERROR;
            } break;
            ...
        }
        ...
    }

说明：该代码在frameworks/av/media/libmedia/IMediaPlayerService.cpp中。  
(01) 先通过interface_cast宏获取IMediaPlayerClient对象，该对象是BpMediaPlayerClient实例。BpMediaPlayerClient定义在frameworks/av/media/libmedia/IMediaPlayer.cpp中。  
(02) 接着，通过create()创建IMediaPlayerService对象。该create()的实现是在BnMediaPlayerService的子类MediaPlayerService.cpp中，即在frameworks/av/media/libmediaplayerservice/MediaPlayerService.cpp中实现。在create()中会新建一个Client，并返回。  
(03) 最后，将这个Client通过Binder返回给MediaPlayer。

后面的流程就不再多说了。本文的核心是Binder机制，而不是MediaPlayer的框架，让我们了解MediaPlayer进程是如何与MediaPlayerService交互即可！而目前，通过CREATE请求，我们已经知道了MediaPlayer是如何和MediaPlayerService进行事务交互的。后面的内容更多的涉及到MediaPlayerService的框架，它不是本章的重点；感兴趣的读者可以自行研究。




[link_binder_01_introduce]: /2014/09/01/Binder-Introduce/
[link_binder_02_datastruct]: /2014/09/02/Binder-Datastruct/
[link_binder_03_ServiceManagerDeamon]: /2014/09/03/Binder-ServiceManager-Daemon/
[link_binder_04_defaultServiceManager]: /2014/09/04/Binder-defaultServiceManager/
[link_binder_05_addService01]: /2014/09/05/BinderCommunication-AddService01/
[link_binder_05_addService02]: /2014/09/05/BinderCommunication-AddService02/
[link_binder_05_addService03]: /2014/09/05/BinderCommunication-AddService03/
[link_binder_06_threadpool]: /2014/09/06/BinderCommunication-ThreadPool/
[link_binder_07_getService01]: /2014/09/07/BinderCommunication-GetService01/
