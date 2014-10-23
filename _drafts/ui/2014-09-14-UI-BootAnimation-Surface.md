---
layout: post
title: "Android UI系统(四) BootAnimation流程二之"
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
# 1. 概述


<a name="anchor2"></a>
# 2. 

在前面的文章中，我们介绍了BootAnimation的构造函数。而且我们了解到，BootAnimation是Thread的子类。

    class BootAnimation : public Thread, public IBinder::DeathRecipient
    {   
        ...
    }

在[Android Binder机制(八) MediaPlayerService服务的消息循环][link_binder_06_threadpool]中，我们介绍过Thread启动后的执行流程。它会先调用readyToRun()执行准备工作，然后调用threadLoop()进入消息循环。

下面，看看BootAnimation中readyToRun()的流程。


<a name="anchor2_1"></a>
## 2.1 BootAnimation::readyToRun()

    status_t BootAnimation::readyToRun() {
        mAssets.addDefaultAssets();
                
        sp<IBinder> dtoken(SurfaceComposerClient::getBuiltInDisplay(
                ISurfaceComposer::eDisplayIdMain));
        DisplayInfo dinfo;
        status_t status = SurfaceComposerClient::getDisplayInfo(dtoken, &dinfo);
        if (status)
            return -1;

        // create the native surface
        sp<SurfaceControl> control = session()->createSurface(String8("BootAnimation"),
                dinfo.w, dinfo.h, PIXEL_FORMAT_RGB_565);

        SurfaceComposerClient::openGlobalTransaction();
        control->setLayer(0x40000000);
        SurfaceComposerClient::closeGlobalTransaction();
                
        sp<Surface> s = control->getSurface();
        
        // initialize opengl and egl
        const EGLint attribs[] = {
                EGL_RED_SIZE,   8,
                EGL_GREEN_SIZE, 8,
                EGL_BLUE_SIZE,  8, 
                EGL_DEPTH_SIZE, 0,
                EGL_NONE
        };
        EGLint w, h, dummy;
        EGLint numConfigs;
        EGLConfig config;
        EGLSurface surface;
        EGLContext context;

        EGLDisplay display = eglGetDisplay(EGL_DEFAULT_DISPLAY);

        eglInitialize(display, 0, 0);
        eglChooseConfig(display, attribs, &config, 1, &numConfigs);
        surface = eglCreateWindowSurface(display, config, s.get(), NULL);
        context = eglCreateContext(display, config, NULL, NULL);
        eglQuerySurface(display, surface, EGL_WIDTH, &w);
        eglQuerySurface(display, surface, EGL_HEIGHT, &h);

        if (eglMakeCurrent(display, surface, surface, context) == EGL_FALSE)
            return NO_INIT;
        mDisplay = display;
        mContext = context;
        mSurface = surface;
        mWidth = w;
        mHeight = h;
        mFlingerSurfaceControl = control;
        mFlingerSurface = s;

        mAndroidAnimation = true;

        // If the device has encryption turned on or is in process 
        // of being encrypted we show the encrypted boot animation.
        char decrypt[PROPERTY_VALUE_MAX];
        property_get("vold.decrypt", decrypt, "");

        bool encryptedAnimation = atoi(decrypt) != 0 || !strcmp("trigger_restart_min_framework", decrypt);

        if ((encryptedAnimation &&
                (access(SYSTEM_ENCRYPTED_BOOTANIMATION_FILE, R_OK) == 0) &&
                (mZip.open(SYSTEM_ENCRYPTED_BOOTANIMATION_FILE) == NO_ERROR)) ||

                ((access(USER_BOOTANIMATION_FILE, R_OK) == 0) &&
                (mZip.open(USER_BOOTANIMATION_FILE) == NO_ERROR)) ||

                ((access(SYSTEM_BOOTANIMATION_FILE, R_OK) == 0) &&
                (mZip.open(SYSTEM_BOOTANIMATION_FILE) == NO_ERROR))) {
            mAndroidAnimation = false;
        }

        return NO_ERROR;
    }

说明：该代码在frameworks/base/cmds/bootanimation/BootAnimation.cpp中。

[skywang-todo]


<a name="anchor2_2"></a>
## 2.2 readyToRun()中dtoken的赋值

    sp<IBinder> dtoken(SurfaceComposerClient::getBuiltInDisplay(
            ISurfaceComposer::eDisplayIdMain));

上面是BootAnimation的readyToRun()中调用getBuiltInDisplay()的代码。下面分析一下该函数。


<a name="anchor2_2_1"></a>
### 2.2.1 ISurfaceComposer::eDisplayIdMain

    class ISurfaceComposer: public IInterface {                       
        ...
        enum {                                                        
            eDisplayIdMain = 0,                                       
            eDisplayIdHdmi = 1                                        
        };                      
        ...
    }

说明：该代码在frameworks/native/include/gui/ISurfaceComposer.h中。由此可见，eDisplayIdMain的值为0。


<a name="anchor2_2_2"></a>
### 2.2.2 SurfaceComposerClient::getBuiltInDisplay()

    sp<IBinder> SurfaceComposerClient::getBuiltInDisplay(int32_t id) {
        return Composer::getInstance().getBuiltInDisplay(id);
    }

说明：该代码在frameworks/native/libs/gui/SurfaceComposerClient.cpp中。该函数会调用Composer::getBuiltInDisplay()。



<a name="anchor2_2_3"></a>
### 2.2.3 Composer::getBuiltInDisplay()

    sp<IBinder> Composer::getBuiltInDisplay(int32_t id) {
        return ComposerService::getComposerService()->getBuiltInDisplay(id);
    }   

说明：ComposerService::getComposerService()的实现在[BootAnimation流程一][link_ui_UI_BootAnimation_Flow]介绍过，它是返回SurfaceFlinger的远程代理，该远程代理实际上是BpSurfaceComposer对象。下面就看看BpSurfaceComposer::getBuiltInDisplay()的代码。


<a name="anchor2_2_4"></a>
### 2.2.4 BpSurfaceComposer::getBuiltInDisplay()

    virtual sp<IBinder> getBuiltInDisplay(int32_t id)
    {
        Parcel data, reply;
        data.writeInterfaceToken(ISurfaceComposer::getInterfaceDescriptor());
        data.writeInt32(id);
        remote()->transact(BnSurfaceComposer::GET_BUILT_IN_DISPLAY, data, &reply);
        return reply.readStrongBinder();
    }

说明：该代码在frameworks/native/libs/gui/ISurfaceComposer.cpp中。它会通过Binder向SurfaceFlinger发送GET_BUILT_IN_DISPLAY请求，然后通过readStrongBinder()读出请求的处理结果，并返回。

在Binder机制中，"远程代理BpSurfaceComposer"发送的请求最终会交给"本地服务SurfaceFlinger"的onTransact()进行处理。下面就看看SurfaceFlinger是如何处理该服务的。


<a name="anchor2_2_5"></a>
### 2.2.5 SurfaceFlinger::onTransact()

    status_t SurfaceFlinger::onTransact(
        uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
    {               
        ...

        status_t err = BnSurfaceComposer::onTransact(code, data, reply, flags);

        ...
    }

说明：SurfaceFlinger的onTransact()会调用BnSurfaceComposer的onTransact()来处理GET_BUILT_IN_DISPLAY请求。



<a name="anchor2_2_6"></a>
### 2.2.6 BnSurfaceComposer::onTransact()

    status_t BnSurfaceComposer::onTransact(
        uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
    {
        switch(code) {
            ...
            case GET_BUILT_IN_DISPLAY: {
                ...
                int32_t id = data.readInt32();
                sp<IBinder> display(getBuiltInDisplay(id));
                reply->writeStrongBinder(display); 
                return NO_ERROR;
            }
            ...
        }
    }

说明：在处理GET_BUILT_IN_DISPLAY请求时，BnSurfaceComposer会先读取id，该id的值是0，它是BpSurfaceComposer的getBuiltInDisplay()中传入的id值。接着，就调用getBuiltInDisplay()。下面看看getBuiltInDisplay()的代码。



<a name="anchor2_2_7"></a>
### 2.2.7 SurfaceFlinger::getBuiltInDisplay()

    sp<IBinder> SurfaceFlinger::getBuiltInDisplay(int32_t id) {
        if (uint32_t(id) >= DisplayDevice::NUM_BUILTIN_DISPLAY_TYPES) {
            ...
        }       
        return mBuiltinDisplays[id];
    }       

说明：参数id的值是0，而NUM_BUILTIN_DISPLAY_TYPES的值为2。因此，该函数会返回mBuiltinDisplays[0]。

这里，补充说明一下NUM_BUILTIN_DISPLAY_TYPES的值，它的定义如下。

    class DisplayDevice : public LightRefBase<DisplayDevice>
    {
        ...
        enum DisplayType {                                            
            DISPLAY_ID_INVALID = -1,                                  
            DISPLAY_PRIMARY     = HWC_DISPLAY_PRIMARY,                
            DISPLAY_EXTERNAL    = HWC_DISPLAY_EXTERNAL,               
            DISPLAY_VIRTUAL     = HWC_DISPLAY_VIRTUAL,                
            NUM_BUILTIN_DISPLAY_TYPES = HWC_NUM_PHYSICAL_DISPLAY_TYPES,
        };          
        ...
    }

说明：该代码在frameworks/native/services/surfaceflinger/DisplayDevice.h中。从中发现，NUM_BUILTIN_DISPLAY_TYPES的值等于HWC_NUM_PHYSICAL_DISPLAY_TYPES。

    enum {    
        HWC_DISPLAY_PRIMARY     = 0,      
        HWC_DISPLAY_EXTERNAL    = 1,    // HDMI, DP, etc.
        HWC_DISPLAY_VIRTUAL     = 2,      

        HWC_NUM_PHYSICAL_DISPLAY_TYPES = 2,
        HWC_NUM_DISPLAY_TYPES          = 3,
    };

说明：该代码在hardware/libhardware/include/hardware/hwcomposer_defs.h中。


<br/>
了解了NUM_BUILTIN_DISPLAY_TYPES的值之后，下面接着说mBuiltinDisplays[0]。

    sp<IBinder> mBuiltinDisplays[DisplayDevice::NUM_BUILTIN_DISPLAY_TYPES];

上面是frameworks/native/services/surfaceflinger/SurfaceFlinger.h中mBuiltinDisplays的声明，而mBuiltinDisplays的赋值则是在SurfaceFlinger.cpp中。下面给mBuiltinDisplays赋值的代码。


    void SurfaceFlinger::init() {
        ...

        for (size_t i=0 ; i<DisplayDevice::NUM_BUILTIN_DISPLAY_TYPES ; i++) {
            DisplayDevice::DisplayType type((DisplayDevice::DisplayType)i);

            if (mHwc->isConnected(i) || type==DisplayDevice::DISPLAY_PRIMARY) {
                ...
                createBuiltinDisplayLocked(type);
                ...
            }
        }
    }

    void SurfaceFlinger::createBuiltinDisplayLocked(DisplayDevice::DisplayType type) {
        mBuiltinDisplays[type] = new BBinder();
        ...
    }

说明：init()是在SurfaceFlinger守护进程的main()函数被调用的，具体的可以查阅frameworks/native/services/surfaceflinger/main_surfaceflinger.cpp。init()中会调用createBuiltinDisplayLocked()，而createBuiltinDisplayLocked()则会给mBuiltinDisplays赋值。  
简单来说，mBuiltinDisplays[0]就是一个BBinder对象。

<br/>
回到BootAnimation.cpp的readyToRun()中，它调用getBuiltInDisplay()获取的就是一个BBinder对象。


<a name="anchor2_3"></a>
## 2.3 readyToRun()中的调用getDisplayInfo()的流程

        status_t status = SurfaceComposerClient::getDisplayInfo(dtoken, &dinfo);

上面是BootAnimation的readyToRun()中调用getDisplayInfo(dtoken, &dinfo)的代码。  
getDisplayInfo()的作用是获取屏幕的相关信息(如宽，高，屏幕密度，帧率等)。其中，dtoken是输入参数，表示要获取的是哪一个显示屏的信息；dinfo则是输出参数，是用来保存获取到的屏幕信息的变量。


<a name="anchor2_3_1"></a>
### 2.3.1 SurfaceComposerClient::getDisplayInfo()

    status_t SurfaceComposerClient::getDisplayInfo(
            const sp<IBinder>& display, DisplayInfo* info)
    {   
        return ComposerService::getComposerService()->getDisplayInfo(display, info);
    }   

说明：ComposerService::getComposerService()返回SurfaceFlinger的远程代理BpSurfaceComposer。接着调用BpSurfaceComposer::getDisplayInfo()来获取显示屏信息。


<a name="anchor2_3_2"></a>
### 2.3.2 BpSurfaceComposer::getDisplayInfo()

    virtual status_t getDisplayInfo(const sp<IBinder>& display, DisplayInfo* info)
    {
        Parcel data, reply;
        data.writeInterfaceToken(ISurfaceComposer::getInterfaceDescriptor());
        data.writeStrongBinder(display);
        remote()->transact(BnSurfaceComposer::GET_DISPLAY_INFO, data, &reply);
        memcpy(info, reply.readInplace(sizeof(DisplayInfo)), sizeof(DisplayInfo));
        return reply.readInt32();
    }

说明：该函数会先通过transact()向SurfaceFlinger发送一个GET_DISPLAY_INFO请求，然后读取请求反馈结果，并拷贝到inifo中。

接下来，我们就看看SurfaceFlinger的onTransact()是如何处理GET_DISPLAY_INFO请求的。


<a name="anchor2_3_3"></a>
### 2.3.3 SurfaceFlinger::onTransact()

    status_t SurfaceFlinger::onTransact(
        uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
    {               
        ...

        status_t err = BnSurfaceComposer::onTransact(code, data, reply, flags);

        ...
    }

说明：SurfaceFlinger会调用BnSurfaceComposer的onTransact()来处理GET_DISPLAY_INFO请求。


<a name="anchor2_3_4"></a>
### 2.3.4 BnSurfaceComposer::onTransact()

    status_t BnSurfaceComposer::onTransact(
        uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
    {
        switch(code) {
            ...
            case GET_DISPLAY_INFO: {
                DisplayInfo info;
                sp<IBinder> display = data.readStrongBinder();
                status_t result = getDisplayInfo(display, &info);
                memcpy(reply->writeInplace(sizeof(DisplayInfo)), &info, sizeof(DisplayInfo));
                reply->writeInt32(result);
                return NO_ERROR;
            }
            ...
        }
    }

说明：BnSurfaceComposer在处理GET_DISPLAY_INFO请求时，会先读出display，它实际上就是getBuiltInDisplay()返回的BBinder对象。接着，会调用getDisplayInfo()来获取DisplayInfo信息。


<a name="anchor2_3_5"></a>
### 2.3.5 SurfaceFlinger::getDisplayInfo()

    status_t SurfaceFlinger::getDisplayInfo(const sp<IBinder>& display, DisplayInfo* info) {
        int32_t type = NAME_NOT_FOUND;
        // 获取type的值。
        for (int i=0 ; i<DisplayDevice::NUM_BUILTIN_DISPLAY_TYPES ; i++) {
            if (display == mBuiltinDisplays[i]) {
                type = i;
                break;
            }
        }   
        
        if (type < 0) {
            return type;
        }
        
        // 获取HWComposer对象
        const HWComposer& hwc(getHwComposer());
        // 获取屏幕在x，y轴上的密度
        float xdpi = hwc.getDpiX(type);
        float ydpi = hwc.getDpiY(type);
            
        class Density {
            static int getDensityFromProperty(char const* propName) {
                char property[PROPERTY_VALUE_MAX];
                int density = 0;
                if (property_get(propName, property, NULL) > 0) {
                    density = atoi(property);
                }
                return density;
            }   
        public: 
            static int getEmuDensity() {
                return getDensityFromProperty("qemu.sf.lcd_density"); }
            static int getBuildDensity()  {
                return getDensityFromProperty("ro.sf.lcd_density"); }
        };      
                
            
        // 设置xdpi，ydpi, density以及orientation的值。
        if (type == DisplayDevice::DISPLAY_PRIMARY) {
            // The density of the device is provided by a build property
            float density = Density::getBuildDensity() / 160.0f;
            if (density == 0) {
                int short_length;
                short_length = hwc.getHeight(HWC_DISPLAY_PRIMARY);
                if(short_length > hwc.getWidth(HWC_DISPLAY_PRIMARY))
                   short_length = hwc.getWidth(HWC_DISPLAY_PRIMARY);

                if(short_length <= 480)
                  xdpi = ydpi = 120;
                else if(short_length <= 600)//all the others : dpi=160
                  xdpi = ydpi = 160;

                density = xdpi / 160.0f;
            }
            if (Density::getEmuDensity()) {
                xdpi = ydpi = density = Density::getEmuDensity();
                density /= 160.0f;
            }
            info->density = density;

            // 设置屏幕的旋转角度
            sp<const DisplayDevice> hw(getDefaultDisplayDevice());
            info->orientation = hw->getOrientation();
        } else {
            // TODO: where should this value come from?
            static const int TV_DENSITY = 213;
            info->density = TV_DENSITY / 160.0f;
            info->orientation = 0;
        }

        // 设置宽，高，xdpi,ydpi和fps(帧率)
        info->w = hwc.getWidth(type);
        info->h = hwc.getHeight(type);
        info->xdpi = xdpi;
        info->ydpi = ydpi;
        info->fps = float(1e9 / hwc.getRefreshPeriod(type));

        // All non-virtual displays are currently considered secure.
        info->secure = true;

        ...
        return NO_ERROR;
    }

说明：getDisplayInfo()的作用是获取显示屏的屏幕密度，宽，高，旋转角度，帧率等信息。  
(01) 该函数会先通过for循环来获取type的值。由于display的值是mBuiltinDisplays[0]，因此type的值为0。  
(02) 接着通过getHwComposer()获取HWComposer对象，实际上getHwComposer()返回的是mHwc所指的HWComposer对象，getHwComposer()的实现在SurfaceFlinger.h中。而mHwc则是在SurfaceFlinger::init()中创建的HWComposer对象。
    
    HWComposer& getHwComposer() const { return *mHwc; }

(03) 在获取到HWComposer对象hwc之后，再通过hwc获取到屏幕密度。  
(04) type等于0，而DISPLAY_PRIMARYy也等于0。于是进入 if (type == DisplayDevice::DISPLAY_PRIMARY) 中对屏幕密度相关的变量xdpi, ydpi, density和orientation进行赋值。  
(05) 最后再将屏幕的宽，高，屏幕密度和帧率等信息保存到info中。


<br/>
获取到屏幕信息之后，再通过Binder机制反馈给BpSurfaceComposer；最终，该信息会保存到readyToRun()的dinfo中。



<a name="anchor2_4"></a>
## 2.4 readyToRun()中createSurface()的调用流程

        sp<SurfaceControl> control = session()->createSurface(String8("BootAnimation"),
                dinfo.w, dinfo.h, PIXEL_FORMAT_RGB_565);

前面通过getDisplayInfo()获取到屏幕信息之后；接着就根据获取的到屏幕的宽和高，调用createSurface()来创建Surface对象。session()返回的是mSession对象，而mSession是在BootAnimation构造函数中创建的SurfaceComposerClient对象。

    sp<SurfaceComposerClient> BootAnimation::session() const {
        return mSession;
    }


<a name="anchor2_4_1"></a>
### 2.4.1 SurfaceComposerClient::createSurface()

    sp<SurfaceControl> SurfaceComposerClient::createSurface(
            const String8& name,
            uint32_t w,
            uint32_t h,
            PixelFormat format,
            uint32_t flags)
    {
        sp<SurfaceControl> sur;
        if (mStatus == NO_ERROR) {
            sp<IBinder> handle;
            sp<IGraphicBufferProducer> gbp;
            status_t err = mClient->createSurface(name, w, h, format, flags,
                    &handle, &gbp);
            if (err == NO_ERROR) {
                sur = new SurfaceControl(this, handle, gbp);
            }
        }
        return sur;
    }

说明：该函数会先调用mClient->createSurface()创建Surface，然后在新建SurfaceControl对象并返回。  
mClient是在onFirstRef()中初始化的，它是通过Client客户端的远程代理，即BpSurfaceComposerClient对象。


<a name="anchor2_4_2"></a>
### 2.4.2 BpSurfaceComposerClient::createSurface()

    class BpSurfaceComposerClient : public BpInterface<ISurfaceComposerClient>
    {           
        ...
        virtual status_t createSurface(const String8& name, uint32_t w,
                uint32_t h, PixelFormat format, uint32_t flags,
                sp<IBinder>* handle,
                sp<IGraphicBufferProducer>* gbp) {
            Parcel data, reply;
            data.writeInterfaceToken(ISurfaceComposerClient::getInterfaceDescriptor());
            data.writeString8(name);
            data.writeInt32(w);
            data.writeInt32(h);
            data.writeInt32(format);
            data.writeInt32(flags);       
            remote()->transact(CREATE_SURFACE, data, &reply);
            *handle = reply.readStrongBinder();
            *gbp = interface_cast<IGraphicBufferProducer>(reply.readStrongBinder());
            return reply.readInt32();
        }
        ...
    }

说明：该函数会先通过Binder机制发送一个CREATE_SURFACE请求给Client客户端。然后读取请求返回结果并赋值给*gbp。

下面，看看Client的onTransact()中处理CREATE_SURFACE的流程。


<a name="anchor2_4_3"></a>
### 2.4.3 Client::onTransact()

    status_t Client::onTransact(
        uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
    {
        // 权限检查
         IPCThreadState* ipc = IPCThreadState::self();
         const int pid = ipc->getCallingPid();
         const int uid = ipc->getCallingUid();
         const int self_pid = getpid();
         if (CC_UNLIKELY(pid != self_pid && uid != AID_GRAPHICS && uid != 0)) {
             // we're called from a different process, do the real check
             if (!PermissionCache::checkCallingPermission(sAccessSurfaceFlinger))
             {
                 ALOGE("Permission Denial: "
                         "can't openGlobalTransaction pid=%d, uid=%d", pid, uid);
                 return PERMISSION_DENIED;
             }
         }
         return BnSurfaceComposerClient::onTransact(code, data, reply, flags);
    }

说明：Client在处理Binder驱动转发的请求时，会先进行权限检查。如果有权限，则调用BnSurfaceComposer::onTransact()来处理请求。


<a name="anchor2_4_4"></a>
### 2.4.4 BnSurfaceComposerClient::onTransact()

    status_t BnSurfaceComposerClient::onTransact(
        uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
    {
         switch(code) {
            case CREATE_SURFACE: {
                CHECK_INTERFACE(ISurfaceComposerClient, data, reply);
                String8 name = data.readString8();
                uint32_t w = data.readInt32();
                uint32_t h = data.readInt32();
                PixelFormat format = data.readInt32();
                uint32_t flags = data.readInt32();
                sp<IBinder> handle;
                sp<IGraphicBufferProducer> gbp;
                status_t result = createSurface(name, w, h, format, flags,
                        &handle, &gbp);
                reply->writeStrongBinder(handle);
                reply->writeStrongBinder(gbp->asBinder());
                reply->writeInt32(result);
                return NO_ERROR;
            } break;
            ...
        }
    }

说明：BnSurfaceComposerClient在处理CREATE_SURFACE请求时，会先读取BpSurfaceComposerClient传递的请求数据，然后在调用createSurface()创建Surface。


<a name="anchor2_4_5"></a>
### 2.4.5 Client::createSurface()

    status_t Client::createSurface(
            const String8& name,
            uint32_t w, uint32_t h, PixelFormat format, uint32_t flags,
            sp<IBinder>* handle,
            sp<IGraphicBufferProducer>* gbp)
    {   
        class MessageCreateLayer : public MessageBase {
            SurfaceFlinger* flinger;
            Client* client;
            sp<IBinder>* handle;
            sp<IGraphicBufferProducer>* gbp;
            status_t result;
            const String8& name;
            uint32_t w, h;
            PixelFormat format;
            uint32_t flags;
        public:
            MessageCreateLayer(SurfaceFlinger* flinger,
                    const String8& name, Client* client,
                    uint32_t w, uint32_t h, PixelFormat format, uint32_t flags,
                    sp<IBinder>* handle,
                    sp<IGraphicBufferProducer>* gbp)
                : flinger(flinger), client(client),
                  handle(handle), gbp(gbp),
                  name(name), w(w), h(h), format(format), flags(flags) {
            }
            status_t getResult() const { return result; }
            virtual bool handler() {
                result = flinger->createLayer(name, client, w, h, format, flags,
                        handle, gbp);
                return true;
            }
        };  
            
        sp<MessageBase> msg = new MessageCreateLayer(mFlinger.get(),
                name, this, w, h, format, flags, handle, gbp);
        mFlinger->postMessageSync(msg);
        return static_cast<MessageCreateLayer*>( msg.get() )->getResult();
    }   

说明：Client与SurfaceFlinger之间通信，将通过将消息发送到SurfaceFlinger的消息循环来进行的。该函数会新建一个MessageCreateLayer对象；然后，调用mFlinger->postMessageSync()将消息发送给SurfaceFlinger，这里的mFlinger就是SurfaceFlinger对象，它是在创建Client对象时赋值的。


<a name="anchor2_4_6"></a>
### 2.4.6 SurfaceFlinger::postMessageSync()

    status_t SurfaceFlinger::postMessageSync(const sp<MessageBase>& msg,
            nsecs_t reltime, uint32_t flags) {
        status_t res = mEventQueue.postMessage(msg, reltime);
        if (res == NO_ERROR) {
            msg->wait();
        }
        return res;
    }

说明：该函数会将消息添加到消息队列mEventQueue中，mEventQueue是MessageQueue的对象。这里的参数relTime和flags都是默认值0。

    mutable MessageQueue mEventQueue; 

上面是SurfaceFlinger.h中mEventQueue的定义。


<a name="anchor2_4_7"></a>
### 2.4.7 MessageQueue::postMessage()

    status_t MessageQueue::postMessage(
            const sp<MessageBase>& messageHandler, nsecs_t relTime)
    {   
        const Message dummyMessage;
        if (relTime > 0) {
            mLooper->sendMessageDelayed(relTime, messageHandler, dummyMessage);
        } else {
            mLooper->sendMessage(messageHandler, dummyMessage);
        }
        return NO_ERROR;
    }   

说明：该代码在frameworks/native/services/surfaceflinger/MessageQueue.cpp中。这里的参数relTime等于0，因此会调用mLooper->sendMessage()。而mLooper又是什么呢？

mLooper是Looper的对象，它是在MessageQueue::init()中初始化的，代码如下：

    void MessageQueue::init(const sp<SurfaceFlinger>& flinger)
    {       
        mFlinger = flinger;
        mLooper = new Looper(true);
        mHandler = new Handler(*this);
    }       

在init()中新建了mLooper对象。而这里的init()是在SurfaceFlinger的onFirstRef()中被调用的。


<a name="anchor2_4_8"></a>
### 2.4.8 Looper::sendMessage()

    void Looper::sendMessage(const sp<MessageHandler>& handler, const Message& message) {
        nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
        sendMessageAtTime(now, handler, message);
    }           

    void Looper::sendMessageAtTime(nsecs_t uptime, const sp<MessageHandler>& handler,
            const Message& message) {

        size_t i = 0;
        // 将消息添加到mMessageEnvelopes矢量队列中
        { 
            AutoMutex _l(mLock);
            
            size_t messageCount = mMessageEnvelopes.size();
            while (i < messageCount && uptime >= mMessageEnvelopes.itemAt(i).uptime) {
                i += 1;
            }
        
            MessageEnvelope messageEnvelope(uptime, handler, message);
            mMessageEnvelopes.insertAt(messageEnvelope, i, 1);

            // Optimization: If the Looper is currently sending a message, then we can skip
            // the call to wake() because the next thing the Looper will do after processing
            // messages is to decide when the next wakeup time should be.  In fact, it does
            // not even matter whether this code is running on the Looper thread.
            if (mSendingMessage) {
                return;
            }
        }
        
        // 唤醒管道上读取mMessageEnvelopes队列中消息的进程
        if (i == 0) {
            wake();
        }
    }

说明：sendMessage()会调用sendMessageAtTime()。而在sendMessageAtTime()中，会先将消息添加到mMessageEnvelopes中。mMessageEnvelopes是一个矢量队列，它相当于"生产-消费者"模型中的仓库。当有消息来的时候，会被添加到mMessageEnvelopes中；当mMessageEnvelopes中的消息不为空的时候，读取消息的进程会取出mMessageEnvelopes的消息。这里，将消息添加到mMessageEnvelopes中之后，就调用wake()唤醒读取消息的进程来读取该消息。


<a name="anchor2_4_9"></a>
### 2.4.9 Looper::wake()

    void Looper::wake() {
        ssize_t nWrite;
        do {
            nWrite = write(mWakeWritePipeFd, "W", 1);
        } while (nWrite == -1 && errno == EINTR);
        
        ...
    }

说明：wake()的内容很简单，就是向管道中写入一个"W"。这里，mWakeWritePipeFd是管道句柄；而写入的内容具体是什么是无关紧要的，重要的是写入了东西。接着，管道读取进程就会被唤醒，进而读出管道中的消息。而哪一个是管道读取进程呢？


管道的读取进程正好就是SurfaceFlinger进程。回想一下SurfaceFlinger进程的流程：在主函数main()中会调用SurfaceFlinger的run()方法。

    int main(int argc, char** argv) {
        ...

        flinger->run();
        return 0;
    }

说明：该代码在frameworks/native/services/surfaceflinger/main_surfaceflinger.cpp中。

    void SurfaceFlinger::run() {
        do {
            waitForEvent();
        } while (true);
    }

    void SurfaceFlinger::waitForEvent() {
        mEventQueue.waitMessage();
    }


说明：SurfaceFlinger中的run()会调用waitForEvent()。而waitForEvent()又会调用mEventQueue.waitMessage()进入等待状态。下面看看MessageQueue中的waitMessage()代码。


    void MessageQueue::waitMessage() {
        do {    
            IPCThreadState::self()->flushCommands();
            int32_t ret = mLooper->pollOnce(-1);
            switch (ret) {
                case ALOOPER_POLL_WAKE:
                case ALOOPER_POLL_CALLBACK:
                    continue;
                case ALOOPER_POLL_ERROR:  
                    ALOGE("ALOOPER_POLL_ERROR");
                case ALOOPER_POLL_TIMEOUT:
                    // timeout (should not happen)
                    continue;
                default:
                    // should not happen
                    ALOGE("Looper::pollOnce() returned unknown status %d", ret);
                    continue;
            }
        } while (true);
    }

说明：waitMessage()就是一个消息循环，它会不断的读取消息队列中消息并进行处理。  
(01) flushCommands()的作用是处理该进程中的Binder请求。但是，该情况下进行中没有任何Binder请求，所以，flushCommands()实际上什么也没做。  
(02) mLooper->pollOnce()就是从消息队列中的读取一则消息并进行处理，如果没有消息，则会进入等待状态。下面看看Looper中pollOnce()的源码。读者可以在[Android消息机制架构和源码解析][link_msg_MessageQueue]一文中了解Android的消息队列框架。


    int Looper::pollOnce(int timeoutMillis, int* outFd, int* outEvents, void** outData) {
        int result = 0;
        for (;;) { 
            while (mResponseIndex < mResponses.size()) {
                const Response& response = mResponses.itemAt(mResponseIndex++);
                int ident = response.request.ident;
                if (ident >= 0) {
                    int fd = response.request.fd;
                    int events = response.events;
                    void* data = response.request.data;
                    if (outFd != NULL) *outFd = fd;
                    if (outEvents != NULL) *outEvents = events;
                    if (outData != NULL) *outData = data;
                    return ident;
                }
            }

            if (result != 0) {
                if (outFd != NULL) *outFd = 0;
                if (outEvents != NULL) *outEvents = 0;
                if (outData != NULL) *outData = NULL;
                return result;
            }       
                
            result = pollInner(timeoutMillis);
        }
    }           

说明：  
(01) mResponseIndex的初始值是0，而mResponses矢量数组也为空；因此，mResponseIndex等于mResponses.size()。  
(02) result等于0，因此会调用pollInner()。下面就看看pollInner()的代码。


    int Looper::pollInner(int timeoutMillis) {

        ...

        // 通过epoll_wait()等待mEpollFd上IO事件的发生
        struct epoll_event eventItems[EPOLL_MAX_EVENTS];
        int eventCount = epoll_wait(mEpollFd, eventItems, EPOLL_MAX_EVENTS, timeoutMillis);

        ...

        // 如果epoll_wait()时出错，则直接跳到Done处。
        if (eventCount < 0) {
            ...
            goto Done;
        }

        // 如果没有IO事件发生，则直接跳到Done处。
        if (eventCount == 0) {
            ...
            goto Done;
        }

        // 如果有IO事件发生，则逐个取出IO事件，如果是写事件(EPOLLIN)，则调用awoken()
        for (int i = 0; i < eventCount; i++) {
            int fd = eventItems[i].data.fd;
            uint32_t epollEvents = eventItems[i].events;
            if (fd == mWakeReadPipeFd) {
                if (epollEvents & EPOLLIN) {
                    awoken();
                } else {
                    ALOGW("Ignoring unexpected epoll events 0x%x on wake read pipe.", epollEvents);
                }
            } else {
                ...
            }
        }

    Done: ;

        // 取出消息并进行处理
        while (mMessageEnvelopes.size() != 0) {
            nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
            const MessageEnvelope& messageEnvelope = mMessageEnvelopes.itemAt(0);
            if (messageEnvelope.uptime <= now) {
                {
                    sp<MessageHandler> handler = messageEnvelope.handler;
                    Message message = messageEnvelope.message;
                    mMessageEnvelopes.removeAt(0);
                    mSendingMessage = true;
                    mLock.unlock();

                    // 处理消息
                    handler->handleMessage(message);
                } // release handler

                ...
            } else {
                ...
            }
        }

        ...

        return result;
    }

说明：pollInner()就是先通过epoll_wait()进入空闲等待状态，等待消息队列的管道上的消息(IO事件)。

OK，执行完epoll_wait()之后，SurfaceFlinger进程就进入了等待状态。而前面的BootAnimation在调用createSurface()时，有向管道中写入消息。写入消息之后，SurfaceFlinger就被唤醒了！  
SurfaceFlinger被唤醒之后，就接着epoll_wait()往下执行。首先，它会调用awoken()将消息读取出来。接着，会取出mMessageEnvelopes矢量队列中的消息，并调用handler->handleMessage()来处理消息。下面就分别对awoken()和handleMessage()来进行说明。


    void Looper::awoken() {
        char buffer[16];
        ssize_t nRead;
        do {
            nRead = read(mWakeReadPipeFd, buffer, sizeof(buffer));
        } while ((nRead == -1 && errno == EINTR) || nRead == sizeof(buffer));
    }

说明：awoken()的作用就是读取管道中的内容，这里就是前面写入管道的字符"W"。前面说过，管道中内容具体是什么并不重要；关键是通过这个内容能实现不同进程间的通信。


下面就说说pollInner()中的handler->handleMessage()。首先，这里的handler是什么呢？回到pollInner()中，它是从mMessageEnvelopes中取出的消息所包含的句柄，而这则消息是BootAnimation调用createSurface()时发送的。根据它的调用流程，我们知道该handler是Client类的createSurface()函数中的内部类MessageCreateLayer的实例。   
搞清楚这个关系之后，就看看handleMessage()的实现。

        
    void MessageBase::handleMessage(const Message&) {
        this->handler();
        ...
    };

说明：该代码在frameworks/native/services/surfaceflinger/MessageQueue.cpp中。它会调用handler()对消息进行处理。


        virtual bool handler() {      
            result = flinger->createLayer(name, client, w, h, format, flags,                    
                    handle, gbp);     
            return true;              
        } 

说明：上面是MessageCreateLayer中handler()的实现。它会调用SurfaceFlinger的createSurface()方法。



## SurfaceFlinger::createLayer

    status_t SurfaceFlinger::createLayer(
            const String8& name,
            const sp<Client>& client,
            uint32_t w, uint32_t h, PixelFormat format, uint32_t flags,
            sp<IBinder>* handle, sp<IGraphicBufferProducer>* gbp)
    {   
        ...
        sp<Layer> layer;
            
        switch (flags & ISurfaceComposerClient::eFXSurfaceMask) { 
            case ISurfaceComposerClient::eFXSurfaceNormal:
                result = createNormalLayer(client,
                        name, w, h, flags, format,
                        handle, gbp, &layer);
                break;
            ...
        }

        if (result == NO_ERROR) {
            addClientLayer(client, *handle, *gbp, layer);
            setTransactionFlags(eTransactionNeeded);
        }
        return result;
    }

说明：flags的值为0，而ISurfaceComposerClient::eFXSurfaceMask的值也为0。因此，会通过createNormalLayer()来创建Surface。



## SurfaceFlinger::createNormalLayer()

    status_t SurfaceFlinger::createNormalLayer(const sp<Client>& client,
            const String8& name, uint32_t w, uint32_t h, uint32_t flags, PixelFormat& format,
            sp<IBinder>* handle, sp<IGraphicBufferProducer>* gbp, sp<Layer>* outLayer)
    {   
        switch (format) {       
        case PIXEL_FORMAT_TRANSPARENT:
        case PIXEL_FORMAT_TRANSLUCENT:
            format = PIXEL_FORMAT_RGBA_8888;
            break;
        case PIXEL_FORMAT_OPAQUE: 
    #ifdef NO_RGBX_8888
            format = PIXEL_FORMAT_RGB_565;
    #else   
            format = PIXEL_FORMAT_RGBX_8888;
    #endif  
            break;
        }
        
    #ifdef NO_RGBX_8888
        if (format == PIXEL_FORMAT_RGBX_8888)
            format = PIXEL_FORMAT_RGBA_8888;
    #endif      
                
        *outLayer = new Layer(this, client, name, w, h, flags);
        status_t err = (*outLayer)->setBuffers(w, h, format, flags);
        if (err == NO_ERROR) {
            *handle = (*outLayer)->getHandle();
            *gbp = (*outLayer)->getBufferQueue();
        }
        
        ALOGE_IF(err, "createNormalLayer() failed (%s)", strerror(-err));
        return err;
    }       









[link_binder_01_introduce]: /2014/09/01/Binder-Introduce/
[link_binder_02_datastruct]: /2014/09/02/Binder-Datastruct/
[link_binder_03_ServiceManagerDeamon]: /2014/09/03/Binder-ServiceManager-Daemon/
[link_binder_04_defaultServiceManager]: /2014/09/04/Binder-defaultServiceManager/
[link_binder_05_addService01]: /2014/09/05/BinderCommunication-AddService01/
[link_binder_05_addService02]: /2014/09/05/BinderCommunication-AddService02/
[link_binder_05_addService03]: /2014/09/05/BinderCommunication-AddService03/
[link_binder_06_threadpool]: /2014/09/06/BinderCommunication-ThreadPool/
[link_binder_07_getService01]: /2014/09/07/BinderCommunication-GetService01/

[link_msg_MessageQueue]: /2014/08/26/MessageQueue/

[link_ui_UI_BootAnimation_Flow]: /2014/09/13/UI-BootAnimation-Flow/

