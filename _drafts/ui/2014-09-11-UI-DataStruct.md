---
layout: post
title: "Android UI系统(二) UI中的数据结构"
description: "android"
category: android
tags: [android]
date: 2014-09-10 09:01
---


> 本文介绍UI框架中涉及到的各类数据结构。

> 注意：本文是基于Android 4.4.2版本进行介绍的！

> **目录**  
> **1**. [Binder架构解析](#anchor1)  



<a name="anchor1"></a>
# 1. HAL层的数据结构


<a name="anchor1_1"></a>
## 1.1 HAL层中描述硬件设备的结构体

<a name="anchor1_1_1"></a>
### 1.1.1 hw_device_t

    typedef struct hw_device_t {
        /** tag must be initialized to HARDWARE_DEVICE_TAG */
        uint32_t tag;
        
        // 版本号
        uint32_t version;

        // 该硬件抽象设备对应的模块
        struct hw_module_t* module;

        // 保留字节
        uint32_t reserved[12];

        // 关闭硬件抽象设备的方法
        int (*close)(struct hw_device_t* device);

    } hw_device_t;

说明：该代码在hardware/libhardware/include/hardware/hardware.h中。hw_device_t是在硬件抽象层中为描述硬件设备而抽象出来的结构体，可以将一个hw_device_t变量看作一个硬件设备。



<a name="anchor1_1_2"></a>
### 1.1.2 alloc_device_t 

    typedef struct alloc_device_t {
        struct hw_device_t common;

        // 分配一个图形缓冲区
        int (*alloc)(struct alloc_device_t* dev,
                int w, int h, int format, int usage,
                buffer_handle_t* handle, int* stride);

        // 释放图形缓冲区
        int (*free)(struct alloc_device_t* dev,
                buffer_handle_t handle);  

        void (*dump)(struct alloc_device_t *dev, char *buff, int buff_len);

        // 保留的数据
        void* reserved_proc[7];
    } alloc_device_t;

说明：该代码在hardware/libhardware/include/hardware/gralloc.h中。alloc_device_t封装了申请/释放图形缓冲区操作的结构体，可以将alloc_device_t看作是一个Gralloc设备。




<a name="anchor1_1_3"></a>
### 1.1.3 framebuffer_device_t

    typedef struct framebuffer_device_t {
        struct hw_device_t common;

        const uint32_t  flags;            

        // 显示屏的宽度和高度
        const uint32_t  width;
        const uint32_t  height;

        // 显示屏中一行有多少个像素点
        const int       stride;           

        // 像素的RGB格式
        const int       format;           

        // 密度
        const float     xdpi;             
        const float     ydpi;             

        // 显示屏的帧率
        const float     fps;

        // 帧缓冲区交换前后两个图形缓冲区的最小时间和最大时间
        const int       minSwapInterval;
        const int       maxSwapInterval;

        // 系统支持的缓冲区个数
        const int       numFramebuffers;

        int reserved[7];

        int (*setSwapInterval)(struct framebuffer_device_t* window,
                int interval);

        int (*setUpdateRect)(struct framebuffer_device_t* window,
                int left, int top, int width, int height);

        // 将图形缓冲区buffer中的内容渲染的帧缓冲区中，即显示到显示屏上
        int (*post)(struct framebuffer_device_t* dev, buffer_handle_t buffer);

        int (*compositionComplete)(struct framebuffer_device_t* dev);

        void (*dump)(struct framebuffer_device_t* dev, char *buff, int buff_len);

        int (*enableScreen)(struct framebuffer_device_t* dev, int enable);

        void* reserved_proc[6];

    } framebuffer_device_t;

说明：该代码在hardware/libhardware/include/hardware/fb.h中。framebuffer_device_t是描述显示缓冲设备的结构体，可以将framebuffer_device_t其看作是一个显示缓冲设备。




<a name="anchor1_2"></a>
## 1.2 HAL层中描述硬件模块的结构体

如果framebuffer_device_t表示一个显示缓冲设备，那么hw_module_t则是显示缓冲设备中的某一厂家生产的具体某一种显示设备。


<a name="anchor1_1_1"></a>
### 1.2.1 hw_module_t

    typedef struct hw_module_t {
        // 标识
        uint32_t tag;

        // 模块的api版本号
        // 通过宏定义为主版本号version_major
        uint16_t module_api_version;
    #define version_major module_api_version

        // 硬件抽象层的版本号
        // 通过宏定义为次版本号version_minor
        uint16_t hal_api_version;
    #define version_minor hal_api_version

        // 模块id
        const char *id;

        // 名称
        const char *name;

        // 作者
        const char *author;

        // 模块的hw_module_methods_t对象，该对象定义了模块的方法
        struct hw_module_methods_t* methods;

        void* dso;

        // 保留位
        uint32_t reserved[32-7];

    } hw_module_t;


说明：该代码在hardware/libhardware/include/hardware/hardware.h中。hw_module_t是硬件抽象层中的通用结构体，每一个硬件模块都有一个hw_module_t变量来记录这个硬件模块的身份信息。身份信息包括"标识，id，版本号，名称，作者"以及 "模块的API(methods成员)"。



<a name="anchor1_2_2"></a>
### 1.2.2 gralloc_module_t

    typedef struct gralloc_module_t {
        struct hw_module_t common;
        
        // 注册缓冲区
        // 实际上是将一块图形缓冲区映射到一个进程的地址空间去
        int (*registerBuffer)(struct gralloc_module_t const* module,
                buffer_handle_t handle);

        // 注销缓冲区
        int (*unregisterBuffer)(struct gralloc_module_t const* module,
                buffer_handle_t handle);

        // 锁定图形缓冲区
        // 锁定时会指定缓冲区的大小，vaddr是输出参数，表示锁定的图形缓冲区的地址
        int (*lock)(struct gralloc_module_t const* module,
                buffer_handle_t handle, int usage,
                int l, int t, int w, int h,
                void** vaddr);

        // 解锁图形缓冲区
        int (*unlock)(struct gralloc_module_t const* module,
                buffer_handle_t handle);


        // 保留的方法
        int (*perform)(struct gralloc_module_t const* module,
                int operation, ... );

        // 锁定图形缓冲区，输出参数是android_ycbcr类型
        int (*lock_ycbcr)(struct gralloc_module_t const* module,
                buffer_handle_t handle, int usage,
                int l, int t, int w, int h,
                struct android_ycbcr *ycbcr);

        // 保留的成员
        void* reserved_proc[6];
    } gralloc_module_t;

说明：该代码在hardware/libhardware/include/hardware/gralloc.h中。gralloc_module_t是Gralloc模块对应的结构体，它是Gralloc提供给SurfaceFlinger等外部程序使用的结构体；外部进程可以通过该它提供的API来进行注册、注销缓冲区等操作。




<a name="anchor1_2_3"></a>
### 1.2.3 private_module_t

    struct private_module_t {
        gralloc_module_t base;

        private_handle_t* framebuffer;    // private_handle_t是描述图形缓冲区的结构体
        uint32_t flags;                   // 是否支持双缓冲标志。PAGE_FLIP位是1则支持，否则不支持。
        uint32_t numBuffers;              // 缓冲区的个数
        uint32_t bufferMask;              // 缓冲区使用情况。假如有两个缓冲区，0x11表示两个都被分配使用了；0x01表示只有第1个被分配使用了。
        pthread_mutex_t lock;             // 互斥锁
        buffer_handle_t currentBuffer;    // 当前缓冲区(目前正在被渲染的图形缓冲区)
        int pmem_master;
        void* pmem_master_base;

        struct fb_var_screeninfo info;    // 显示屏属性信息。该成员是用来存储可设置的属性。
        struct fb_fix_screeninfo finfo;   // 显示屏属性信息。该成员是用来存储只读属性。
        float xdpi;                       // x轴的显示密度(x轴上每英寸多少像素)
        float ydpi;                       // y轴的显示密度(x轴上每英寸多少像素)
        float fps;                        // 显示屏的帧率(每秒多少帧)
    };

说明：该代码在hardware/libhardware/modules/gralloc/gralloc_priv.h中。private_module_t，顾名思义，它是私有结构体。对于Gralloc模块而言，它就是描述Gralloc的私有信息的结构体，这些私有信息主要包括"缓冲区"和"显示屏的信息"。




<a name="anchor1_3"></a>
## 1.3 其他结构体

private_module_t中包含了private_handle_t和buffer_handle_t这两种结构体。下面分别对它们进行介绍。

<a name="anchor1_3_1"></a>
### 1.3.1 buffer_handle_t和native_handle_t

下面是buffer_handle_t的定义。

    typedef const native_handle_t* buffer_handle_t;

说明：该代码在system/core/include/system/window.h中。由此可见，buffer_handle_t实际上是native_handle_t*类型。


    typedef struct native_handle
    {
        int version;        // 版本标识。实际上大小等于size_of(native_handle_t)
        int numFds;         // data[0]中文件描述符的个数
        int numInts;        // data[0]中整型数的个数
        int data[0];        // 数据(包括文件描述符和整型数)
    } native_handle_t;


说明：该代码在system/core/include/cutils/native_handle.h中。native_handle_t是HAL中描述句柄信息的结构体，它主要包含"文件描述符"和"整型数"这些内容，但是它们是干吗的呢？暂时还无从得知；但看过它的子类private_handle_t之类，就能了解这些成员的用途了。


<a name="anchor1_3_1"></a>
### 1.3.1 private_handle_t

    #ifdef __cplusplus
    struct private_handle_t : public native_handle {
    #else
    struct private_handle_t {
        struct native_handle nativeHandle;
    #endif

        enum {
            PRIV_FLAGS_FRAMEBUFFER = 0x00000001
        };

        // file-descriptors
        int     fd;          // 文件描述符
        // ints
        int     magic;       // 魔数
        int     flags;       // 图形缓冲区的标记
        int     size;        // 图形缓冲区的大小
        int     offset;      // 图形缓冲区的偏移

        // FIXME: the attributes below should be out-of-line
        int     base;        // 图形缓冲区的实际地址(加上了offset偏移)
        int     pid;         // 创建图形缓冲区的进程的pid

    #ifdef __cplusplus
        static const int sNumInts = 6;
        static const int sNumFds = 1;
        static const int sMagic = 0x3141592;

        private_handle_t(int fd, int size, int flags) :
            fd(fd), magic(sMagic), flags(flags), size(size), offset(0),
            base(0), pid(getpid())
        {
            version = sizeof(native_handle);
            numInts = sNumInts;
            numFds = sNumFds;
        }
        ~private_handle_t() {
            magic = 0;
        }

        static int validate(const native_handle* h) {
            const private_handle_t* hnd = (const private_handle_t*)h;
            if (!h || h->version != sizeof(native_handle) ||
                    h->numInts != sNumInts || h->numFds != sNumFds ||
                    hnd->magic != sMagic)
            {
                ALOGE("invalid gralloc handle (at %p)", h);
                return -EINVAL;
            }
            return 0;
        }
    #endif
    };

说明：该代码在hardware/libhardware/modules/gralloc/gralloc_priv.h中。private_handle_t是Gralloc中描述图形缓冲区的句柄的结构体，它在C编译器和C++编译器中编译得到的结果是不一样的。假设是在C++环境中编译的gralloc_priv.h，即编译环境定义有宏__cplusplus。那么，结构体private_handle_t就是从结构体native_handle_t继承下来的，它包含有1个文件描述符以及6个整数，以及三个静态成员变量。  
(01)  
(02)  
(03) flag是图形缓冲区的标记。当flag的PRIV_FLAGS_FRAMEBUFFER位是1时，表示该图形缓冲区是从系统帧缓冲区中分配的；否则，该图形缓冲区是从内存中分配的。


