---
layout: post
title: "Android UI系统(四) BootAnimation流程一之创建SurfaceFlinger客户端"
description: "android"
category: android
tags: [android]
date: 2014-09-10 09:01
---


> 这是关于Android UI系统中的Gralloc。Android设备都大多有一个显示屏，这个显示屏通过驱动映射到文件节点/dev/graphics/fb0上。通过操作/dev/graphics/fb0就能更新显示内容，

> 注意：本文是基于Android 4.4.2版本进行介绍的！

> **目录**  
> **1**. [Binder架构解析](#anchor1)  



<a name="anchor1"></a>
# 1. BootAnimation概述


<a name="anchor2"></a>
# 2. BootAnimation流程

<a name="anchor2_1"></a>
## 2.1 BootAnimation入口


    int main(int argc, char** argv)
    {
        // 设置优先级
        setpriority(PRIO_PROCESS, 0, ANDROID_PRIORITY_DISPLAY);

        char value[PROPERTY_VALUE_MAX];
        property_get("debug.sf.nobootanimation", value, "0");
        int noBootAnimation = atoi(value);
        ALOGI_IF(noBootAnimation,  "boot animation disabled");
        if (!noBootAnimation) {

            sp<ProcessState> proc(ProcessState::self());
            // 启动线程池，进入消息循环
            ProcessState::self()->startThreadPool();

            // 启动BootAnimation
            sp<BootAnimation> boot = new BootAnimation();

            IPCThreadState::self()->joinThreadPool();

        }   
        return 0;
    }
    
说明：该代码在frameworks/base/cmds/bootanimation/bootanimation_main.cpp中。 它是BootAnimation的入口程序，目的是启动BootAnimation动画。  
(01) 首先，它会获取全局的ProcessState对象，然后通过调用startThreadPool()创建线程池。因为，BootAnimation需要和SurfaceFlinger仅次能够IPC通信，所以才需要通过ProcessState启动线程池，来通过Binder和SurfaceFlinger通信。  
(02) 然后，它会新建BootAnimation对象。即，启动BootAnimation动画。  
(03) 最后，进入消息循环。

接下来，重点介绍BootAnimation的相关内如；至于ProcessState和IPCThreadState，在介绍"Binder机制"时，已经详细介绍过它们。这里就不再重复说明，若对Binder不熟悉，建议先了解[Binder机制][link_binder_01_introduce]。


<a name="anchor2_2"></a>
## 2.2 BootAnimation构造函数

    BootAnimation::BootAnimation() : Thread(false)
    {
        mSession = new SurfaceComposerClient();
    }

说明：该代码在frameworks/base/cmds/bootanimation/BootAnimation.cpp中。BootAnimation继承于Thread类，它的构造函数中会新建SurfaceComposerClient对象。


<a name="anchor2_3"></a>
## 2.3 SurfaceComposerClient构造函数

    SurfaceComposerClient::SurfaceComposerClient()
        : mStatus(NO_INIT), mComposer(Composer::getInstance())
    {
    }

说明：该代码在frameworks/native/libs/gui/SurfaceComposerClient.cpp中。它会新建一个Composer对象，而Composer是SurfaceComposerClient.cpp中的内部类，我们不需要特别的关注Composer的类结构。


<a name="anchor2_4"></a>
## 2.4 SurfaceComposerClient::onFirstRef

由于SurfaceComposerClient继承于RefBase。而对于RefBase类型的类，当创建它的实例并且该实例是强引用类型时，会调用该类的onFirstRef()接口。而前面在BootAnimation的构造函数中创建SurfaceComposerClient对象时，就是一个强引用类型。下面就看看SurfaceComposerClient的onFirstRef()代码。

    void SurfaceComposerClient::onFirstRef() {
        sp<ISurfaceComposer> sm(ComposerService::getComposerService());
        if (sm != 0) {
            sp<ISurfaceComposerClient> conn = sm->createConnection();
            if (conn != 0) {
                mClient = conn;
                mStatus = NO_ERROR;
            }
        }
    }

说明：该函数会先通过getComposerService()获取一个ISurfaceComposer对象，该对象实际上就是SurfaceFlinger服务。然后再调用createConnection()建立与SurfaceFlinger之间的连接，而这个所谓的连接，实际上就是通过SurfaceFlinger服务返回一个客户端ISurfaceComposerClient对象。


<a name="anchor2_5"></a>
## 2.5 ComposerService::getComposerService()

    class ComposerService : public Singleton<ComposerService>
    {
        sp<ISurfaceComposer> mComposerService;
        sp<IBinder::DeathRecipient> mDeathObserver;
        Mutex mLock;

        ComposerService();                
        void connectLocked();             
        void composerServiceDied();       
        friend class Singleton<ComposerService>;
    public:   

        static sp<ISurfaceComposer> getComposerService();
    };


说明：该代码在frameworks/native/include/private/gui/ComposerService.h中。它是ComposerService的头文件，而ComposerService的实现在SurfaceComposerClient.cpp中。从头文件可以看书，getComposerService()是个全局的静态方法。

    sp<ISurfaceComposer> ComposerService::getComposerService() {
        ComposerService& instance = ComposerService::getInstance();
        Mutex::Autolock _l(instance.mLock);
        if (instance.mComposerService == NULL) {
            ComposerService::getInstance().connectLocked();
            assert(instance.mComposerService != NULL);
            ALOGD("ComposerService reconnected"); 
        }
        return instance.mComposerService;
    }   

说明：该代码定义在frameworks/native/libs/gui/SurfaceComposerClient.cpp中。它的作用是获取ISurfaceComposer对象，实际上就是SurfaceFlinger服务。该函数会获取ComposerService的实例。ComposerService继承于Singleton，也就是说它是单例模式实现的。下面看看它的构造函数。


<a name="anchor2_6"></a>
## 2.6 ComposerService::ComposerService()

    ComposerService::ComposerService()
    : Singleton<ComposerService>() {
        Mutex::Autolock _l(mLock);
        connectLocked();
    }

说明：ComposerService的构造函数中会调用connectLocked()。


<a name="anchor2_7"></a>
## 2.7 ComposerService::connectLocked() 

    void ComposerService::connectLocked() {
        const String16 name("SurfaceFlinger");
        while (getService(name, &mComposerService) != NO_ERROR) {
            usleep(250000);
        }
        assert(mComposerService != NULL);

        // Create the death listener.
        class DeathObserver : public IBinder::DeathRecipient {
            ComposerService& mComposerService;
            virtual void binderDied(const wp<IBinder>& who) {
                ALOGW("ComposerService remote (surfaceflinger) died [%p]",
                      who.unsafe_get());
                mComposerService.composerServiceDied();
            }
         public:
            DeathObserver(ComposerService& mgr) : mComposerService(mgr) { }
        };

        mDeathObserver = new DeathObserver(*const_cast<ComposerService*>(this));
        mComposerService->asBinder()->linkToDeath(mDeathObserver);
    }

说明：该函数会调用getService()获取SurfaceFlinger服务，并将服务保存到mComposerService对象中。关于getService()是如何获取到名称为"SurfaceFlinger"的流程就不详细介绍了，可以通过"[Android Binder机制(九) getService详解][link_binder_07_getService01]"了解getService()的流程。

[skywang-todo]

上面是SurfaceFlinger的类图。从图中可知，getService()获取到的mComposerService是sp<ISurfaceComposer>类型的，它实际上是SurfaceFlinger的代理类BpSurfaceComposer的实例。


<br/>
我们回到SurfaceComposerClient的onFirstRef()中，它在通过getComposerService获取到ISurfaceComposer对象之后；它会再次调用该对象的createConnection()方法。下面就看看BpSurfaceComposer()的createConnection()方法。


<a name="anchor2_8"></a>
## 2.8 BpSurfaceComposer::createConnection

    class BpSurfaceComposer : public BpInterface<ISurfaceComposer>
    {   
        ...

        virtual sp<ISurfaceComposerClient> createConnection()
        {   
            uint32_t n;
            Parcel data, reply;
            data.writeInterfaceToken(ISurfaceComposer::getInterfaceDescriptor());                   
            remote()->transact(BnSurfaceComposer::CREATE_CONNECTION, data, &reply);
            return interface_cast<ISurfaceComposerClient>(reply.readStrongBinder());
        }

        ...
    }

说明：该代码在frameworks/native/libs/gui/ISurfaceComposer.cpp中。BpSurfaceComposer是SurfaceFlinger的远程代理，而createConnection()的作用就是通过Binder向SurfaceFlinger本地服务发送CREATE_CONNECTION请求。而该请求的返回结果是ISurfaceComposerClient对象。

Binder传递CREATE_CONNECTION请求的详细流程，请参考[Android Binder机制(五) addService详解][link_binder_05_addService01]。这里直接介绍SurfaceFlinger本地服务收到CREATE_CONNECTION的处理过程。Binder在将请求转发给本地服务时，会先找到本地服务对象，然后调用本地服务的onTransact()来处理请求。下面就看看SurfaceFlinger的onTransact()中处理CREATE_CONNECTION的流程。


<a name="anchor2_9"></a>
## 2.9 SurfaceFlinger::onTransact()

    status_t SurfaceFlinger::onTransact(
        uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
    {
        switch (code) {
            ...

            case CREATE_CONNECTION:
            case CREATE_DISPLAY:
            case SET_TRANSACTION_STATE:
            case BOOT_FINISHED:
            case BLANK:
            case UNBLANK:
            {
                // 权限检测
                IPCThreadState* ipc = IPCThreadState::self();
                const int pid = ipc->getCallingPid();
                const int uid = ipc->getCallingUid();
                if ((uid != AID_GRAPHICS) &&
                        !PermissionCache::checkPermission(sAccessSurfaceFlinger, pid, uid)) {
                    ALOGE("Permission Denial: "
                            "can't access SurfaceFlinger pid=%d, uid=%d", pid, uid);
                    return PERMISSION_DENIED;
                }
                break;
            }
            ...
        }

        status_t err = BnSurfaceComposer::onTransact(code, data, reply, flags);
        if (err == UNKNOWN_TRANSACTION || err == PERMISSION_DENIED) {
            ...
        }
    }

说明：该代码在frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp中。SurfaceFlinger继承于BnSurfaceComposer。在处理CREATE_CONNECTION请求时，它会先进行权限检查，然后调用BnSurfaceComposer()来处理请求。



<a name="anchor2_10"></a>
## 2.10 BnSurfaceComposer::onTransact()

    status_t BnSurfaceComposer::onTransact(
        uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
    {       
        switch(code) {
            case CREATE_CONNECTION: {
                CHECK_INTERFACE(ISurfaceComposer, data, reply);
                sp<IBinder> b = createConnection()->asBinder();
                reply->writeStrongBinder(b);
                return NO_ERROR;
            }
            ...
        }
    }

说明：该代码在frameworks/native/libs/gui/ISurfaceComposer.cpp中。在处理CREATE_CONNECTION请求时，BnSurfaceComposer通过createConnection()创建一个SurfaceFlinger的客户端对象，并获取该客户端对象的IBinder对象。然后通过writeStrongBinder()将该服务保存到reply中，进而返回给请求程序。下面看看createConnection()的代码。



<a name="anchor2_11"></a>
## 2.11 SurfaceFlinger::createConnection
    
    sp<ISurfaceComposerClient> SurfaceFlinger::createConnection()
    {
        sp<ISurfaceComposerClient> bclient;
        sp<Client> client(new Client(this));
        status_t err = client->initCheck();
        if (err == NO_ERROR) {
            bclient = client;
        }           
        return bclient;
    }

说明：该代码在frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp中。它会新建一个Client对象，并返回该Client对象。这个Client就是SurfaceFlinger服务对应的客户端。Client继承于BnSurfaceComposerClient，也就是说Client是ISurfaceComposerClient的本地客户端，它对应的远程代理是BpSurfaceComposerClient。

[skywang-todo]

上面是ISurfaceComposerClient客户端的类图。ISurfaceComposerClient是SurfaceFlinger服务的客户端接口，它对应的本地客户端是Client，而远程客户端是BpSurfaceComposerClient。Client的构造函数没有执行什么特殊的动作，initCheck()也单单只返回了NO_ERROR。

    Client::Client(const sp<SurfaceFlinger>& flinger)
        : mFlinger(flinger)
    {
    }

    status_t Client::initCheck() const {
        return NO_ERROR;
    }

<br/>
现在回到BnSurfaceComposer::onTransact()中，它处理CREATE_CONNECTION请求时，先是通过createConnection()获取了一个客户端Client对象；然后，通过asBinder()获取该对象的IBinder对象；最后，通过writeStrongBinder()将IBinder对象写入到reply中，并返回给请求程序。

接着，回到BpSurfaceComposer的createConnection()中，在通过transact()执行了CREATE_CONNECTION请求之后，它会通过readStrongBinder()读取请求回复，而该回复就是前面BnSurfaceComposer中createConnection()创建的客户读对象(即，Client对象)。

    class BpSurfaceComposer : public BpInterface<ISurfaceComposer>
    {   
        ...

        virtual sp<ISurfaceComposerClient> createConnection()
        {   
            uint32_t n;
            Parcel data, reply;
            data.writeInterfaceToken(ISurfaceComposer::getInterfaceDescriptor());                   
            remote()->transact(BnSurfaceComposer::CREATE_CONNECTION, data, &reply);
            return interface_cast<ISurfaceComposerClient>(reply.readStrongBinder());
        }

        ...
    }


在获取到Client对象之后，再通过interface_cast<ISurfaceComposerClient>(new Client())将其转换为ISurfaceComposerClient对象。关于interface_cast的原理已经在[Android Binder机制(四) defaultServiceManager()的实现][link_binder_04_defaultServiceManager]中详细介绍过了，它会通过asInterface()获取到ISurfaceComposerClient对象。下面看看interface_cast<ISurfaceComposerClient>展开后的asInterface()函数。


    android::sp<ISurfaceComposerClient> ISurfaceComposerClient::asInterface(const android::sp<android::IBinder>& obj)
    {
        android::sp<ISurfaceComposerClient> intr;
        if (obj != NULL) {
            intr = static_cast<ISurfaceComposerClient*>(
                obj->queryLocalInterface(
                        ISurfaceComposerClient::descriptor).get());
            if (intr == NULL) {
                intr = new BpSurfaceComposerClient(obj);
            }
        }
        return intr;
    }

asInterface()最终会返回BpSurfaceComposerClient对象。实际上，BpSurfaceComposerClient就是Client的远程代理。再次回到SurfaceComposerClient::onFirstRef()中：  
(01) 它先是通过getComposerService()获取到ISurfaceComposer对象，实际上就是获取SurfaceFlinger服务的远程代理。  
(02) 接着，它会调用createConnection()。该函数会通过SurfaceFlinger的远程代理向SurfaceFlinger服务本身发送"创建客户端的请求"。SurfaceFlinger收到请求之后，会新建Client客户端，并将该客户端的远程代理返回给SurfaceComposerClient，也就是将客户端的远程代理对象赋值给conn。  
(03) 最后，因为conn不为零，SurfaceComposerClient会将客户端的远程代理赋值给mClient。

    void SurfaceComposerClient::onFirstRef() {
        sp<ISurfaceComposer> sm(ComposerService::getComposerService());
        if (sm != 0) {
            sp<ISurfaceComposerClient> conn = sm->createConnection();
            if (conn != 0) {
                mClient = conn;
                mStatus = NO_ERROR;
            }
        }
    }


<br/>
至此，在BootAnimation中创建SurfaceComposerClient的强引用对象就介绍完了。在该过程中，先是调用了SurfaceComposerClient的构造函数；由于，SurfaceComposerClient继承于RefBase，并且BootAnimation中创建的是SurfaceComposerClient的强引用，因此会执行onFirstRef()函数。  
在执行onFirstRef()时，SurfaceComposerClient会获取SurfaceFlinger服务的远程代理，然后通过该远程代理给SurfaceFlinger发送"创建客户端的请求"。SurfaceFlinger收到请求之后，就会新建Client客户端，并将客户端的远程代理返回给SurfaceComposerClient。接着，SurfaceComposerClient会将该客户端的远程代理保存到mClient中。  
这样，BootAnimation就相当于打通了和SurfaceFlinger服务之间的连接，它自身就相当于一个SurfaceFlinger客户端。接着，BootAnimation就可以通过SurfaceFlinger来进行动画渲染了。

    BootAnimation::BootAnimation() : Thread(false)
    {
        mSession = new SurfaceComposerClient();
    }




[link_binder_01_introduce]: /2014/09/01/Binder-Introduce/
[link_binder_04_defaultServiceManager]: /2014/09/04/Binder-defaultServiceManager/
[link_binder_05_addService01]: /2014/09/05/BinderCommunication-AddService01/
[link_binder_07_getService01]: /2014/09/07/BinderCommunication-GetService01/
