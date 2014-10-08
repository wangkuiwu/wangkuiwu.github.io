---
layout: post
title: "Android Binder机制(五) addService详解01之 请求的发送"
description: "android"
category: android
tags: [android]
date: 2014-09-05 09:01
---

> 终于要开始讲解Client-Server交互了，若标题所示，本文要讲解的是addService请求，即添加服务请求。本文选取的题材是MediaPlayerService服务通过addService请求注册到ServiceManager中。  
> 在这个addService请求中，MediaPlayerService是Client，而ServiceManager是Server。由于涉及到的过程比较复杂，这里会将addService请求分为3篇进行说明，这3篇的主题分别是：请求的发送，请求的处理，以及请求的反馈。和以往一样，在讲解详细的代码之前，先做个整体介绍。

> 注意：本文是基于Android 4.4.2版本进行介绍的！


<a name="anchor1"></a>
# 1. addService流程的时序图

<a href="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/binder/addService_01_flow.jpg"><img src="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/binder/addService_01_flow.jpg" alt="" /></a>

上面是addService流程的时序图。理解这个图的前提是理解图中的三种角色之间的关系：  
(01) MediaPlayerService和ServiceManager是两个不同的进程。它们都位于用户空间，都有各自的内存单元，两者之间不能直接进行通信；因此，需要Binder驱动的帮助才能通信。  
(02) Binder驱动位于内核空间，它映射到节点"/dev/binder"上。MediaPlayerService和ServiceManager都有通过open("/dev/binder")打开该节点，并通过mmap()将内存映射到各自所在的进程中；这也就是说MediaPlayerService能和Binder驱动通信，而且ServiceManager也能和Binder驱动通信。而在Binder驱动中，有一个全局变量，依靠这个全局变量，就能实现MediaPlayerService和ServiceManager之间的通信。  依靠的这个全局变量，就是[Android Binder机制(三) ServiceManager守护进程][link_binder_03_ServiceManagerDeamon]中介绍过的binder_context_mgr_node变量，它是ServiceManager的Binder实体。

搞清楚它们三者之间的关系之后，再回到上面的时序图中。  
1. WAIT  
这表示ServiceManager进入了中断等待状态。它进入等待状态的详细流程，在[Android Binder机制(三) ServiceManager守护进程][link_binder_03_ServiceManagerDeamon]有介绍过。  

2. BC_TRANSACTION  
这是MediaPlayerService向ServiceManager发送addService请求对应的事务。这个事务是请求，而不是回复；因此是BC开发，B代表Binder，而C代表Command。如果是回复，则会以BR开发，R表示Reply。Binder驱动在收到BC_TRANSACTION之后，会将分配内存，将请求数据保存到所分配的内存中。

3. WAKE_UP  
MediaPlayerService通过BC_TRANSACTION提交一个请求，该请求是交给ServiceManager来处理的。因此，Binder驱动在收到该请求后，会将其发送到ServiceManager的待处理事务队列中，并将ServiceManager唤醒。

4. BR_TRANSACTION_COMPLETE  
MediaPlayerService在发起了一个请求之后，它需要知道该请求是否发送成功。因此，Binder驱动在将该请求提交给ServiceManager之后，会反馈一个BR_TRANSACTION_COMPLETE给MediaPlayerService，表示MediaPlayerService发送的请求已经被Binder驱动收到了。

5. WAIT  
MediaPlayerService在知道自己的请求已经发送成功之后，就进入等待状态，等待请求的反馈结果。

6. BR_NOOP和BR_TRANSACTION  
ServiceManager被唤醒之后，收到Binder驱动的BR_NOOP和BR_TRANSACTION指令。BR_NOOP指令什么也不会做；而对于BR_TRANSACTION指令时，ServiceManager在解析出该事务是添加服务请求，会将MediaPlayerService的相关信息保存到一个链表中。

7. BC_FREE_BUFFER和BC_REPLY  
ServiceManager在保存了MediaPlayerService的相关信息之后，便处理完毕了MediaPlayerService的请求。此时，它便反馈BC_FREE_BUFFER和BC_REPLY给Binder驱动。Binder驱动在收到BC_FREE_BUFFER之后，会释放保存请求数据所申请的内存；收到BC_REPLY之后，Binder驱动则知道ServiceManager已经处理完了MediaPlayerService的请求。  
接着，Binder驱动便会唤醒MediaPlayerService，并发送BR_NOOP和BR_REPLY给MediaPlayerService，告诉MediaPlayerService请求已经处理完毕。同时，它还会发送一个BR_TRANSACTION_COMPLETE给ServiceManager，告诉ServiceManager该事务已经处理完毕。 MediaPlayerService在收到BR_REPLY反馈之后，知道addService请求已经成功处理；接着，它会再次进入等待状态，等待Client的请求。   
最后，ServiceManager处理MediaPlayerService的请求之后，没有其他事务可处理，也再次进入了等待状态。




<a name="anchor2"></a>
# 2. IMediaPlayerService的类图

本文是以MediaPlayerService为例，对addService进行解析。下面看看MediaPlayerService相关联的类图。

<a href="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/binder/IMediaPlayerService_leitu.jpg"><img src="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/binder/IMediaPlayerService_leitu.jpg" alt="" /></a>

IMediaPlayerService的类图和"[Android Binder机制(四) defaultServiceManager()的实现][link_binder_04_defaultServiceManager]"中IServiceManager的类图类似。这里就不再逐一对每个类进行介绍了。

需要知道的是，对于一个MediaPlayerService而言，它存在一个"远程BpBinder对象"和"本地BBinder对象"。   
(01) 远程BpBinder对象的作用，是和Binder驱动进行交互。例如，当本文所讲到的addService请求，就是通过defaultServiceManager()调用到远程BpBinder对象的transact()方法，而该方法又会调用到IPCThreadState::transact()接口，通过IPCThreadState类来和Binder驱动交互。   
(02) MediaPlayerService是"本地BBinder的子类"。当Client向MediaPlayerService发起请求时，会调用BBinder的onTransact()方法，而BnServiceManager又重写了该方法，从而调用onTransact()完成对请求的处理。



<a name="anchor3"></a>
# addService请求发送的代码解析

下面通过代码来查看addService请求的发送流程。



<a name="anchor3_1"></a>
## 1. MediaPlayerService的main()函数

先看看MediaPlayerService的main()函数代码。

    int main(int argc, char** argv)
    {
        signal(SIGPIPE, SIG_IGN);
        char value[PROPERTY_VALUE_MAX];
        bool doLog = (property_get("ro.test_harness", value, "0") > 0) && (atoi(value) == 1);
        pid_t childPid;

        if (doLog && (childPid = fork()) != 0) {
            ...
        } else {
            // all other services
            ...
            sp<ProcessState> proc(ProcessState::self());
            sp<IServiceManager> sm = defaultServiceManager();
            ...
            MediaPlayerService::instantiate();
            ...
            ProcessState::self()->startThreadPool();
            IPCThreadState::self()->joinThreadPool();
        }
    }

说明：该代码在frameworks/av/media/mediaserver/main_mediaserver.cpp中。  
(01) property_get("ro.test_harness", value, "0")是获取"ro.test_harness"属性，为false。  
(02) ProcessState:self()是获取ProcessState对象，并赋值给proc。ProcessState::self()在[Android Binder机制(四) defaultServiceManager()的实现][link_binder_04_defaultServiceManager]中已经介绍过了。  
(03) defaultServiceManager()是获取IServiceManager对象，它的实现在[Android Binder机制(四) defaultServiceManager()的实现][link_binder_04_defaultServiceManager]中也有详细介绍。  
(04) MediaPlayerService::instantiate()是初始化MediaPlayerService服务。  





<a name="anchor3_2"></a>
## 2. MediaPlayerService::instantiate()

    void MediaPlayerService::instantiate() {
        defaultServiceManager()->addService(
                String16("media.player"), new MediaPlayerService());
    }

说明：该代码在frameworks/av/media/libmediaplayerservice/MediaPlayerService.cpp中。它会新建MediaPlayerService对象；然后调用defaultServiceManager()获取到的BpServiceManager的实例；最后，调用BpServiceManager的addService()方法，将MediaPlayerService对象添加到Service Manager中。MediaPlayerService服务的名称是"media.player"。




<a name="anchor3_3"></a>
## 3. MediaPlayerService::MediaPlayerService()

    MediaPlayerService::MediaPlayerService()
    {
        ALOGV("MediaPlayerService created");
        mNextConnId = 1; 

        mBatteryAudio.refCount = 0; 
        for (int i = 0; i < NUM_AUDIO_DEVICES; i++) {
            mBatteryAudio.deviceOn[i] = 0; 
            mBatteryAudio.lastTime[i] = 0; 
            mBatteryAudio.totalTime[i] = 0; 
        }    
        // speaker is on by default
        mBatteryAudio.deviceOn[SPEAKER] = 1; 
        mOOMKilling = false;
        MediaPlayerFactory::registerBuiltinFactories();
    }

说明：MediaPlayerService的构造函数比较简单，就是进行一些变量的初始化。



<a name="anchor3_4"></a>
## 4. BpServiceManager::addService()

    class BpServiceManager : public BpInterface<IServiceManager>
    {
    public:
        ...

        virtual status_t addService(const String16& name, const sp<IBinder>& service,          
                bool allowIsolated)         
        {     
            Parcel data, reply;             
            data.writeInterfaceToken(IServiceManager::getInterfaceDescriptor());
            data.writeString16(name);       
            data.writeStrongBinder(service);
            data.writeInt32(allowIsolated ? 1 : 0);
            status_t err = remote()->transact(ADD_SERVICE_TRANSACTION, data, &reply);          
            return err == NO_ERROR ? reply.readExceptionCode() : err;
        }     

        ...
    }

说明：该代码在frameworks/native/libs/binder/IServiceManager.cpp中。addService()会现将MediaPlayerService服务的名称("media.player")以及它的实例等参数保存到data(Parcel对象)中，然后再调用remote()返回的BpBinder对象的transact()与Binder驱动进行交互。  
(01) 先看看addService()的各个参数。name="media.player"，即MediaPlayerService服务的名称；service就是MediaPlayerService对象，而IBinder是MediaPlayerService的父类；allowIsolated这个值默认为false，默认值的定义在frameworks/native/include/binder/IServiceManager.h的addService()函数声明中。  
(02) Parcel是Binder通信的数据存储结构，它的各个成员和函数在[Android Binder机制(二) Binder中的数据结构][link_binder_02_datastruct]中有详细说明。  
在向data中写入数据时，先通过writeInterfaceToken()写入数据头，这里的数据头是：int32的整形数+字符串(字符串是"android.os.IServiceManager")。writeString16(name)写入的是服务的名称，即"media.player"。writeStrongBinder(service)是将MediaPlayerService封装到flat_binder_object结构体中。最后的writeInt32()暂时不用关心。  
下面，我们逐个对data的赋值进行介绍。





<a name="anchor3_5"></a>
## 5. Parcel::Parcel()

先看看Parcel的构造函数。

    Parcel::Parcel()
    {   
        initState(); 
    }   

说明：该代码在frameworks/native/libs/binder/Parcel.cpp中。  


<a name="anchor3_6"></a>
## 6. Parcel::initState()

    void Parcel::initState()
    {
        mError = NO_ERROR;
        mData = 0;              // 数据的地址指针
        mDataSize = 0;          // 数据的大小
        mDataCapacity = 0;      // 数据的容量
        mDataPos = 0;           // 数据的位置
        mObjects = NULL;        // 保存对象的地址指针
        mObjectsSize = 0;       // 对象的个数
        mObjectsCapacity = 0;   // 对象的容量
        mNextObjectHint = 0;
        mHasFds = false;
        mFdsKnown = true;
        mAllowFds = true;
        mOwner = NULL;
    }

说明：该函数对Parcel的成员进行了初始化。




<a name="anchor3_7"></a>
## 7. Parcel::writeInterfaceToken()

    下面看看data.writeInterfaceToken(IServiceManager::getInterfaceDescriptor())的实现。getInterfaceDescriptor()是通过宏IMPLEMENT_META_INTERFACE()实现的，该宏已经在[Android Binder机制(四) defaultServiceManager()的实现][link_binder_04_defaultServiceManager]中介绍过了；getInterfaceDescriptor()的返回值是"android.os.IServiceManager"。  
    即data.writeInterfaceToken("android.os.IServiceManager")。下面看看writeInterfaceToken()的实现。

    status_t Parcel::writeInterfaceToken(const String16& interface)
    {       
        writeInt32(IPCThreadState::self()->getStrictModePolicy() |
                   STRICT_MODE_PENALTY_GATHER);
        // currently the interface identification token is just its name as a string
        return writeString16(interface);
    }   


说明：该函数先通过writeInt32()写入一个32位的int数到Parcel中，然后再通过writeString16()将字符串写入到Parcel中。它所写入的是数据头，ServiceManager中收到该数据之后，会先获取数据头，并根据数据头来判断数据的有效性！    
(01) IPCThreadState::self()返回IPCThreadState对象；然后，调用IPCThreadState::getStrictModePolicy()，返回的是mStrictModePolicy，mStrictModePolicy的初始值是0。因此，writeInt32()就可以简化为writeInt32(STRICT_MODE_PENALTY_GATHER)。  
(02) writeString16(interface)是writeString16("android.os.IServiceManager")。




<a name="anchor3_8"></a>
## 8. Parcel::writeInt32()

    status_t Parcel::writeInt32(int32_t val)
    {   
        return writeAligned(val);
    }   

说明：该函数调用writeAligned()。



<a name="anchor3_9"></a>
## 9. Parcel::writeAligned()

    template<class T>
    status_t Parcel::writeAligned(T val) {
        COMPILE_TIME_ASSERT_FUNCTION_SCOPE(PAD_SIZE(sizeof(T)) == sizeof(T));

        if ((mDataPos+sizeof(val)) <= mDataCapacity) {
    restart_write:
            *reinterpret_cast<T*>(mData+mDataPos) = val;
            return finishWrite(sizeof(val));
        }

        status_t err = growData(sizeof(val));
        if (err == NO_ERROR) goto restart_write;
        return err;
    }

说明：writeAligned()的作用是是写入数据，比同步相应的变量。  
(01) mDataPos的初始值=0，sizeof(val)=4，mDataCapacity的初始值=0。因此，if((mDataPos+sizeof(val)) <= mDataCapacity)为false。  
(02) 接下来，会先调用growData(sizeof(val))来增加容量，然后再将数据写入到mData中。  



<a name="anchor3_10"></a>
## 10. Parcel::growData()

    status_t Parcel::growData(size_t len)
    {
        size_t newSize = ((mDataSize+len)*3)/2;
        return (newSize <= mDataSize)
                ? (status_t) NO_MEMORY
                : continueWrite(newSize);
    }

说明：Parcel增加容量时，是按1.5倍进行增长。mDataSize=0，而len=4；因此会执行continueWrite(6)。  



<a name="anchor3_11"></a>
## 11. Parcel::continueWrite()

    status_t Parcel::continueWrite(size_t desired)
    {
        size_t objectsSize = mObjectsSize;

        ...

        if (mOwner) {
            ...
        } else if (mData) {
            ...

            // We own the data, so we can just do a realloc().
            if (desired > mDataCapacity) {
                uint8_t* data = (uint8_t*)realloc(mData, desired);
                if (data) {
                    mData = data;
                    mDataCapacity = desired;
                } else if (desired > mDataCapacity) {
                    ...
                }
            } else {
                ...
            }

        } else {
            ...
        }

        return NO_ERROR;
    }

说明：mObjectsSize的初始值为0，mOwner的初始值为NULL，mData非空；并且，desired=6，mDataCapacity=0。因此，会调用realloc()给mData重新分配内存大小为6字节。分配成功后，更新"数据地址mData"和"数据容量mDataCapacity=6"。  


接下来，回到writeAligned()中，它会跳转到restart_write标签处。先将int32_t的整形数保存到mData中，然后再调用finishWrite()进行同步。


<a name="anchor3_12"></a>
## 12. Parcel::finishWrite()

    status_t Parcel::finishWrite(size_t len)
    {
        mDataPos += len;
        if (mDataPos > mDataSize) {
            mDataSize = mDataPos;
            ...
        }
        return NO_ERROR;
    }

说明：前面已经将数据写入到mData中，现在就通过finishWrite()来改变数据的当前指针位置(方便下一次写入)和数据的大小。  
(01) len是int32_t的大小，很显然是4个字节，len=4。所以，mDataPos=4。  
(02) mDataPos=4，mDataSize=0；因此if(mDataPos>mDataSize)为true，所以，mDataSize=4。  

此时，就分析完了writeInterfaceToken()中的writeInt32()就分析完毕了.    
**mData**：它的第0~3个字节保存了int32_t类型的数据STRICT_MODE_PENALTY_GATHER。  
**mDataPos**：值为4，即下一个写入mData中的数据从第4个字节开始。  
**mDataSize**：值为4，即mData中数据的大小。   
**mDataCapacity**：值为6，即mData的数据容量为6字节。   
此时，mData的数据如下图所示：

<a href="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/binder/parcel_add01.jpg"><img src="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/binder/parcel_add01.jpg" alt="" /></a>

接下来，看看再writeString16("android.os.IServiceManager")如何将字符串写入到Parcel中。


<a name="anchor3_13"></a>
## 13. Parcel::writeString16()

    status_t Parcel::writeString16(const String16& str)
    {
        return writeString16(str.string(), str.size());
    }
        
    status_t Parcel::writeString16(const char16_t* str, size_t len)
    {
        if (str == NULL) return writeInt32(-1);

        // 将字符串长度写入到Parcel中
        status_t err = writeInt32(len);
        if (err == NO_ERROR) {
            len *= sizeof(char16_t);
            // 在将字符串写入之前，增加mData的容量
            uint8_t* data = (uint8_t*)writeInplace(len+sizeof(char16_t));
            if (data) {
                // 将字符串拷贝到mData中
                memcpy(data, str, len);
                // 字符串结束符
                *reinterpret_cast<char16_t*>(data+len) = 0;
                return NO_ERROR;
            }
            err = mError;
        }
        return err;
    }

说明：writeString16()是重载函数。   
(01) writeString16(str, len)中，str="android.os.IServiceManager"；len是由str.size()得来，虽然这里的字符串是String16类型(即每个字符占2个字节)，但是str.size()是获取str中有效数据的个数(不包含字符串结束符)，因此，len=26。  
(02) 首先调用writeInt32(len)将字符串的长度写入到Parcel中，writeInt32()在前面已经介绍过了。当再次写入int32_t类型的数据时，数据容量不够，会再次增长为12，即mDataCapacity=12；而写入int32_t类型的数据之后，mDataPos和mDataSize都增长为8。 此时，mData的数据如下图所示：

<a href="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/binder/parcel_add02.jpg"><img src="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/binder/parcel_add02.jpg" alt="" /></a>

在通过writeInt32(len)写入数据长度之后，再重新计算len=52；接着，通过writeInplace()写入数据。  



<a name="anchor3_14"></a>
## 14. Parcel::writeInplace()

    #define PAD_SIZE(s) (((s)+3)&~3)

    void* Parcel::writeInplace(size_t len)
    {   
        // 4字节对齐
        const size_t padded = PAD_SIZE(len);

        ...
                
        if ((mDataPos+padded) <= mDataCapacity) {
    restart_write:                        
            uint8_t* const data = mData+mDataPos;
            
            // 如果padded!=len，则根据大端法还是小端法进行地址对齐设置。
            if (padded != len) {
                ...
            }
            
            finishWrite(padded);
            return data;
        }   
            
        status_t err = growData(padded);
        if (err == NO_ERROR) goto restart_write;
        return NULL;
    }

说明：参数len=54。  
(01) PAD_SIZE()是4字节对齐的宏，PAD_SIZE(54)=56。   
(02) 函数的初始值为padded=56，mDataPos=8，mDataCapacity=12。因此，会先调用growData(padded)来增加数据容量。growData()在前面已经介绍过；此时，它会将容量mDataCapacity增加至96。  
(03) 接着会跳转到restart_write标签处，然后调用finishWrite(padded)来更新mDataPos和mDataSize。


至此，writeInplace()就分析完了，它的作用就是增加mData的容量，并返回即将写入数据的地址。接着，回到writeString16()中，执行mmap(data, str, len)将数据拷贝到mData中；拷贝完毕之后，设置字符串的结束符为0。


    status_t Parcel::writeString16(const char16_t* str, size_t len)
    {
        if (str == NULL) return writeInt32(-1);

        // 将字符串长度写入到Parcel中
        status_t err = writeInt32(len);
        if (err == NO_ERROR) {
            len *= sizeof(char16_t);
            // 在将字符串写入之前，增加mData的容量
            uint8_t* data = (uint8_t*)writeInplace(len+sizeof(char16_t));
            if (data) {
                // 将字符串拷贝到mData中
                memcpy(data, str, len);
                // 字符串结束符
                *reinterpret_cast<char16_t*>(data+len) = 0;
                return NO_ERROR;
            }
            err = mError;
        }
        return err;
    }

<br/>
这样，data.writeInterfaceToken(IServiceManager::getInterfaceDescriptor())就分析完了。此时，mData中数据如下图所示：

<a href="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/binder/parcel_add03.jpg"><img src="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/binder/parcel_add03.jpg" alt="" /></a>


<a name="anchor3_15"></a>
## 15. Parcel::writeString16()

继续回到addService()中，接着会通过data.writeString16(name)将MediaPlayerService服务的名称写入到data中，此处的name="media.player"。在前面已经详细介绍过writeString16()，这里执行完该语句后，mData中的数据如下：

<a href="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/binder/parcel_add04.jpg"><img src="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/binder/parcel_add04.jpg" alt="" /></a>


接着，addService()会调用data.writeStrongBinder(service)将MediaPlayerService对象写入到data中。这个数据最重要，下面分析下writeStrongBinder()的实现。  

<a name="anchor3_16"></a>
## 16. Parcel::writeStrongBinder()

    status_t Parcel::writeStrongBinder(const sp<IBinder>& val)
    {
        return flatten_binder(ProcessState::self(), val, this);
    }

说明：该函数调用flatten_binder()将数据打包。


<a name="anchor3_17"></a>
## 17. Parcel::flatten_binder()

    status_t flatten_binder(const sp<ProcessState>& proc,
        const sp<IBinder>& binder, Parcel* out)
    {       
        flat_binder_object obj;
                
        obj.flags = 0x7f | FLAT_BINDER_FLAG_ACCEPTS_FDS;
        if (binder != NULL) {
            IBinder *local = binder->localBinder();
            if (!local) {
                ...
            } else {
                obj.type = BINDER_TYPE_BINDER;
                obj.binder = local->getWeakRefs();
                obj.cookie = local;
            }       
        } else {
            ...
        }
            
        return finish_flatten_binder(binder, obj, out);
    }

说明：该函数是将MediaPlayerService对象封装到结构体flat_binder_object中。Binder驱动认识flat_binder_object结构体类型的数据，在C++层将数据发送给Binder驱动后，Binder驱动能够解析该结构体。  
(01) 先看看参数，proc是ProcessState对象，binder是MediaPlayerService对象，out是Parcel自己。  
(02) binder不为NULL，因此，执行if(binder!=NULL)中的语句。binder->localBinder()返回的BBinder对象，即本地Binder对象。(BBinder是MediaPlayerService的父类，localBinder()函数在frameworks/native/libs/binder/Binder.cpp中实现)。因此，local不为NULL。  

    obj.flags = 0x7f | FLAT_BINDER_FLAG_ACCEPTS_FDS;    // 标记
    obj.type = BINDER_TYPE_BINDER;                      // 类型
    obj.binder = local->getWeakRefs();                  // MediaPlayerService的弱引用
    obj.cookie = local;                                 // MediaPlayerService自身

注意：从这里就可以看出，MediaPlayerService添加服务时，发送给驱动的数据是MediaPlayerService的本地Binder对象，即BBinder实例。准确的来说，该数据是保存在obj.cookie中的，该数据的类型是BINDER_TYPE_BINDER。

(03) 调用finish_flatten_binder()将数据写入到Parcel中。



<a name="anchor3_18"></a>
## 18. Parcel::finish_flatten_binder()

    inline static status_t finish_flatten_binder(
        const sp<IBinder>& binder, const flat_binder_object& flat, Parcel* out)
    {       
        return out->writeObject(flat, false);
    }       

说明：该函数是flat_binder_object对象写入到Parcel中。 


<a name="anchor3_19"></a>
## 19. Parcel::writeObject()

    status_t Parcel::writeObject(const flat_binder_object& val, bool nullMetaData)
    {   
        const bool enoughData = (mDataPos+sizeof(val)) <= mDataCapacity;
        const bool enoughObjects = mObjectsSize < mObjectsCapacity;
        if (enoughData && enoughObjects) {
    restart_write:
            *reinterpret_cast<flat_binder_object*>(mData+mDataPos) = val;
            
            // val.binder非空
            if (nullMetaData || val.binder != NULL) {
                // 将地址偏移位置保存到mObjects[0]中
                mObjects[mObjectsSize] = mDataPos;
                acquire_object(ProcessState::self(), val, this);
                // 增加mObjectsSize的值
                mObjectsSize++;
            }
            
            ...

            return finishWrite(sizeof(flat_binder_object));
        }
        
        if (!enoughData) {
            const status_t err = growData(sizeof(val));
            if (err != NO_ERROR) return err;
        }
        if (!enoughObjects) {
            // 增加容量
            size_t newSize = ((mObjectsSize+2)*3)/2;
            // 分配内存
            size_t* objects = (size_t*)realloc(mObjects, newSize*sizeof(size_t));
            if (objects == NULL) return NO_MEMORY;
            // 设置mObjects的内存地址起始地址
            mObjects = objects;
            // 设置mObjects对象的容量
            mObjectsCapacity = newSize;
        }
                
        goto restart_write;
    }

说明：
(01) 此时，mDataPos=96, sizeof(val)=32, mDataCapacity=96；因此，enoughData=false。mObjectsSize和mObjectsCapacity的初始值=0，因此，enoughObjects=false。  
(02) 首先，执行if(!enoughData)部分，通过growData()将数据的容量增加至192。即，mDataCapacity=192。    
(03) 接着，执行if(!enoughObjects)部分，该部分的目的是分配对象空间，并修改mObjects和mObjectsCapacity的值。增加之后的容量mObjectsCapacity=3。  
(04) 然后，跳转到restart_write标签处。 *reinterpret_cast<flat_binder_object*>(mData+mDataPos) = val是保存val对象到mDataPos+mDataPos所指的地址中。  
(04) mObjects[mObjectsSize]=mDataPos，此处的mObjectsSize=0；这里是将对象的地址偏移mDataPos保存到mObjects[0]中。随后执行mObjectsSize++增加mObjectsSize的值为1。  
(05) 最后，调用finishWrite()更新mDataPos和mDataSize的值。

<br/>至此，data.writeStrongBinder()就分析完了。将MediaPlayerService写入data之后，它的数据如下图所示：

<a href="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/binder/parcel_add05.jpg"><img src="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/binder/parcel_add05.jpg" alt="" /></a>


最后，调用data.writeInt32(allowIsolated ? 1 : 0)。allowIsolated为false，因此，data.writeInt32(0)。执行该函数之后，data的数据如下图所示：

<a href="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/binder/parcel_add06.jpg"><img src="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/binder/parcel_add06.jpg" alt="" /></a>

以上就是addService()中的data的数据。接下来执行remote()->transact(ADD_SERVICE_TRANSACTION, data, &reply)。前面已经说过，remote()返回的是BpBinder对象，该BpBinder对象是在[Android Binder机制(四) defaultServiceManager()的实现][link_binder_04_defaultServiceManager]中调用defaultServiceManager()时初始化的。下面查看BpBinder的transact()。




<a name="anchor3_20"></a>
## 20. BpBinder::transact()

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

说明：该代码在frameworks/native/libs/binder/BpBinder.cpp中。由于mAlive的初始值为1，因此该函数会调用IPCThreadState::self()->transact()。我们知道，IPCThreadState::self()是获取全局IPCThreadState对象，因此最终会调用IPCThreadState::transact()。



<a name="anchor3_21"></a>
## 21. IPCThreadState::transact()


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

说明：该代码在frameworks/native/libs/binder/IPCThreadState.cpp中。  
(01) 先看看函数的参数。handle是BpBinder中的mHandle对象，BpBinder中的mHandle是ServiceManager的句柄，值为0。code=ADD_SERVICE_TRANSACTION。data就是在addService中设置的Parcel对象。reply是用来接收Binder驱动反馈数据的Parcel对象。flags是默认值0。  
(02) 该函数会先通过writeTransactionData()将数据打包。  
(03) flags的初始化为0，并且reply非空。因此，将数据打包号之后，会调用waitForResponse()将数据发送给Binder驱动，然后等待Binder驱动反馈。




<a name="anchor3_22"></a>
## 22. IPCThreadState::writeTransactionData()

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


说明：该函数会读取Parcel中的数据，然后将其打包到binder_transaction_data结构体中。binder_transaction_data结构体是Binder驱动能够识别并对之进行解析的数据结构。   
  ipcDataSize()是返回mDataSize，ipcData()是返回mData，ipcObjectsCount()是返回mObjectsSize，而ipcObjects则是返回mObjects。这些数据就是前面我们在addService中分析的Parcel对象的数据。下面给出初始化之后tr的值。  

    tr.target.handle = handle;  // 0，即Service Manager的句柄
    tr.code = code;             // ADD_SERVICE_TRANSACTION
    tr.flags = binderFlags;     // TF_ACCEPT_FDS
    tr.cookie = 0;
    tr.sender_pid = 0;

    tr.data_size = data.ipcDataSize();      // 数据大小(对应mDataSize)
    tr.data.ptr.buffer = data.ipcData();    // 数据的起始地址(对应mData)
    tr.offsets_size = data.ipcObjectsCount()*sizeof(size_t); // data中保存的对象个数(对应mObjectsSize)
    tr.data.ptr.offsets = data.ipcObjects();                 // data中保存的对象的偏移地址数组(对应mObjects)

初始化tr之后，将cmd=BC_TRANSACTION和tr重新打包到mOut中。mOut中的数据将来会被以请求的方式发送给Binder驱动。重新打包后的数据如下图所示：

<a href="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/binder/data_01.jpg"><img src="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/binder/data_01.jpg" alt="" /></a>

在上图中，mOut包含了"事务指令"+"binder_transaction_data"结构体对象。而具体的MediaPlayerService对象，则包含在binder_transaction_data的data数据区域；它是被封装在flat_binder_object结构体中的。




<a name="anchor3_23"></a>
## 23. IPCThreadState::waitForResponse()

writeTransactionData()分析完毕之后，再看看waitForResponse()的代码。

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




<a name="anchor3_24"></a>
## 24. IPCThreadState::talkWithDriver()

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

说明：talkWithDriver()会先初始化bwr(binder_write_read类型的变量)，然后将bwr变量通过ioctl()发送给Binder驱动。该函数的参数doReceive的默认值为true。  
(01) 现在，mIn中还没有被写入数据，因此它的值都是初始值。那么，mIn.dataPosition()返回mDataPos，它的值为0；mIn.dataSize()返回mDataSize，它的初始值也为0。因此，needRead=true。  
(02) doReceive=true，但是needRead=true；因此，outAvail=mOut.dataSize，outAvail不为0。接下来，就对bwr进行初始化，关于bwr的介绍，请参考[Android Binder机制(二) Binder中的数据结构][link_binder_02_datastruct]。bwr初始化完毕之后，各个成员的值如下：  

    bwr.write_size = outAvail;                          // mOut中数据大小，大于0
    bwr.write_buffer = (long unsigned int)mOut.data();  // mOut中数据的地址
    bwr.write_consumed = 0;
    bwr.read_size = mIn.dataCapacity();                 // 256
    bwr.read_buffer = (long unsigned int)mIn.data();    // mIn.mData，实际上为空
    bwr.read_consumed = 0;

(03) bwr初始化完成之后，调用ioctl(,BINDER_WRITE_READ,)和Binder驱动进行交互。

通过binder_write_read再次打包后的数据如下图所示：

<a href="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/binder/data_02.jpg"><img src="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/binder/data_02.jpg" alt="" /></a>

如上图所示，ioctl()传输的数据包含"BINDER_WRITE_READ"+"binder_write_read结构体对象"。在binder_write_read的write_buffer中包含了事务数据；而在数据数据的data中又包含了flat_binder_object等数据。在flat_binder_object中就包含了需要传输的MediaPlayerService对象。  
总体来看，数据经过了三次封装。下面看看在Binder驱动中是如何一层层将它们剖析开来的。


<a name="anchor3_25"></a>
## 25. Binder驱动中binder_ioctl()的BINDER_WRITE_READ相关部分的源码

    static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
    {
      int ret;
      struct binder_proc *proc = filp->private_data;
      struct binder_thread *thread;
      unsigned int size = _IOC_SIZE(cmd);
      void __user *ubuf = (void __user *)arg;

      // 中断等待函数。
      // 1. 当binder_stop_on_user_error < 2为true时；不会进入等待状态；直接跳过。
      // 2. 当binder_stop_on_user_error < 2为false时，进入等待状态。
      //    当有其他进程通过wake_up_interruptible来唤醒binder_user_error_wait队列，并且binder_stop_on_user_error < 2为true时；
      //    则继续执行；否则，再进入等待状态。
      ret = wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);

      binder_lock(__func__);
      // 在proc进程中查找该线程对应的binder_thread；若查找失败，则新建一个binder_thread，并添加到proc->threads中。
      thread = binder_get_thread(proc);
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

说明：关于该函数在[Android Binder机制(三) ServiceManager守护进程][link_binder_03_ServiceManagerDeamon]中已经介绍过了。这里将binder_write_read从用户空间拷贝到内核空间之后，读取bwr.write_size和bwr.read_size都>0，因此先写后读。



<a name="anchor3_26"></a>
## 26. Binder驱动中binder_thread_write()的源码

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

说明：读取出来的交易码是BC_TRANSACTION。在通过copy_from_user()将数据拷贝从用户空间拷贝到内核空间之后，就调用binder_transaction()进行处理。  



<a name="anchor3_27"></a>
## 27. Binder驱动中binder_transaction()的源码

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
                // 事务目标对象是ServiceManager的binder实体
                // 即，该事务是交给Service Manager来处理的。
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

        // 设置from，表示该事务是MediaPlayerService发起的
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
        // tr->data.ptr.offsets就是数据中的对象偏移地址数组，tr->offsets_size就数据中的对象个数
        // 拷贝之后，offp就是flat_binder_object对象数组在内核空间的偏移数组的起始地址
        if (copy_from_user(offp, tr->data.ptr.offsets, tr->offsets_size)) {
            ...
        }
        ...
        // off_end就是flat_binder_object对象数组在内核空间的偏移地址的结束地址
        off_end = (void *)offp + tr->offsets_size;
        // 将所有的flat_binder_object对象读取出来
        // 对MediaPlayerService而言，只有一个flat_binder_object对象。
        for (; offp < off_end; offp++) {
            struct flat_binder_object *fp;
            ...
            fp = (struct flat_binder_object *)(t->buffer->data + *offp);

            switch (fp->type) {
            case BINDER_TYPE_BINDER:
            case BINDER_TYPE_WEAK_BINDER: {
                struct binder_ref *ref;
                // 在proc中查找binder实体对应的binder_node
                struct binder_node *node = binder_get_node(proc, fp->binder);
                // 若找不到，则新建一个binder_node；下次就可以直接使用了。
                if (node == NULL) {
                    node = binder_new_node(proc, fp->binder, fp->cookie);
                    if (node == NULL) {
                        ...
                    }
                    node->min_priority = fp->flags & FLAT_BINDER_FLAG_PRIORITY_MASK;
                    node->accept_fds = !!(fp->flags & FLAT_BINDER_FLAG_ACCEPTS_FDS);
                }
                ...
                // 在target_proc(即，ServiceManager的进程上下文)中查找是否包行"该Binder实体的引用"，
                // 如果没有找到的话，则将"该binder实体的引用"添加到target_proc->refs_by_node红黑树中。这样，就可以通过Service Manager对该
    Binder实体进行管理了。
                ref = binder_get_ref_for_node(target_proc, node);
                if (ref == NULL) {
                    ...
                }
                // 修改type
                if (fp->type == BINDER_TYPE_BINDER)
                    fp->type = BINDER_TYPE_HANDLE;
                else
                    fp->type = BINDER_TYPE_WEAK_HANDLE;
                // 修改handle。handle和binder是联合体，这里将handle设为引用的描述。
                // 根据该handle可以找到"该binder实体在target_proc中的binder引用"；
                // 即，可以根据该handle，可以从Service Manager找到对应的Binder实体的引用，从而获取Binder实体。
                fp->handle = ref->desc;
                // 增加引用计数，防止"该binder实体"在使用过程中被销毁。
                binder_inc_ref(ref, fp->type == BINDER_TYPE_HANDLE,
                           &thread->todo);

                trace_binder_transaction_node_to_ref(t, node, ref);
                ...
            } break;
            ...
            }
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

说明：这里的tr->target.handle=0，因此，会设置target_node为ServiceManager对应的Binder实体。下面是target_node,target_proc等值初始化之后的值。  

    target_node = binder_context_mgr_node; // 目标节点为Service Manager对应的Binder实体
    target_proc = target_node->proc;       // 目标进程为Service Manager对应的binder_proc进程上下文信息
    target_list = &target_thread->todo;    // 待处理事务队列
    target_wait = &target_thread->wait;    // 等待队列

目标节点是Service Manager对应的Binder实体。这是指MediaPlayerService的addService()这个指令是来提交给Service Manager进行处理的，它最终会发送给Service Manager进行处理。。


在初始化完target_node等目标节点之后，会新建一个待处理事务t和待完成的工作tcomplete，并对它们进行初始化。待处理事务t会被提交给目标(即ServiceManager对应的Binder实体)进行处理；而待完成的工作tcomplete则是为了反馈给MediaPlayerService服务，告诉MediaPlayerService它的请求Binder驱动已经收到了。注意，这里仅仅是告诉MediaPlayerService该请求已经被收到，而不是处理完毕！待ServiceManager处理完毕该请求之后，Binder驱动会再次反馈相应的消息给MediaPlayerService。

        // 分配一个待处理的事务t，t是binder事务(binder_transaction对象)
        t = kzalloc(sizeof(*t), GFP_KERNEL);
        ...

        // 分配一个待完成的工作tcomplete，tcomplete是binder_work对象。
        tcomplete = kzalloc(sizeof(*tcomplete), GFP_KERNEL);
        ...

        t->debug_id = ++binder_last_id;
        ...

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


在初始化完待处理事务t之后，接着将MediaPlayerService请求的数据拷贝到内核空间并解析出来。从数据中解析出MediaPlayerService请求数据中的flat_binder_object对象，只有一个flat_binder_object对象。该flat_binder_object对象的类型是BINDER_TYPE_BINDER，然后调用binder_get_node()在当前进程的上下文环境proc中查找fp->binder对应的Binder实体，fp->binder是Android的flatten_binder()中赋值的，它是MediaPlayerService对象的本地引用的描述(即MediaPlayerService对应的BBinder对象的描述)；此外，在MediaPlayerService是初次与Binder驱动通信，因此肯定找不到该对象fp->binder对应的Binder实体；因此node=NULL。  接下来，就调用binder_new_node()新建fp->binder对应的Binder实体，这也就是MediaPlayerService对应的Binder实体。然后，调用binder_get_ref_for_node(target_proc, node)获取该Binder实体在target_proc(即ServiceManager的进程上下文环境)中的Binder引用，此时，在target_proc中肯定也找不到该Binder实体对应的引用；那么，就新建Binder实体的引用，并将其添加到target_proc->refs_by_node红黑树 和 target_proc->refs_by_desc红黑树中。 这样，Service Manager的进程上下文中就存在MediaPlayerService的Binder引用，Service Manager也就可以对MediaPlayerService进行管理了！  
  然后，修改fp->type=BINDER_TYPE_HANDLE，并使fp->handle = ref->desc。

这样，就将MediaPlayerService的请求数据解析出来，并且在Binder驱动中创建了MediaPlayerService对应的Binder实体，而且将该Binder实体添加到MediaPlayerService的进程上下文proc中。更重要的是，在ServiceManager的refs_by_node和refs_by_desc这两颗红黑树中创建了"MediaPlayerService对应的Binder实体的Binder引用"。这意味着，在Binder驱动中，已经能在ServiceManager的进程上下文中找到MediaPlayerService。


        // 将"用户空间的数据"拷贝到内核中
        // tr->data.ptr.buffer就是用户空间数据的起始地址，tr->data_size就是数据大小
        if (copy_from_user(t->buffer->data, tr->data.ptr.buffer, tr->data_size)) {
            ...
        }
        // 将"用户空间的数据中所含对象的偏移地址"拷贝到内核中
        // tr->data.ptr.offsets就是数据中的对象偏移地址数组，tr->offsets_size就数据中的对象个数
        // 拷贝之后，offp就是flat_binder_object对象数组在内核空间的起始地址
        if (copy_from_user(offp, tr->data.ptr.offsets, tr->offsets_size)) {
            ...
        }
        ...
        // off_end就是flat_binder_object对象数组在内核空间的结束地址
        off_end = (void *)offp + tr->offsets_size;
        // 将所有的flat_binder_object对象读取出来
        // 对MediaPlayerService而言，只有一个flat_binder_object对象。
        for (; offp < off_end; offp++) {
            struct flat_binder_object *fp;
            ...
            fp = (struct flat_binder_object *)(t->buffer->data + *offp);

            switch (fp->type) {
            case BINDER_TYPE_BINDER:
            case BINDER_TYPE_WEAK_BINDER: {
                struct binder_ref *ref;
                // 在proc中查找binder实体对应的binder_node
                struct binder_node *node = binder_get_node(proc, fp->binder);
                // 若找不到，则新建一个binder_node；下次就可以直接使用了。
                if (node == NULL) {
                    node = binder_new_node(proc, fp->binder, fp->cookie);
                    if (node == NULL) {
                        ...
                    }
                    node->min_priority = fp->flags & FLAT_BINDER_FLAG_PRIORITY_MASK;
                    node->accept_fds = !!(fp->flags & FLAT_BINDER_FLAG_ACCEPTS_FDS);
                }
                ...
                // 在target_proc(即，Service Manager的进程上下文)中查找是否包行"该binder实体的引用"，
                // 如果没有找到的话，则将"该binder实体的引用"添加到target_proc->refs_by_node红黑树中。这样，就可以通过Service Manager对该
    Binder实体进行管理了。
                ref = binder_get_ref_for_node(target_proc, node);
                if (ref == NULL) {
                    ...
                }
                // 修改type
                if (fp->type == BINDER_TYPE_BINDER)
                    fp->type = BINDER_TYPE_HANDLE;
                else
                    fp->type = BINDER_TYPE_WEAK_HANDLE;
                // 修改handle。handle和binder是联合体，这里将handle设为引用的描述。
                // 根据该handle可以找到"该binder实体在target_proc中的binder引用"；
                // 即，可以根据该handle，可以从Service Manager找到对应的binder实体的引用，从而获取binder实体。
                fp->handle = ref->desc;
                // 增加引用计数，防止"该binder实体"在使用过程中被销毁。
                binder_inc_ref(ref, fp->type == BINDER_TYPE_HANDLE,
                           &thread->todo);

                trace_binder_transaction_node_to_ref(t, node, ref);
                ...
            } break;
            ...
            }
        }


然后，设置待处理事务的类型为BINDER_WORK_TRANSACTION，并将其添加到target_list中。即，添加事务到Service Manager对应的待处理事务队列中。  
设置待完成工作的类型为BINDER_WORK_TRANSACTION_COMPLETE，并将其添加到当前线程的待完成工作中。此时，Binder驱动已经收到了MediaPlayerService的请求，这个所谓的待完成工作，就是用来让Binder驱动告诉MediaPlayerService，它的请求已经被处理了。  
最后，target_wait是ServiceManager的等待队列，肯定不为空(因为前面刚刚将BINDER_WORK_TRANSACTION事务添加到待处理事务中)。因此，便会执行wake_up_interruptible(target_wait)唤醒Service Manager进程。  
**注意**，此时都是运行在MediaPlayerService的进程中的！


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

此时，MediaPlayerService进程还会继续运行，而且它也通过wake_up_interruptible()唤醒了ServiceManager进程。ServiceManager被唤醒后，所做的工作就是将MediaPlayerService注册到它的服务队列中进行管理；它的具体流程稍候再分析，现在还是先分析完MediaPlayerService进程。


至此，binder_transaction()就分析完了。在binder_transaction()中，我们主要进行了以下工作：
(01) 解析出来MediaPlayerService的请求数据。  
(02) 新建MediaPlayerService对应的Binder实体和Binder引用，并将ServiceManager的进程上下文中存在MediaPlayerService的Binder引用。  
(03) 新建了待处理事务，并将该事务添加到了ServiceManager的待处理事务队列中。然后，唤醒ServiceManager来处理该事务。    
(04) 新建了待完成工作，并将待完成工作添加到了当前线程的待完成工作队列中。  


<a name="anchor3_28"></a>
## 28. Binder驱动中binder_thread_write()的源码

接着分析MediaPlayerService进程的工作。binder_thread_write()中执行binder_transaction()后，会更新*consumed的值，即bwr.write_consumed的值。意味着，Binder驱动已经驱动完成MediaPlayerService的请求数据。

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



<a name="anchor3_29"></a>
## 29. Binder驱动中binder_thread_read()的源码

接下来，ioctl()会执行binder_thread_read()来设置反馈数据给MediaPlayerService进程。  

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

                binder_stat_br(proc, thread, cmd);
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
(01) 先看看函数的参数，buffer是bwr.read_buffer，是反馈数据缓冲区。size是bwr.read_size，是缓冲区大小，为256字节；而consumed是指向bwr.read_consumed的，它的值是0，表示反馈数据还没有被MediaPlayerService读取过。non_block为0。  
(02) *consumed=0，因此会先将BR_NOOP从内核空间拷贝到用户空间，即拷贝到bwr.read_buffer中。  
(03) 在binder_transaction()中，我们有添加待完成工作到thread的待完成工作队列中。因此，wait_for_proc_work是false。  
(04) binder_has_thread_work(thread)为ture，因此wait_event_interruptible()不会进入中断等待状态，而是继续往下运行。  
(05) 接着，进入while循环。list_empty(&thread->todo)为flase，执行list_first_entry(&thread->todo, struct binder_work, entry)从thread的待完成工作队列中取出待完成的工作t。  
(06) 根据binder_transaction()中的分析可知，t->type的值为BINDER_WORK_TRANSACTION_COMPLETE。执行对应的case分支，会将数据cmd=BR_TRANSACTION_COMPLETE拷贝到用户空间，即bwr.read_buffer中。拷贝之后，即代表该工作已完成，然后从当前线程的工作队列中将该工作删除，并释放所分配的空间。  
(07) 由于t=null，因此，会再次从头开始执行while循环。而此时，list_empty(&thread->todo)为true，并且list_empty(&proc->todo)也为true；因此会执行break跳出while循环。  
(08) 在跳出while循环之后，会更新*consumed的值。即，更新bwr.read_consumed的值。此时，由于写入了BR_NOOP和BR_TRANSACTION_COMPLETE两个指令，bwr.read_consumed=8。



<br/>
接下来，回到binder_ioctl()中。将bwr数据拷贝到用户空间后返回。此时，bwr中各个参数的值如下：

    bwr.write_size = outAvail;                          
    bwr.write_buffer = (long unsigned int)mOut.data();
    bwr.write_consumed = outAvail;                      // 等于write_size
    bwr.read_size = mIn.dataCapacity();
    bwr.read_buffer = (long unsigned int)mIn.data();    // 存储了BR_NOOP和BR_TRANSACTION_COMPLETE两个返回指令
    bwr.read_consumed = 8;                              // 等于write_size

bwr中的write_*参数是保存"MediaPlayerService发送给Binder驱动的请求内容的"，而read_*则是保存"Binder驱动反馈给MediaPlayerService的内容的"。此时，write_consumed和write_size相同，意味着"Binder驱动已经将请求的内容都处理完毕了"；而read_consumed>0，则意味着"Binder驱动有反馈内容给MediaPlayerService"。  
回到talkWithDriver()中，看看ioctl()之后做了些什么？



<a name="anchor3_30"></a>
## 30. IPCThreadState::talkWithDriver()

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

说明：ioctl()返回值为0，err=NO_ERROR，退出while循环。  
(01) bwr.write_consumed>0，并且bwr.write_consumed=mOut.dataSize。因此，调用mOut.setDataSize(0)将释放mOut的内存，并且将mOut的mDataSize和mObjectsSize设为0。  
(02) bwr.read_consumed>0，因此调用mIn.setDataSize()为mIn分配空间，并将mIn的mDataSize设为=bwr.read_consumed。然后，将位置mDataPos初始化为0。  
之后，跳出talkWithDriver()，返回到waitForResponse()中。


<a name="anchor3_31"></a>
## 31. IPCThreadState::waitForResponse()

    status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
    {       
        int32_t cmd;
        int32_t err;

        while (1) {
            // 先通过talkWithDriver()和Binder驱动交互
            if ((err=talkWithDriver()) < NO_ERROR) break;
            err = mIn.errorCheck();
            if (err < NO_ERROR) break;
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

说明：从talkWithDriver()正常返回之后，会读取mIn中的数据。而mIn中的数据就是Binder驱动返回的"BR_NOOP和BR_TRANSACTION_COMPLETE两个指令"。先读出的指令是BR_NOOP，因此这里执行executeCommand(cmd)。


<a name="anchor3_32"></a>
## 32. IPCThreadState::executeCommand()

    status_t IPCThreadState::executeCommand(int32_t cmd)
    {
        BBinder* obj;
        RefBase::weakref_type* refs;
        status_t result = NO_ERROR;

        switch (cmd) {
        case BR_ERROR:
            ...
        case BR_OK:
            ...
        case BR_NOOP:
            break;
        default:
            ...
        }

        if (result != NO_ERROR) {
            mLastError = result;
        }

        return result;
    }

说明：BR_NOOP没有进行任何操作，直接返回。继续回到waitForResponse()中，重新开始while循环，执行talkWithDriver()。




<a name="anchor3_33"></a>
## 33. IPCThreadState::talkWithDriver()

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

说明：
(01) 此时，因为在waitForResponse()中已经通过mIn.readInt32()读取了4个字节，因此mIn.dataPosition()=4，而mIn.dataSize()=8；因此，needRead=false。  
(02) needRead=false，而doReceive=true；因此，outAvail=0。  
最终，由于 bwr.write_size和bwr.read_size都为0，因此直接返回NO_ERROR。

再次回到waitForResponse()中，此时读出的cmd为BR_TRANSACTION_COMPLETE。此时，由于reply不为NULL，因此再次重新执行while循环，调用talkWithDriver()。

(01) 此时，已经读取了mIn中的全部数据，因此mIn.dataPosition()=8，而mIn.dataSize()=8；因此，needRead=true。  
(02) outAvail=mOut.dataSize()，前面已经将mOut清空，因此outAvail=0。bwr初始化完毕之后，各个成员的值如下：

        bwr.write_size = 0;
        bwr.write_buffer = (long unsigned int)mOut.data();
        bwr.write_consumed = 0;
        bwr.read_size = mIn.dataCapacity();                 // 256字节
        bwr.read_buffer = (long unsigned int)mIn.data();
        bwr.read_consumed = 0;

其实，此时MediaPlayerService已经处理完"addService()这个请求，包括已经处理完了该请求的反馈"。对MediaPlayerService而言，它已经成功的注册到Service Manager中；接下来，就是等待Client的请求了。  
那么如何去等待Client的请求呢？这和前面分析Service Manager服务启动之后等待Client的请求类似。MediaPlayerService服务，会通过ioctl()给Binder驱动发送读写请求，而此时的bwr.write_size=0，意味着不会进行写；bwr.read_size>0，意味着会进行读。这样，Binder驱动就会执行读取动作，进而去查看"MediaPlayerService在Binder驱动中的待处理事务队列"是否有事务需要处理，有的话，就进行事务处理；否则，就进入中断等待状态，等待Client的请求。

下面，看看它到底是如何做到的。


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

此时，bwr.write_size=0，因此不会执行binder_thread_write()。而bwr.read_size>0，因此会调用binder_thread_read()进行读取动作。


<a name="anchor3_34"></a>
## 34. Binder驱动中binder_thread_read()的源码

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
          // 修改指针位置
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
          // 设置当前线程的优先级=proc->default_priority。
          // 即，当前线程要处理proc的事务，所以设置优先级和proc一样。
          binder_set_nice(proc->default_priority);
          if (non_block) {
              ...
          } else
              // 阻塞式的读取，则阻塞等待事务的发生。
              ret = wait_event_interruptible_exclusive(proc->wait, binder_has_proc_work(proc, thread));
      } else {
          ...
      }
      ...
    }

(01) 此时，bwr.read_consumed=0，意味着*consumed=0。因此，还是会先将BR_NOOP写入到bwr.read_buffer中。  
(02) 此时，当前线程的事务栈和待处理事务队列都是空，因此wait_for_proc_work=true。  
(03) 在调用binder_set_nice()设置当前线程的优先级之后，就会调用wait_event_interruptible()。而此时binder_has_proc_work()为false，因此当前线程会进入中断等待状态。当Service Manager处理完MediaPlayerService的请求之后，就会将其唤醒。


<br>
至此，MediaPlayerService进程的addService的请求发送部分就讲解完了。在继续了解请求的处理之前，先回顾一下本部分的内容。

<a href="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/binder/addService01_send.jpg"><img src="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/binder/addService01_send.jpg" alt="" /></a>

如上图所示，MediaPlayerService发送一个BC_TRANSACTION事务给Binder驱动。Binder驱动收到该事务之后，对请求数据进行解析，在Kernel中新建了MediaPlayerService对应的Binder实体，并将在ServiceManager的进程上下文中添加了该Binder实体的Binder引用。解析完数据之后，新增一个待处理事务并提交到ServiceManager的待处理事务列表中；接着，就唤醒了ServiceManager。与此同时，Binder驱动还反馈了一个BR_TRANSACTION_COMPLETE给MediaPlayerService，告诉MediaPlayerService它的addService请求已经发送成功；MediaPlayerService在解析完BR_TRANSACTION_COMPLETE之后，就进入等待状态，等待ServiceManager的处理完请求之后反馈结果给它。

下面一篇文章，就看看ServiceManager被唤醒之后，具体都做了些什么工作！



[link_binder_01_introduce]: /2014/09/01/Binder-Introduce/
[link_binder_02_datastruct]: /2014/09/02/Binder-Datastruct/
[link_binder_03_ServiceManagerDeamon]: /2014/09/03/Binder-ServiceManager-Daemon/
[link_binder_04_defaultServiceManager]: /2014/09/04/Binder-defaultServiceManager/

