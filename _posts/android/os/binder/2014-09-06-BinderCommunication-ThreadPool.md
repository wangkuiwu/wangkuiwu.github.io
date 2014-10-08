---
layout: post
title: "Android Binder机制(八) MediaPlayerService服务的消息循环"
description: "android"
category: android
tags: [android]
date: 2014-09-06 09:01
---


> 在前面的3篇文章中，我们以MediaPlayerService为例，介绍了C-S中的Server服务是如何通过addService请求添加到ServiceManager中的。但是，在[Android Binder机制(七) addService详解03之 请求的反馈][link_binder_05_addService03]的结尾，我们提到过：MediaPlayerService仅仅只是将自己注册到了ServiceManager中，它还没有进入消息循环等待Client的请求。  
> 本文，就接着介绍MediaPlayerService是如何进入消息循环的。

> 注意：本文是基于Android 4.4.2版本进行介绍的！



<a name="anchor1"></a>
# 1. MediaPlayerService的main()函数

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

说明：该代码在frameworks/av/media/mediaserver/main_mediaserver.cpp中。对于MediaPlayerService::instantiate()，已经详细介绍过了；它的作用是将MediaPlayerService已经注册到ServiceManager中。下面看看startThreadPool()的流程。



<a name="anchor2"></a>
# 2. ProcessState::startThreadPool();

    void ProcessState::startThreadPool()
    {
        AutoMutex _l(mLock);
        if (!mThreadPoolStarted) {
            mThreadPoolStarted = true;
            spawnPooledThread(true);
        }
    }

说明：该代码定义在frameworks/native/libs/binder/ProcessState.cpp中。  
mThreadPoolStarted的初始值为false，因此这里设置mThreadPoolStarted=true之后，就调用spawnPooledThread()。



<a name="anchor2"></a>
# 2. ProcessState::spawnPooledThread()

    void ProcessState::spawnPooledThread(bool isMain)
    {
        if (mThreadPoolStarted) {
            String8 name = makeBinderThreadName();
            ALOGV("Spawning new pooled thread, name=%s\n", name.string());
            sp<Thread> t = new PoolThread(isMain);
            t->run(name.string());
        }
    }

说明：该代码定义在frameworks/native/libs/binder/ProcessState.cpp中。 此时mThreadPoolStarted=true，因此会先调用makeBinderThreadName()为线程取一个名称；然后新建PoolThread线程，并运行。  
makeBinderThreadName()的代码比较简单，这里就不列出了。线程的名称是"Binder_X"(其实X是16进制数)，每新建一个线程X的值都会+1。下面看看PoolThread。


<a name="anchor3"></a>
# 3. ProcessState::spawnPooledThread()

    class PoolThread : public Thread
    {
    public:
        PoolThread(bool isMain)
            : mIsMain(isMain)
        {
        }

    protected:
        virtual bool threadLoop()
        {
            IPCThreadState::self()->joinThreadPool(mIsMain);
            return false;
        }

        const bool mIsMain;
    };


说明：该代码定义在frameworks/native/libs/binder/ProcessState.cpp中。PoolThread继承于Thread，在线程启动之后，会调用threadLoop()进入消息循环中。 

下面简单说说，当PoolThread启动之后，是如何调用到threadLoop()的。PoolThread继承于Thread，先看看Thread的构造函数，然后再看看run()的代码。


<a name="anchor4"></a>
# 4. Thread::Thread

    Thread::Thread(bool canCallJava)
        :   mCanCallJava(canCallJava),
            mThread(thread_id_t(-1)),
            mLock("Thread::mLock"),
            mStatus(NO_ERROR),
            mExitPending(false), mRunning(false)
            , mTid(-1)
    {
    }   

说明：该代码定义在system/core/libutils/Threads.cpp中。新建Thread对象时，会进行一些列初始化。这里设置mCanCallJava=true。


<a name="anchor5"></a>
# 5. Thread::run

    status_t Thread::run(const char* name, int32_t priority, size_t stack)
    {   
        Mutex::Autolock _l(mLock);

        ...

        // 初始化
        mStatus = NO_ERROR;
        mExitPending = false;
        mThread = thread_id_t(-1);
        mHoldSelf = this;
        mRunning = true;            
                                    
        bool res;
        if (mCanCallJava) {
            res = createThreadEtc(_threadLoop,
                    this, name, priority, stack, &mThread);
        } else {
            ...
        }
        
        ...
        return NO_ERROR;
    }

说明：该代码定义在system/core/libutils/Threads.cpp中。  
(01) 先看看函数参数。name是spawnPooledThread()中创建的Binder线程名称，形式是"Binder_X"。priority是优先级(默认值为PRIORITY_DEFAULT)，stack是线程栈数量(默认是0)；它们都是使用默认值，在system/core/include/utils/Thread.h中定义。  
(02) 先进行初始化；mCanCallJava的值在构造函数中被初始化为true。因此，会调用createThreadEtc()。


<a name="anchor6"></a>
# 6. createThreadEtc

    // Create thread with lots of parameters
    inline bool createThreadEtc(thread_func_t entryFunction,
                                void *userData,
                                const char* threadName = "android:unnamed_thread",             
                                int32_t threadPriority = PRIORITY_DEFAULT,                     
                                size_t threadStackSize = 0,                                    
                                thread_id_t *threadId = 0)
    {
        return androidCreateThreadEtc(entryFunction, userData, threadName,
            threadPriority, threadStackSize, threadId) ? true : false;
    }

说明：该代码定义在system/core/include/utils/AndroidThreads.h中。它会调用androidCreateThreadEtc()。


<a name="anchor7"></a>
# 7. androidCreateThreadEtc

    static android_create_thread_fn gCreateThreadFn = androidCreateRawThreadEtc;

    int androidCreateThreadEtc(android_thread_func_t entryFunction,
                                void *userData,
                                const char* threadName,
                                int32_t threadPriority,
                                size_t threadStackSize,
                                android_thread_id_t *threadId)
    {
        return gCreateThreadFn(entryFunction, userData, threadName,
            threadPriority, threadStackSize, threadId);
    }

    void androidSetCreateThreadFunc(android_create_thread_fn func)
    {
        gCreateThreadFn = func;
    }

说明：该代码定义在system/core/libutils/Threads.cpp中。 androidCreateThreadEtc()会调用gCreateThreadFn()。gCreateThreadFn()是个函数指针，它的值是androidCreateRawThreadEtc。


<a name="anchor8"></a>
# 8. androidCreateRawThreadEtc

    int androidCreateRawThreadEtc(android_thread_func_t entryFunction,
                                   void *userData,
                                   const char* threadName,
                                   int32_t threadPriority,
                                   size_t threadStackSize,
                                   android_thread_id_t *threadId)
    {
        ...
        int result = pthread_create(&thread, &attr,
                        (android_pthread_entry)entryFunction, userData);

        ...
        return 1;
    }

说明：该代码定义在system/core/libutils/Threads.cpp中。 该函数会调用pthread_create()，而pthread_create()则是我们非常熟悉的Linux的标准接口，它的作用就是创建线程。线程创建成功之后运行时，会以执行entryFunction对应的函数。而entryFunction这个函数指针的值是_threadLoop。因此，当线程启动之后，会执行_threadLoop。


<a name="anchor9"></a>
# 9. _threadLoop

    int Thread::_threadLoop(void* user)
    {
        ...

        bool first = true;

        do {
            bool result;
            if (first) {
                first = false;
                self->mStatus = self->readyToRun();
                result = (self->mStatus == NO_ERROR);

                if (result && !self->exitPending()) {
                    result = self->threadLoop();
                }
            } else {
                ...
            }

            ...
        } while(strong != 0);

        return 0;
    }

说明：该代码定义在system/core/libutils/Threads.cpp中。first的初始值为true，因此进入到if(first)中。  readyToRun()的实现在Threads.cpp中，返回NO_ERROR。因此result为true，而mExitPending的默认值为false，即self0>exitPending()返回false。因此会执行self->threadLoop()。由于PoolThread重载了threadLoop()，因此，这里的self->threadLoop()会调用PoolThread中的threadLoop()。


<a name="anchor10"></a>
# 10. PoolThread::threadLoop()

        virtual bool threadLoop()
        {
            IPCThreadState::self()->joinThreadPool(mIsMain);
            return false;
        }

说明：这是PoolThread中实现的threadLoop()函数。它会先通过IPCThreadState::self()获取IPCThreadState对象，然后调用IPCThreadState::joinThreadPool(mIsMain)，其中mIsMain为true。


<a name="anchor11"></a>
# 11. IPCThreadState::joinThreadPool()

    void IPCThreadState::joinThreadPool(bool isMain)
    {
        ...

        mOut.writeInt32(isMain ? BC_ENTER_LOOPER : BC_REGISTER_LOOPER);

        ...
        do {  
            ...
            result = getAndExecuteCommand();

            if (result < NO_ERROR && result != TIMED_OUT && result != -ECONNREFUSED && result != -EBADF) {
                ...
                abort();
            } 
              
            if(result == TIMED_OUT && !isMain) {
                break;
            } 
        } while (result != -ECONNREFUSED && result != -EBADF);

        ...

        mOut.writeInt32(BC_EXIT_LOOPER);    
        talkWithDriver(false);              
    }

说明：该代码定义在frameworks/native/libs/binder/IPCThreadState.cpp中。在该函数中，便进入了消息循环！  
(01) isMain=true，因此会先将BC_ENTER_LOOPER指令写入到mOut中。  
(02) 接着调用getAndExecuteCommand()。  


<a name="anchor12"></a>
# 12. IPCThreadState::getAndExecuteCommand()

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

说明：该函数会调用talkWithDriver()和Binder驱动进行交互。对于talkWithDriver()，前面已经多次提到。在此，talkWithDriver()会将BC_ENTER_LOOPER指令发送给Binder驱动，告诉Binder驱动，MediaPlayerService进入了消息循环状态。BC_ENTER_LOOPER的流程在[Android Binder机制(三) ServiceManager守护进程][link_binder_03_ServiceManagerDeamon]中已经介绍过了。  
当BC_ENTER_LOOPER处理完毕，MediaPlayerService再次调用ioctl()和Binder驱动通信时，由于MediaPlayerService对应的待处理事务列表为空，因此MediaPlayerService线程会进入中断等待状态。当有Client向MediaPlayerService发送请求时，MediaPlayerService就会被唤醒。




[link_binder_01_introduce]: /2014/09/01/Binder-Introduce/
[link_binder_02_datastruct]: /2014/09/02/Binder-Datastruct/
[link_binder_03_ServiceManagerDeamon]: /2014/09/03/Binder-ServiceManager-Daemon/
[link_binder_04_defaultServiceManager]: /2014/09/04/Binder-defaultServiceManager/
[link_binder_05_addService01]: /2014/09/05/BinderCommunication-AddService01/
[link_binder_05_addService02]: /2014/09/05/BinderCommunication-AddService02/
[link_binder_05_addService03]: /2014/09/05/BinderCommunication-AddService03/
