---
layout: post
title: "Android Binder机制(四) defaultServiceManager()的实现"
description: "android"
category: android
tags: [android]
date: 2014-09-04 09:04
---


> 本文会介绍Android的消息处理机制。  

> **目录**  
> **1**. [Android消息机制的架构](#anchor1)  
> **2**. [Android消息机制的源码解析](#anchor2)  
> **2.1**. [消息循环](#anchor2_1)  
> **2.2**. [消息的发送](#anchor2_2)  
> **2.3**. [消息的处理](#anchor2_3)  

> 注意：本文是基于Android 4.4.2版本进行介绍的！



<a name="anchor1"></a>
# 1. defaultServiceManager()

    sp<IServiceManager> defaultServiceManager()
    {
        if (gDefaultServiceManager != NULL) return gDefaultServiceManager;

        {
            AutoMutex _l(gDefaultServiceManagerLock);
            while (gDefaultServiceManager == NULL) {
                gDefaultServiceManager = interface_cast<IServiceManager>(
                    ProcessState::self()->getContextObject(NULL));
                if (gDefaultServiceManager == NULL)
                    sleep(1);
            }
        }

        return gDefaultServiceManager;
    }


说明：该代码定义在frameworks/native/libs/binder/IServiceManager.cpp中。它是获取Service Manager的接口，该函数的声明在frameworks/native/include/binder/IServiceManager.h中。  
(01) gDefaultServiceManagerLock是全局互斥锁，gDefaultServiceManager是全局的IServiceManager对象。它们都定义在frameworks/native/libs/binder/Static.cpp中。  
(02) 该函数肯定会返回gDefaultServiceManager对象。gDefaultServiceManager是单例模式对象，它的实现可以简化为以下语句：

    gDefaultServiceManager = interface_cast<IServiceManager>(ProcessState::self()->getContextObject(NULL));



<a name="anchor2"></a>
# 2. ProcessState::self()

    sp<ProcessState> ProcessState::self()
    {
        Mutex::Autolock _l(gProcessMutex);
        if (gProcess != NULL) {
            return gProcess;
        }
        gProcess = new ProcessState;
        return gProcess;
    }

说明：该代码定义在frameworks/native/libs/binder/ProcessState.cpp中，它的作用是返回gProcess对象。gProcess也是单例模式对象，它也定义在frameworks/native/libs/binder/Static.cpp中。第一次执行self()，会新建ProcessState对象。  



<a name="anchor3"></a>
# 3. ProcessState::ProcessStatef()

    ProcessState::ProcessState()
        : mDriverFD(open_driver())
        , mVMStart(MAP_FAILED)
        , mManagesContexts(false)
        , mBinderContextCheckFunc(NULL)
        , mBinderContextUserData(NULL)
        , mThreadPoolStarted(false)
        , mThreadPoolSeq(1)
    {
        if (mDriverFD >= 0) {
            mVMStart = mmap(0, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);
            if (mVMStart == MAP_FAILED) {
                // *sigh*
                ALOGE("Using /dev/binder failed: unable to mmap transaction memory.\n");
                close(mDriverFD);
                mDriverFD = -1;
            }
        }
        ...
    }

说明：在ProcessState的构造函数中，它会进行一系列的初始化。  
(01) 通过open_driver()打开"/open/binder"，并将文件句柄赋值给mDriverFD。  
(02) 通过调用mmap()映射内存。  
下面，查看这两个步骤的代码。


<a name="anchor4"></a>
# 4. ProcessState::open_driver()

    static int open_driver()
    {
        // 打开文件/dev/binder
        int fd = open("/dev/binder", O_RDWR);
        if (fd >= 0) {
            fcntl(fd, F_SETFD, FD_CLOEXEC);
            int vers;
            // 检查/dev/binder的版本
            status_t result = ioctl(fd, BINDER_VERSION, &vers);
            if (result == -1) {
                close(fd);
                ...
            }
            if (result != 0 || vers != BINDER_CURRENT_PROTOCOL_VERSION) {
                close(fd);
                ...
            }

            // 设置该进程最大线程数
            size_t maxThreads = 15;
            result = ioctl(fd, BINDER_SET_MAX_THREADS, &maxThreads);
            if (result == -1) {
                ...
            }
        } else {
            ...
        }
        return fd;
    }

说明：  
(01) open_driver()首先打开/dev/binder。它会对应执行Binder驱动的open()函数，open()函数在[TODO]中已经详细介绍过了。  
(02) 在成功打开文件之后，就会调用ioctl检查Binder版本，检查版本的部分非常简单(就是读取出版本号，判断读取的版本号与已有的版本号是否一样!)，这里就不再对Binder驱动的BINDER_VERSION进行展开了。  
(03) 在检查版本通过之后，在调用ioctl(,BINDER_SET_MAX_THREADS,)设置该进程的最大线程数。它会对应调用Binder驱动的binder_ioctl()函数。    

**注意**：区分"此处的open("/dev/binder",...)" 和 "Service Manager守护进程中也调用过open("/dev/binder",...)"。它们分别是属于不同的进程，每个进程打开/dev/binder文件后，Binder驱动都会创建相应的binder_proc对象来保存进程的上下文信息。



<a name="anchor5"></a>
# 5. Binder驱动中binder_ioctl()的BINDER_SET_MAX_THREADS相关部分的源码


    static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
    {
      int ret;
      struct binder_proc *proc = filp->private_data;
      struct binder_thread *thread;
      unsigned int size = _IOC_SIZE(cmd);
      void __user *ubuf = (void __user *)arg;

      ret = wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);

      // 在proc进程中查找该线程对应的binder_thread；若查找失败，则新建一个binder_thread，并添加到proc->threads中。
      thread = binder_get_thread(proc);
      ...

      switch (cmd) {
        case BINDER_SET_MAX_THREADS:
            if (copy_from_user(&proc->max_threads, ubuf, sizeof(proc->max_threads))) {
                ret = -EINVAL;
                goto err;
            }
            break;

      ...
      }
      ret = 0;

      ...
      return ret;
    }

说明：BINDER_SET_MAX_THREADS的代码很简单，就是将最大线程数目从用户空间拷贝到内核空间，进而赋值给binder_proc->max_threads。


<br/>
到目前为止，ProcessState::self()就分析完毕。返回defaultServiceManager()函数就可以转换为以下语句：

    gDefaultServiceManager = interface_cast<IServiceManager>(ProcessState::getContextObject(NULL));


<a name="anchor6"></a>
# 6. ProcessState::getContextObject()

    sp<IBinder> ProcessState::getContextObject(const sp<IBinder>& caller)
    {
        return getStrongProxyForHandle(0);
    }

说明：getContextObject()调用了getStrongProxyForHandle(0)。这里的0是指定的Service Manager的句柄。



<a name="anchor7"></a>
# 7. ProcessState::getStrongProxyForHandle()

    sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle)
    {
        sp<IBinder> result;

        AutoMutex _l(mLock);

        // 在矢量数组中查找"句柄值为handle的handle_entry对象"；
        // 找到的话，则直接返回；找不到的话，则新建handle对应的handle_entry。
        handle_entry* e = lookupHandleLocked(handle);

        if (e != NULL) {
            IBinder* b = e->binder;
            if (b == NULL || !e->refs->attemptIncWeak(this)) {
                // 当handle==0(即是Service Manager的句柄)时，尝试去ping Binder驱动。
                if (handle == 0) {
                    Parcel data;
                    status_t status = IPCThreadState::self()->transact(
                            0, IBinder::PING_TRANSACTION, data, NULL, 0);
                    if (status == DEAD_OBJECT)
                       return NULL;
                }

                // 新建BpBinder代理
                b = new BpBinder(handle);
                e->binder = b;
                if (b) e->refs = b->getWeakRefs();
                result = b;
            } else {
                ...
            }
        }

        return result;
    }

说明：getStrongProxyForHandle()的目的是返回Service Manager(句柄为0)的IBinder代理。  
(01) lookupHandleLocked()，是在矢量数组中查找是否有handler对应的handle_entry，有的话则返回；没有的话，则新建handle对应的handle_entry，并将其添加到矢量数组中，然后再返回。  
(02) 很显然，此时e!=NULL为true，进入if(e!=NULL)中。而此时e->binder=NULL，并且handle=0；则调用IPCThreadState::self()->transact()尝试去和Binder驱动通信(尝试去ping内核中Binder驱动)。由于Binder驱动已启动，ping通信是能够成功的。ping通信涉及到"Binder机制中Server和Client的通信"，后面再专门对Server和Client的交互进行介绍；这里只要了解ping通信能够成功即可。  
(03) 接着，新建BpBinder对象，并赋值给e->binder。然后，将该BpBinder对象返回。

上面对流程进行了简单介绍，下面逐个进行分析！


<a name="anchor8"></a>
# 8. ProcessState::lookupHandleLocked()

    ProcessState::handle_entry* ProcessState::lookupHandleLocked(int32_t handle)
    {               
        const size_t N=mHandleToObject.size();
        if (N <= (size_t)handle) {
            handle_entry e;
            e.binder = NULL;
            e.refs = NULL;
            status_t err = mHandleToObject.insertAt(e, N, handle+1-N);
            if (err < NO_ERROR) return NULL;
        }   
        return &mHandleToObject.editItemAt(handle);
    }           

说明：mHandleToObject是Vector矢量数组。mHandleToObject的初始大小为0，因此if (N <= handle)为true。接下来，就新建handle_entry，并将其添加到mHandleToObject中，然后返回该handle_entry。mHandleToObject和handle_entry的定义如下：

    class ProcessState : public virtual RefBase
    {
        ...
    private:
        ...
        struct handle_entry {
            IBinder* binder;
            RefBase::weakref_type* refs;
        };

        ...

        Vector<handle_entry>mHandleToObject;
        ...
    }

说明：该代码定义在frameworks/native/include/binder/ProcessState.h中。



<a name="anchor9"></a>
# 9. BpBinder::BpBinder

new BpBinder(0)会新建BpBinder对象，下面看看BpBinder的构造函数。

    BpBinder::BpBinder(int32_t handle)
        : mHandle(handle)
        , mAlive(1)
        , mObitsSent(0)
        , mObituaries(NULL)
    {
        ALOGV("Creating BpBinder %p handle %d\n", this, mHandle);

        extendObjectLifetime(OBJECT_LIFETIME_WEAK);
        IPCThreadState::self()->incWeakHandle(handle);
    }

说明：该代码定义在frameworks/native/libs/binder/BpBinder.cpp中。主要工作是初始化。  
(01) 将Service Manager的句柄handle保存到mHandle中。  
(02) 增加IPCThreadState的引用计数。IPCThreadState::self()是获取IPCThreadState对象，实际上，在前面介绍的ProcessState::getStrongProxyForHandle()中已经调用过该函数。下面看看它的代码。




<a name="anchor10"></a>
# 10. IPCThreadState::self()

    static pthread_mutex_t gTLSMutex = PTHREAD_MUTEX_INITIALIZER;
    static bool gHaveTLS = false;
    static pthread_key_t gTLS = 0;
    static bool gShutdown = false;
    static bool gDisableBackgroundScheduling = false;

    IPCThreadState* IPCThreadState::self()
    {
        if (gHaveTLS) {
    restart:
            const pthread_key_t k = gTLS;
            IPCThreadState* st = (IPCThreadState*)pthread_getspecific(k);
            if (st) return st;
            return new IPCThreadState;
        }

        if (gShutdown) return NULL;

        pthread_mutex_lock(&gTLSMutex);
        if (!gHaveTLS) {
            if (pthread_key_create(&gTLS, threadDestructor) != 0) {
                pthread_mutex_unlock(&gTLSMutex);
                return NULL;
            }
            gHaveTLS = true;
        }
        pthread_mutex_unlock(&gTLSMutex);
        goto restart;
    }

说明：该代码定义在frameworks/native/libs/binder/IPCThreadState.cpp中。self()的源码比较简单，它的作用是获取IPCThreadState对象。若干有该对象已经存在，则直接返回；否则，新建IPCThreadState对象。




<a name="anchor11"></a>
# 11. IPCThreadState::IPCThreadState()

    IPCThreadState::IPCThreadState()
        : mProcess(ProcessState::self()),
          mMyThreadId(androidGetTid()),
          mStrictModePolicy(0),
          mLastTransactionBinderFlags(0)
    {
        pthread_setspecific(gTLS, this);
        clearCaller();
        mIn.setDataCapacity(256);
        mOut.setDataCapacity(256);
    }

说明：  
(01) 获取ProcessState对象(前面介绍过，ProcessState是单例模式，返回的是gProcess全局对象)，并将其赋值给成员mProcess。   
(02) 设置mIn和mOut的容量为256字节。  



<br/>
到目前为止，ProcessState::getContextObject()就分析完了。获取的gDefaultServiceManager的语句可以转换成：

    gDefaultServiceManager = interface_cast<IServiceManager>(new BpBinder(0));

接下来，看看interface_cast<IServiceManager>。



<a name="anchor12"></a>
# 12. interface_cast()

    template<typename INTERFACE>
    inline sp<INTERFACE> interface_cast(const sp<IBinder>& obj)
    {
        return INTERFACE::asInterface(obj);
    }   

说明：该代码定义在IServiceManager的父类IInterface中，具体是frameworks/native/include/binder/IInterface.h文件。该函数是模板函数，返回IServiceManager::asInterface()。  


<a name="anchor13"></a>
# 13. IServiceManager::asInterface()

IServiceManager::asInterface()是通过DECLARE_META_INTERFACE()来声明，并通过IMPLEMENT_META_INTERFACE()来实现的。

(01) IServiceManager中的DECLARE_META_INTERFACE()声明和IMPLEMENT_META_INTERFACE()实现，分别在头文件frameworks/native/include/binder/IServiceManager.h 以及 frameworks/native/libs/binder/IServiceManager.cpp中。

    // IServiceManager.h中的代码
    class IServiceManager : public IInterface
    {
    public:
        DECLARE_META_INTERFACE(ServiceManager);
        ...
    }

    // IServiceManager.h中的代码
    IMPLEMENT_META_INTERFACE(ServiceManager, "android.os.IServiceManager");

(02) DECLARE_META_INTERFACE()和IMPLEMENT_META_INTERFACE的定义在frameworks/native/include/binder/IInterface.h中。 

    #define DECLARE_META_INTERFACE(INTERFACE)                               \
        static const android::String16 descriptor;                          \
        static android::sp<I##INTERFACE> asInterface(                       \
                const android::sp<android::IBinder>& obj);                  \
        virtual const android::String16& getInterfaceDescriptor() const;    \
        I##INTERFACE();                                                     \
        virtual ~I##INTERFACE();                                            \

    #define IMPLEMENT_META_INTERFACE(INTERFACE, NAME)                       \
        const android::String16 I##INTERFACE::descriptor(NAME);             \
        const android::String16&                                            \
                I##INTERFACE::getInterfaceDescriptor() const {              \
            return I##INTERFACE::descriptor;                                \
        }                                                                   \
        android::sp<I##INTERFACE> I##INTERFACE::asInterface(                \
                const android::sp<android::IBinder>& obj)                   \
        {                                                                   \
            android::sp<I##INTERFACE> intr;                                 \
            if (obj != NULL) {                                              \
                intr = static_cast<I##INTERFACE*>(                          \
                    obj->queryLocalInterface(                               \
                            I##INTERFACE::descriptor).get());               \
                if (intr == NULL) {                                         \
                    intr = new Bp##INTERFACE(obj);                          \
                }                                                           \
            }                                                               \
            return intr;                                                    \
        }                                                                   \
        I##INTERFACE::I##INTERFACE() { }                                    \
        I##INTERFACE::~I##INTERFACE() { }                                   \


    #define CHECK_INTERFACE(interface, data, reply)                         \
        if (!data.checkInterface(this)) { return PERMISSION_DENIED; }       \



用ServiceManager替换INTERFACE之后，得到结果如下：
IMPLEMENT_META_INTERFACE(ServiceManager, "android.os.IServiceManager");

    #define DECLARE_META_INTERFACE(IServiceManager)                         \
        static const android::String16 descriptor;                          \
        static android::sp<IServiceManager> asInterface(                    \
                const android::sp<android::IBinder>& obj);                  \
        virtual const android::String16& getInterfaceDescriptor() const;    \
        IServiceManager();                                                  \
        virtual ~IServiceManager();                                         \


    #define IMPLEMENT_META_INTERFACE(IServiceManager, "android.os.IServiceManager")        \
        const android::String16 IServiceManager::descriptor("android.os.IServiceManager"); \
        const android::String16&                                                           \
                IServiceManager::getInterfaceDescriptor() const {                          \
            return IServiceManager::descriptor;                                            \
        }                                                                                  \
        android::sp<IServiceManager> IServiceManager::asInterface(                         \
                const android::sp<android::IBinder>& obj)                                  \
        {                                                                                  \
            android::sp<IServiceManager> intr;                                             \
            if (obj != NULL) {                                                             \
                intr = static_cast<IServiceManager*>(                                      \
                    obj->queryLocalInterface(                                              \
                            IServiceManager::descriptor).get());                           \
                if (intr == NULL) {                                                        \
                    intr = new BpServiceManager(obj);                                      \
                }                                                                          \
            }                                                                              \
            return intr;                                                                   \
        }                                                                                  \
        IServiceManager::IServiceManager() { }                                             \
        IServiceManager::~IServiceManager() { }



因此，IServiceManager::asInterface()的源码如下：

    android::sp<IServiceManager> IServiceManager::asInterface(const android::sp<android::IBinder>& obj)
    {
        android::sp<IServiceManager> intr;
        if (obj != NULL) {
            intr = static_cast<IServiceManager*>(
                obj->queryLocalInterface(
                        IServiceManager::descriptor).get());
            if (intr == NULL) {
                intr = new BpServiceManager(obj);
            }
        }
        return intr;
    }

说明：asInterface()的作用是获取IServiceManager接口。  
(01) obj是传入的BpBinder对象，不为NULL。因此，执行obj->queryLocalInterface("android.os.IServiceManager")来查找名称为"android.os.IServiceManager"的本地接口，queryLocalInterface()的实现在BpBinder的父类IBinder中，具体在文件frameworks/native/libs/binder/Binder.cpp中。很显然，IServiceManager接口还没创建，因此intr=NULL。   
(02) 新建BpServiceManager(obj)对象，并返回。BpServiceManager的实现在frameworks/native/libs/binder/IServiceManager.cpp中。  

    sp<IInterface>  IBinder::queryLocalInterface(const String16& descriptor)
    {   
        return NULL;
    }   


<br/>
到目前为止，gDefaultServiceManager的创建流程就分析完了，它实际返回的是一个BpServiceManager对象，该对象包含IBinder的代理BpBinder。以下是转换后的获取gDefaultServiceManager的语句。

    gDefaultServiceManager = new BpServiceManager(new BpBinder(0));




























<a name="anchor10"></a>
# 10. IPCThreadState::transact()


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
                ...
            } else {
                Parcel fakeReply;
                err = waitForResponse(&fakeReply);
            }

        } else {
            ...
        }

        return err;
    }

说明：transact()是事务处理函数，它会与Binder驱动进行交互从而处理相应的事务。此时，该函数的具体参数是transact(0, IBinder::PING_TRANSACTION, data, NULL, 0)。  
(01) errorCheck()返回的err为NO_ERROR。接下来，会执行writeTransactionData()将数据打包到binder_transaction_data结构体中。  
(02) 传进来的flag=0，因此flag & TF_ONE_WAY为true。而reply=0，则执行waitForResponse(&fakeReply)。waitForResponse()会将请求发送给Binder驱动，并得到返回值err，而并不需要Binder驱动对请求进行答复。  



<a name="anchor11"></a>
# 11. IPCThreadState::writeTransactionData()
                    

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
            ...
        } else {
            ...
        }

        mOut.writeInt32(cmd);
        mOut.write(&tr, sizeof(tr));

        return NO_ERROR;
    }

说明：该函数会将数据打包到binder_transaction_data中，binder_transaction_data是Binder驱动和Android的C++层的事务交互的数据格式，在[skywang-todo]中有对该结构体的详细介绍。处理完毕之后，对应的tr内容如下：  

    tr.target.handle = 0;
    tr.code = IBinder::PING_TRANSACTION; 
    tr.flags = TF_ACCEPT_FDS;
    tr.cookie = 0;
    tr.sender_pid = 0;
    tr.sender_euid = 0;
    // 没有具体的数据
    tr.data_size = 0;
    tr.data.ptr.buffer = NULL;
    tr.offsets_size = 0;
    tr.data.ptr.offsets = NULL;

也就是说，PING_TRANSACTION只是单纯发送一个指令(PING_TRANSACTION)给Binder驱动，而没有任何实质的数据。在tr对象初始化完毕之后，再将cmd(BC_TRANSACTION)和该tr对象写入到mOut中。数据准备OK了，接下来就执行waitForResponse()将数据发送给Binder驱动，然后获取返回值。  



<a name="anchor12"></a>
# 12. IPCThreadState::waitForResponse()

    status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
    {
        ...

        while (1) {
            if ((err=talkWithDriver()) < NO_ERROR) break;
                err = mIn.errorCheck();
            if (err < NO_ERROR) break;
            if (mIn.dataAvail() == 0) continue;

            ...
        }

        ...
        return err;
    }

说明：该函数会先调用talkWithDriver()将请求发送给Binder驱动，待Binder驱动处理完毕返回之后，再根据返回结果进行相关应的处理。  



<a name="anchor13"></a>
# 13. IPCThreadState::talkWithDriver()

    status_t IPCThreadState::talkWithDriver(bool doReceive)
    {
        ...

        binder_write_read bwr;

        const bool needRead = mIn.dataPosition() >= mIn.dataSize();

        const size_t outAvail = (!doReceive || needRead) ? mOut.dataSize() : 0;

        bwr.write_size = outAvail;
        bwr.write_buffer = (long unsigned int)mOut.data();

        if (doReceive && needRead) {
            bwr.read_size = mIn.dataCapacity();
            bwr.read_buffer = (long unsigned int)mIn.data();
        } else {
            ...
        }

        if ((bwr.write_size == 0) && (bwr.read_size == 0)) return NO_ERROR;

        bwr.write_consumed = 0;
        bwr.read_consumed = 0;
        status_t err;
        do {
            ...
            if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0)
                err = NO_ERROR;
            else
                err = -errno;
            ...
        } while (err == -EINTR);

        ...
    }

说明：talkWithDriver()是与Binder驱动交互的函数。它会先将数据整合到binder_write_read结构体中，然后再先通过binder_write_read将请求发送给Binder驱动。  
(01) doReceive默认是true。  
(02) mIn中没有数据，所以mIn.dataPosition()和mIn.dataSize()都是0。这样，needRead()就是true。  
(03) outAvail的值=mOut.dataSize()，不为0。因为在writeTransactionData()中，有向mOut中写入数据；mOut.dataSize()!=0，outAvail也就不为0。    
(04) 在初始化完毕bwr之后，就通过ioctl()与Binder驱动进行事务交互，它对应会调用Binder驱动的binder_ioctl()。

此时，bwr的数据如下：

        bwr.write_size = outAvail;                          // mOut的数据大小，非零
        bwr.write_buffer = (long unsigned int)mOut.data();  // mOut的数据地址
        bwr.write_consumed = 0;
        bwr.read_size = mIn.dataCapacity();                 // mIn的容量，为256字节。
        bwr.read_buffer = (long unsigned int)mIn.data();    // mIn的数据大小，零。
        bwr.read_consumed = 0;


<a name="anchor14"></a>
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

说明：bwr.write_size>0，且bwr.read_size>0；则先写后读。  



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




<a name="anchor16"></a>
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

    e = binder_transaction_log_add(&binder_transaction_log);
    e->call_type = reply ? 2 : !!(tr->flags & TF_ONE_WAY);
    e->from_proc = proc->pid;
    e->from_thread = thread->pid;
    e->target_handle = tr->target.handle;
    e->data_size = tr->data_size;
    e->offsets_size = tr->offsets_size;

    if (reply) {
    ...
    } else {
        if (tr->target.handle) {
        } else {
            // 事务目标对象是Service Manager的binder实体
            // 即，该事务是交给Service Manager来处理的。
            target_node = binder_context_mgr_node;
            ...
        }
        e->to_node = target_node->debug_id;
        // 设置处理事务的目标进程
        target_proc = target_node->proc;
        ...
    }


