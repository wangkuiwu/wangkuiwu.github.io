---
layout: post
title: "Android Binder机制(五) addService详解01之 请求的发送"
description: "android"
category: android
tags: [android]
date: 2014-09-05 09:01
---


> 本文会介绍Android的消息处理机制。  

> **目录**  
> **1**. [Android消息机制的架构](#anchor1)  

> 注意：本文是基于Android 4.4.2版本进行介绍的！



<a name="anchor1"></a>
# MediaPlayerService的main()函数


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
            ...
            ProcessState::self()->startThreadPool();
            IPCThreadState::self()->joinThreadPool();
        }
    }

说明：该代码在frameworks/av/media/mediaserver/main_mediaserver.cpp中。  
(01) property_get("ro.test_harness", value, "0")是获取"ro.test_harness"属性，为false。  
(02) ProcessState:self()是获取ProcessState对象，并赋值给proc。ProcessState::self()在[skywang-TODO]中已经介绍过了。  
(03) defaultServiceManager()是获取IServiceManager对象，它的实现在[skywang-TODO]中也有详细介绍。  
(04) MediaPlayerService::instantiate()是初始化MediaPlayerService服务。  





<a name="anchor2"></a>
# 2. MediaPlayerService::instantiate()

    void MediaPlayerService::instantiate() {
        defaultServiceManager()->addService(
                String16("media.player"), new MediaPlayerService());
    }

说明：该代码在frameworks/av/media/libmediaplayerservice/MediaPlayerService.cpp中。它会新建MediaPlayerService对象；然后调用defaultServiceManager()获取到的BpServiceManager的实例；最后，调用BpServiceManager的addService()方法，将MediaPlayerService对象添加到Service Manager中。




<a name="anchor3"></a>
# 3. MediaPlayerService::MediaPlayerService()

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



<a name="anchor4"></a>
# 4. BpServiceManager::addService()

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
(02) Parcel是Binder通信的数据存储结构，它的各个成员和函数在[skywang-todo]中有详细说明。下面，我们逐个对data的赋值进行介绍。





<a name="anchor5"></a>
# 5. Parcel::Parcel()

先看看Parcel的构造函数。

    Parcel::Parcel()
    {   
        initState(); 
    }   

说明：该代码在frameworks/native/libs/binder/Parcel.cpp中。  


<a name="anchor6"></a>
# 6. Parcel::initState()

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




<a name="anchor7"></a>
# 7. Parcel::writeInterfaceToken()

    下面看看data.writeInterfaceToken(IServiceManager::getInterfaceDescriptor())的实现。getInterfaceDescriptor()是通过宏IMPLEMENT_META_INTERFACE()实现的，实现比较简单，就不再具体分析了，感兴趣的可参考[skywang-TODO]中的代码；getInterfaceDescriptor()的返回值是"android.os.IServiceManager"。
    即data.writeInterfaceToken("android.os.IServiceManager")。下面看看writeInterfaceToken()的实现。

    status_t Parcel::writeInterfaceToken(const String16& interface)
    {       
        writeInt32(IPCThreadState::self()->getStrictModePolicy() |
                   STRICT_MODE_PENALTY_GATHER);
        // currently the interface identification token is just its name as a string
        return writeString16(interface);
    }   


说明：该函数先通过writeInt32()写入一个32位的int数到Parcel中，然后再通过writeString16()将字符串写入到Parcel中。  
(01) IPCThreadState::self()返回IPCThreadState对象；然后，调用IPCThreadState::getStrictModePolicy()，返回的是mStrictModePolicy，mStrictModePolicy的初始值是0。因此，writeInt32()就可以简化为writeInt32(STRICT_MODE_PENALTY_GATHER)。  
(02) writeString16(interface)是writeString16("android.os.IServiceManager")。




<a name="anchor8"></a>
# 8. Parcel::writeInt32()

    status_t Parcel::writeInt32(int32_t val)
    {   
        return writeAligned(val);
    }   

说明：该函数调用writeAligned()。



<a name="anchor9"></a>
# 9. Parcel::writeAligned()

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

说明：writeAligned()是写入后，并更新相应的变量。  
(01) mDataPos的初始值=0，sizeof(val)=4，mDataCapacity的初始值=0。因此，if((mDataPos+sizeof(val)) <= mDataCapacity)为true。调用*reinterpret_cast<T*>(mData+mDataPos) = val将val赋值到写入到mData中。  
> 简单分析一下该赋值语句，mData+mDataPos中mData是地址起始地址，mDataPos的初始值=0，而T是int32_t类型，因此reinterpret_cast<T*>是将mData+mDataPos转换为int32_t*类型的指针；接下来就是将val赋值给该int32_t*类型的指针所指的地址中。

(02) 将数据写入到mData中之后，调用finishWrite()修改对应的变量。



<a name="anchor10"></a>
# 10. Parcel::finishWrite()

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
**mDataSize**：值为4，即mData中数据的大小。此时，mData的数据如下图所示：[skywang-todo]  

接下来，看看再writeString16("android.os.IServiceManager")将字符串写入到Parcel中。


<a name="anchor12"></a>
# 12. Parcel::writeString16()

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
(01) writeString16(str, len)中，str="android.os.IServiceManager"；len是由str.size()得来，虽然这里的字符串是String16类型(即每个字符占2个字节)，但是str.size()是获取str中有效数据的个数(不包含字符窗结束符)，因此，len=26。  
(02) 首先调用writeInt32(len)将字符串的长度写入到Parcel中。writeInt32()在前面已经介绍过了，这里不再重复说明；在将字符串长度26写入到mData后，会修改mDataPos和mDataSize的值。调用writeInt32(len)之后，mData的数据如下图所示：[skywang-todo]
  
(03) 接着，if(err==NO_ERROR)为true，修改 len的值，sizeof(char16_t)=2；因此len=26*2=52。接着调用writeString16(len+sizeof(char16_t)，即writeString16(54)。




# Parcel::writeInplace()

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

说明：PAD_SIZE()是4字节对齐的宏，PAD_SIZE(54)=56。函数的初始值为padded=56，mDataPos=8，mDataCapacity=0。因此，会先调用growData()增加容量。




# Parcel::growData()

    status_t Parcel::growData(size_t len)
    {
        size_t newSize = ((mDataSize+len)*3)/2;
        return (newSize <= mDataSize)
                ? (status_t) NO_MEMORY
                : continueWrite(newSize);
    }

说明：Parcel增加容量时，是增加为len的1.5倍。这里，len=56，因此，newSize=84。此时，mDataSize=8；故执行continueWrite()。



# Parcel::continueWrite()

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

说明：mObjectsSize的初始值为0，mOwner的初始值为NULL，mData非空；并且，desired=84，mDataCapacity=0。因此，会调用realloc()给mData重新分配内存大小为84字节。分配成功后，重设mData的地址，并且设置mDataCapacity=84。

现在，回到writeInplace()中继续分析，在growData()分配内存之后，就跳转到restart_write标签处。由于之前通过PAD_SIZE对数据进行了字节对齐处理，因此如果padded!=len，则根据大端法/小端法对数据进行调整。调整之后，再调用finishWrite(padded)更新mDataPos和mDataSize的值。更新后的mDataPos=8+84=92，mDataSize=92。

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

至此，writeInplace()就分析完了，它的作用就是增加mData的容量，并返回即将写入数据的地址。继续回到writeString16()中，执行mmap(data, str, len)将数据拷贝到mData中；拷贝完毕之后，设置字符串的结束符为0。
 
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

<br/>这样，data.writeInterfaceToken(IServiceManager::getInterfaceDescriptor())就分析完了。此时，mData中数据如下：[skywang-todo]





继续回到addService()中，接着会通过data.writeString16(name)将MediaPlayerService的名称写入到data中，此处的name="media.player"。执行该语句后，data中的数据如下：[skywang-todo]


接着，addService()会调用data.writeStrongBinder(service)将MediaPlayerService对象写入到data中。这个数据最重要，下面分析下writeStrongBinder()的实现。  

<a name="anchor11"></a>
# 11. Parcel::writeStrongBinder()

    status_t Parcel::writeStrongBinder(const sp<IBinder>& val)
    {
        return flatten_binder(ProcessState::self(), val, this);
    }

说明：该函数调用flatten_binder()将数据打包。


<a name="anchor12"></a>
# 12. Parcel::flatten_binder()

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

说明：该函数是将MediaPlayerService对象封装到结构体flat_binder_object中。Binder驱动认识该结构体，在C++层将数据发送给Binder驱动后，Binder驱动能够解析该结构体。  
(01) 先看看参数，proc是ProcessState对象，binder是MediaPlayerService对象，out是Parcel自己。  
(02) binder不为NULL，因此，执行if(binder!=NULL)中的语句。binder->localBinder()返回BBinder对象(BBinder是MediaPlayerService的父类，localBinder函数在frameworks/native/libs/binder/Binder.cpp中实现)。因此，local不为NULL。  

    obj.flags = 0x7f | FLAT_BINDER_FLAG_ACCEPTS_FDS;    // 标记
    obj.type = BINDER_TYPE_BINDER;                      // 类型
    obj.binder = local->getWeakRefs();                  // MediaPlayerService的弱引用
    obj.cookie = local;                                 // MediaPlayerService自身

(03) 调用finish_flatten_binder()将数据写入到Parcel中。



<a name="anchor13"></a>
# 13. Parcel::finish_flatten_binder()

    inline static status_t finish_flatten_binder(
        const sp<IBinder>& binder, const flat_binder_object& flat, Parcel* out)
    {       
        return out->writeObject(flat, false);
    }       

说明：该函数是flat_binder_object对象写入到Parcel中。




<a name="anchor14"></a>
# 14. Parcel::writeObject()

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
(01) [TODO] mDataPos=96, sizeof(val)=32, mDataCapacity=138；因此，enoughData=true。mObjectsSize和mObjectsCapacity的初始值=0，因此，enoughObjects=false。  
(02) 首先执行if(!enoughObjects)部分，该部分的目的是分配对象空间，并修改mObjects和mObjectsCapacity的值。mObjectsCapacity=3。  
(03) 执行goto restart_write，跳转到restart_write标签处。 *reinterpret_cast<flat_binder_object*>(mData+mDataPos) = val是保存val对象到mDataPos+mDataPos所指的地址中。  
(04) mObjects[mObjectsSize]=mDataPos，此处的mObjectsSize=0；这里是将对象的地址偏移mDataPos保存到mObjects[0]中。随后执行mObjectsSize++增加mObjectsSize的值为1。  
(05) 最后，调用finishWrite()更新mDataPos和mDataSize的值。

<br/>至此，data.writeStrongBinder()就分析完了。将MediaPlayerService写入data之后，它的数据如下所示：[skywang-todo]



最后，调用data.writeInt32(allowIsolated ? 1 : 0)。allowIsolated为false，因此，data.writeInt32(0)。执行该函数之后，data的数据如下所示：[skywang-todo]


以上就是addService()中的data的数据。接下来执行remote()->transact(ADD_SERVICE_TRANSACTION, data, &reply)。前面已经说过，remote()返回的是BpBinder对象，该BpBinder对象是在[TODO]中调用defaultServiceManager()时初始化的。下面查看BpBinder的transact()。




# BpBinder::transact()

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



# IPCThreadState::transact()


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
(01) 先看看函数的参数。handle是BpBinder中的mHandle对象，BpBinder中的mHandle是Service Manager的句柄，值为0。code=ADD_SERVICE_TRANSACTION。data就是在addService中设置的Parcel对象。reply是用户接收反馈数据的Parcel对象。flags是默认值0。  
(02) 该函数会先通过writeTransactionData()将数据打包。  
(03) flags的初始化为0，并且reply非空。因此，将数据打包号之后，会调用waitForResponse()将数据发送给Binder驱动，然后等待Binder驱动反馈。




# IPCThreadState::writeTransactionData()

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
    tr.data.ptr.offsets = data.ipcObjects();    // data中保存的对象的偏移地址数组(对应mObjects)

初始化tr之后，将cmd=BC_TRANSACTION和tr重新打包到mOut中。mOut中的数据将来会被以请求的方式发送给Binder驱动。




# IPCThreadState::waitForResponse()

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




# IPCThreadState::talkWithDriver()

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
(02) doReceive=true，但是needRead=true；因此，outAvail=mOut.dataSize，outAvail不为0。接下来，就对bwr进行初始化，关于bwr的介绍，请参考[skywang-todo]。bwr初始化完毕之后，各个成员的值如下：  

    bwr.write_size = outAvail;                          // mOut中数据大小，大于0
    bwr.write_buffer = (long unsigned int)mOut.data();  // mOut中数据的地址
    bwr.write_consumed = 0;
    bwr.read_size = mIn.dataCapacity();                 // 256
    bwr.read_buffer = (long unsigned int)mIn.data();    // mIn.mData，实际上为空
    bwr.read_consumed = 0;

(03) bwr初始化完成之后，调用ioctl(,BINDER_WRITE_READ,)和Binder驱动进行交互。


## 14. Binder驱动中binder_ioctl()的BINDER_WRITE_READ相关部分的源码

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

说明：关于该函数在[skywang-todo]中已经介绍过了。这里将binder_write_read从用户空间拷贝到内核空间之后，读取bwr.write_size和bwr.read_size都>0，因此先写后读。



<a name="anchor15"></a>
## 15. Binder驱动中binder_thread_write()的源码

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



## 16. Binder驱动中binder_transaction()的源码

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
                // 事务目标对象是Service Manager的binder实体
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

说明：这里的tr->target.handle=0，因此，会设置target_node为Service Manager对应的Binder实体。下面是target_node,target_proc等值初始化之后的值。  

    target_node = binder_context_mgr_node; // 目标节点为Service Manager对应的Binder实体
    target_proc = target_node->proc;       // 目标进程为Service Manager对应的binder_proc进程上下文信息
    target_list = &target_thread->todo;    // 待处理事务队列
    target_wait = &target_thread->wait;    // 等待队列

目标节点是Service Manager对应的Binder实体。这是指MediaPlayerService的addService()这个指令是来提交给Service Manager进行处理的，它最终会发送给Service Manager进行处理。。


在初始化完target_node等目标节点之后，会新建一个待处理事务t和待完成的工作tcomplete，并对它们进行初始化。待处理事务t会被提交给目标(即Service Manager对应的Binder实体)进行处理；而待完成的工作tcomplete则是为了反馈给MediaPlayerService服务，告诉MediaPlayerService它的请求Binder驱动已经处理了。

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


在初始化完待处理事务t之后，接着将MediaPlayerService请求的数据拷贝到内核空间并解析出来。从数据中解析出MediaPlayerService请求数据中的flat_binder_object对象，只有一个flat_binder_object对象。该flat_binder_object对象的类型是BINDER_TYPE_BINDER，然后调用binder_get_node()在当前进程的上下文环境proc中查找fp->binder对应的Binder实体，fp->binder是Android的flatten_binder()中赋值的，它是MediaPlayerService对象的本地引用；此外，在MediaPlayerService是初次与Binder驱动通信，因此肯定找不到该对象fp->binder对应的Binder实体；因此node=NULL。  接下来，就调用binder_new_node()新建fp->binder对应的Binder实体，这也就是MediaPlayerService对应的Binder实体。然后，调用binder_get_ref_for_node(target_proc, node)获取该Binder实体在target_proc(即Service Manager的进程上下文环境)中的Binder引用，此时，在target_proc中肯定也找不到该Binder实体对应的引用；那么，就新建Binder实体的引用，并将其添加到target_proc->refs_by_node红黑树 和 target_proc->refs_by_desc红黑树中。 这样，Service Manager的进程上下文中就存在MediaPlayerService的Binder引用，Service Manager也就可以对MediaPlayerService进行管理了！  
  然后，修改fp->type=BINDER_TYPE_HANDLE，并使fp->handle = ref->desc。
[skywang-todo],why?



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
最后，target_wait是Service Manager的等待队列，肯定不为空。因此，便会执行wake_up_interruptible(target_wait)唤醒Service Manager进程。  
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

此时，MediaPlayerService进程还会继续运行，而且它也通过wake_up_interruptible()唤醒了Service Manager进程。我们还是先分析完MediaPlayerService进程，然后再看Service Manager被唤醒后会干什么？


至此，binder_transaction()就分析完了。在binder_transaction()中，我们主要进行了以下工作：
(01) 解析出来MediaPlayerService的请求数据。  
(02) 新建MediaPlayerService对应的Binder实体和Binder引用，并将它的Binder引用添加到在Service Manager的进程上下文中进行管理。  
(03) 新建了待处理事务，并将该事务添加到了Service Manager的待处理事务队列中。  
(04) 新建了待完成工作，并将待完成工作添加到了当前线程的待完成工作队列中。  


binder_thread_write()中执行binder_transaction()后，会更新*consumed的值，即bwr.write_consumed的值。意味着，Binder驱动已经驱动完成MediaPlayerService的请求数据。

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


接下来，ioctl()会执行binder_thread_read()来设置反馈数据给MediaPlayerService进程。  


## Binder驱动中binder_thread_read()的源码

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



<br/>接下来，回到binder_ioctl()中。将bwr数据拷贝到用户空间后返回。此时，bwr中各个参数的值如下：

    bwr.write_size = outAvail;                          
    bwr.write_buffer = (long unsigned int)mOut.data();
    bwr.write_consumed = outAvail;                      // 等于write_size
    bwr.read_size = mIn.dataCapacity();
    bwr.read_buffer = (long unsigned int)mIn.data();    // 存储了BR_NOOP和BR_TRANSACTION_COMPLETE两个返回指令
    bwr.read_consumed = 8;                              // 等于write_size

bwr中的write_*参数是保存"MediaPlayerService发送给Binder驱动的请求内容的"，而read_*则是保存"Binder驱动反馈给MediaPlayerService的内容的"。此时，write_consumed和write_size相同，意味着"Binder驱动已经将请求的内容都处理完毕了"；而read_consumed>0，则意味着"Binder驱动有反馈内容给MediaPlayerService"。  
回到talkWithDriver()中，看看ioctl()之后做了些什么？


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




# IPCThreadState::talkWithDriver()

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


## Binder驱动中binder_thread_read()的源码

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


<br>至此，MediaPlayerService进程的addService()部分就讲解完了。 

在前面，我们说到MediaPlayerService在执行事务，即调用binder_transaction()时，会将一个待处理事务添加到"Service Manager的等待队列"中，然后再调用wake_up_interruptible()将Service Manager进程唤醒。  
下面，就接着[skywang-todo]中的休眠部分进行讲解，看看Service Manager被唤醒后，会干些什么。


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
            // 这里，MediaPlayerService的目标是Service Manager，因此target_node是Service Manager对应的节点；
            // 它它的值在事务交互时(binder_transaction中)，被赋值为Service Manager对应的Binder实体。  
            if (t->buffer->target_node) {
                // 事务目标对应的Binder实体(即，Service Manager对应的Binder实体)
                struct binder_node *target_node = t->buffer->target_node;
                // Binder实体在用户空间的地址(Service Manager的ptr为NULL)
                tr.target.ptr = target_node->ptr;
                // Binder实体在用户空间的其它数据(Service Manager的cookie为NULL)
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

说明：Service Manager进程在调用wait_event_interruptible_exclusive(proc->wait, binder_has_proc_work(proc, thread))进入等待之后，被MediaPlayerService进程唤醒。唤醒之后，binder_has_thread_work()为true，因为Service Manager中有个待处理事务(即，MediaPlayerService添加服务的请求)。  
(01) 进入while循环后，首先取出待处理事务。  
(02) 事务的类型是BINDER_WORK_TRANSACTION，得到对应的binder_transaction*类型指针t之后，跳出switch语句。很显然，此时t不为NULL，因此继续往下执行。下面的工作的目的，是将t中的数据转移到tr中(tr是事务交互数据包结构体binder_transaction_data对应的指针)，然后将指令和tr数据都拷贝到用户空间，让Service Manager读取后进行处理。  
下面选取比较重要的几个部分进行说明。


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

说明：上面是将MediaPlayerService在执行addService()时发送的数据赋值到tr中。回顾一下，MediaPlayerService发送的数据，就是图中的数据[skywang-todo]。data_size是数据的大小，offsets_size是对象个数，buffer是数据，offsets是对象的偏移数组。

这里着重强调一下地址的赋值方式，因为它涉及到Binder机制的数据拷贝原理！   
(01) t->buffer是在binder_transaction()中，通过binder_alloc_buf()分配的内核空间地址。现在要将数据返回给Service Manager守护进程，即将内核空间的数据拷贝到用户空间。前面，在[skywang-todo]的mmap()中，我们将内核虚拟地址和进程虚拟地址映射到同一个物理存储区；现在，已知内核虚拟地址(即t->buffer->data)。那么，只需要将t->buffer->data加上proc->user_buffer_offset(内核虚拟地址和进程虚拟地址的偏移)即可得到在用户空间的地址。  
(02) 至于tr.data.ptr.offsets的值，即数据中对象的偏移数组。将"数据的起始指针" 加上 "数据的大小"即可得到。

在tr赋值完毕之后，就将完整数据拷贝到用户空间。此时，该事务已经在Binder驱动中被处理，于是将事务从Service Manager的待处理事务队列中删除。Binder驱动随后会将该事务发送给Service Manager守护进程，Service Manager守护进程在处理完事务之后，需要反馈结果给Binder驱动。因此，接下来会设置t->to_thread和t->transaction_stack等成员。最后，修改*consumed的值，即bwr.read_consumed的值，表示待读取内容的大小。  
执行完binder_thread_read()之后，回到binder_ioctl()中，执行copy_to_user()将数据拷贝到用户空间。接下来，就回到了Service Manager的守护进程当中，即回到binder_loop()中。


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

说明：此处里的cmd就是bwr.read_buffer指针。而在Binder驱动的binder_thread_read()中，反馈的第一个指令是BR_NOOP；因此这里的cmd=BR_NOOP，不执行任何动作，继续取出下一个指令cmd=BR_TRANSACTION。在BR_TRANSACTION中，会先取出消息，再对消息处理之后，再将反馈信息发送给Binder驱动。下面是BR_TRANSACTION的详细内容。  
(01) 首先，将ptr转换成struct binder_txn结构体指针。struct binder_txn是与binder_transaction_datad对应的结构体，在[skywang-todo]中有它的详细介绍。  
(02) 此处的func是函数指针svcmgr_handler，不为空；因此，先调用bio_init()初始化reply，再调用bio_init_from_txn()来初始化msg。  
(03) 初始化完毕之后，就调用svcmgr_handler()对消息进行处理。  
(04) 消息处理完毕，就通过binder_send_reply()将处理结果反馈给Binder驱动。  




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



    void bio_init_from_txn(struct binder_io *bio, struct binder_txn *txn)
    {           
        bio->data = bio->data0 = txn->data;    // 数据起始地址
        bio->offs = bio->offs0 = txn->offs;    // 数据中对象的偏移数组的起始地址
        bio->data_avail = txn->data_size;      // 数据大小
        bio->offs_avail = txn->offs_size / 4;  // 对象个数
        bio->flags = BIO_F_SHARED;
    }

说明：bio_init_from_txn()就是根据已有的数据txn初始化struct binder_io的各个成员。 




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

说明：binder_object是与flat_binder_object对应的结构体，关于它的详细介绍可以参考[skywang-todo]。
(01) _bio_get_obj(bio)的代码就不展开了，它是根据bio创建binder_object对象。实际上，obj就是MediaPlayerService打包成的flat_binder_object对象。  
(02) obj->type的值是BINDER_TYPE_HANDLE。原来MediaPlayerService对应的type是BINDER_TYPE_BINDER，但在Binder驱动的binder_transaction()中，将type修改成了BINDER_TYPE_HANDLE。因此，返回obj->pointer，而obj->pointer实际上是flat_binder_object中的handle，而该handle在Binder驱动中被赋值为"MediaPlayerService对应的Binder引用的描述，即binder_ref->desc"。根据该引用描述，可以在Binder驱动中找到MediaPlayerService对应的Binder实体以及MediaPlayerService对应的进程上下文信息，进而可以给MediaPlayerService发送消息。  


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

接下来，回到svcmgr_handler()中，调用bio_put_uint32(reply, 0)；将0写入到reply中。

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
(01) 先看看参数。bs是struct binder_state，它保存了打开"/dev/binder"文件的相关信息。reply是中有数据0。buffer_to_free是对应binder_transaction_data中保存请求数据的buffer缓冲区，它是在Binder驱动的binder_transaction()中分配的。status_t=0。  
(02) 该函数中的私有结构体struct是用来描述返回给Binder驱动的数据。我们知道，Binder机制的交互数据的格式是"指令+数据"。这里，返回的指令有两个BC_FREE_BUFFER和BC_REPLY，BC_FREE_BUFFER是告诉Binder驱动，请求处理完毕，让Binder驱动释放数据缓冲；而BC_REPLY是告诉Binder驱动，这是回复，回复的内容是data.txt.data，这里面的内容就是好reply中的数值0。  
(03) 最后，调用binder_write()将数据打包。



## binder_write()的源码


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

说明：binder_write()单单只是向Kernel发送一个消息，而不会去读取消息反馈。此时，便再次进入到Binder驱动中。

## Binder驱动中binder_ioctl()的BINDER_WRITE_READ相关部分的源码

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
(01) binder_write_read()先读出BC_FREE_BUFFER指令，然后保存数据的内存。代码中给出了相应的注释，这里就不再详细说明了。  
(02) 接着，读出BC_REPLY指令，将数据拷贝到内核空间之后，便执行binder_transaction()对数据进行处理。


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

        /* TODO: reuse incoming transaction for reply */
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



[skywang-todo: tag]

下面，还是先看完Service Manager的流程，然后再来看MediaPlayerService被唤醒后做了什么。


Service Manager执行完binder_transaction()后，回到binder_thread_write()中；此时，数据已经处理完毕，便返回到binder_ioctl()中。binder_ioctl()将数据拷贝到用户空间后，Binder驱动的就结束了。

回到Service Manager守护进程中，binder_write()执行完ioctl()后，返回到binder_send_reply()中，binder_send_reply()则进一步返回到binder_parse()。binder_parse()已经解析完请求数据，于是进一步返回到binder_loop()中。  
binder_loop()会再次开始循环，调用ioctl(,BINDER_WRITE_READ,)到Binder驱动执行读操作。此时，再次进入到Binder驱动的binder_ioctl()，然后会调用binder_thread_read()执行读操作。此时，Service Manager线程中有一个类型为BINDER_WORK_TRANSACTION_COMPLETE的待处理事务；于是，执行BINDER_LOOPER_STATE_NEED_RETURN动作，将该事务从Service Manager的待处理事务队列中删除，并反馈cmd=BR_TRANSACTION_COMPLETE信息给Service Manager守护进程。Service Manager守护进程收到Binder驱动的反馈后，解析出BR_TRANSACTION_COMPLETE，该指令什么也不做。于是，Service Manager再次调用ioctl(,BINDER_WRITE_READ,)，此时，待处理事务队列为空，因此，Service Manager再次进入中断等待状态。

至此，MediaPlayerService发送addService()请求给Service Manager，Service Manager的全部工作已经处理完毕！






[skywang-todo: tag]

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

说明：MediaPlayerService进程被Service Manager唤醒，同时它的待处理事务队列中有Service Manager添加的事务；此时，binder_has_thread_work()为true。因此，MediaPlayerService会继续往下执行。  
(01) 进入while循环后，首先取出待处理事务。  
(02) 事务的类型是BINDER_WORK_TRANSACTION，得到对应的binder_transaction*类型指针t之后，跳出switch语句。时t不为NULL，因此继续往下执行。下面的工作的目的，是将t中的数据转移到tr中(tr是事务交互数据包结构体binder_transaction_data对应的指针)，然后将指令和tr数据都拷贝到用户空间，让MediaPlayerService读取后进行处理。  
binder_thread_read()的内容，前面已经详细介绍国了。这里说一下与前面不同的地方，由于这里的消息是要反馈给MediaPlayerService；因此，此时的cmd = BR_REPLY，在将事务对应的数据都拷贝到用户空间之后，会将事务删除。


MediaPlayerService收到的Binder驱动的反馈包含了两个指令：BR_NOOP和BR_REPLY。 BR_NOOP的处理过程，在前面已经介绍过了；实际上，BR_NOOP不会引起任何实质性的改变。接着，MediaPlayerService会解析出BR_REPLY指令，并对之进行处理。下面，只截取与BR_REPLY处理相关的部分进行说明。


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

说明：ipcSetDataReference()是根据参数的值重新初始化Parcel的数据和对象。  
前面我们说过，Binder驱动反馈的BR_REPLY的数据中只有数字0而已。下面我们就看看ipcSetDataReference()的各个参数，data是数字0的地址，dataSize是数据大小；而数据中没有对象，因此objectsCount=0。  该函数会先调用freeDataNoInit()来释放已有的内存。然后再重新初始化mData和mDataSize等成员。

在执行完该函数之后，MediaPlayerService的addService()请求层层返回。MediaPlayerService::instantiate()也就正式执行完了。



