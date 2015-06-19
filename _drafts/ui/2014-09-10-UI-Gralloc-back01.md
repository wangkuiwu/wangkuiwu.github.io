---
layout: post
title: "Android UI系统(一) 硬件抽象层Gralloc"
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
# 1. Gralloc概述

Gralloc是Android中的硬件抽象层的模块，它封装了对帧缓冲区的所有访问操作。什么是帧缓冲区呢？显示屏中的整屏画面就是一帧，而帧缓冲区则是保存有一帧画面的内存缓冲区。




<a name="anchor2"></a>
# 2. Gralloc的加载流程

<a name="anchor2_1"></a>
## 2.1 SurfaceFlinger中加载Gralloc的入口

Gralloc模块是在GraphicBufferAllocator.cpp中加载并打开的。

    GraphicBufferAllocator::GraphicBufferAllocator()
        : mAllocDev(0)
    {
        hw_module_t const* module;
        int err = hw_get_module(GRALLOC_HARDWARE_MODULE_ID, &module);
        ALOGE_IF(err, "FATAL: can't find the %s module", GRALLOC_HARDWARE_MODULE_ID);
        if (err == 0) {
            gralloc_open(module, &mAllocDev);
        }   
    }

说明：该代码在frameworks/native/libs/ui/GraphicBufferAllocator.cpp中。GraphicBufferAllocator是SurfaceFlinger服务中用来分配GraphicBuffer的类，这点以后在介绍SurfaceFlinger时会详细说明。现在，只需要了解在SurfaceFlinger中会通过hw_get_module()加载Gralloc口即可！

下面，我们就看看hw_get_module()的实现。


<a name="anchor2_2"></a>
## 2.2 hw_get_module()

    int hw_get_module(const char *id, const struct hw_module_t **module)
    {
        return hw_get_module_by_class(id, NULL, module);
    }

说明：该代码在hardware/libhardware/hardware.c中。它是通过调用hw_get_module_by_class()来实现的，在介绍hw_get_module_by_class()之前，先说说hw_get_module()的作用和参数。

  hw_get_module(id, module)的作用是加载Android中的硬件抽象层，即HAL层(Hardware Abstraction Layer)。  
  顾名思义，硬件抽象层是用硬件的抽象表示，将硬件的功能都封装到软件接口中；它是以.so库的形式存在于系统中的。hw_get_module(id, module)就是加载id对应的.so库，并将.so库映射到结构体变量module中；之后，通过操作module，就相当于操作硬件。其中，id是硬件抽象层模块的身份标识；例如，Gralloc的id是GRALLOC_HARDWARE_MODULE_ID；而module则是hw_module_t**类型的指针。 

Gralloc的id定义在hardware/libhardware/include/hardware/gralloc.h中，代码如下：

    #define GRALLOC_HARDWARE_MODULE_ID "gralloc"

hw_module_t定义在hardware/libhardware/include/hardware/hardware.h中，代码如下：

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

说明：hw_module_t中记录了模块的身份信息(标识，id，版本号，名称，作者)以及 模块的方法api(methods成员)。



<a name="anchor2_3"></a>
## 2.3 hw_get_module_by_class()

下面看看hw_module_t()调用的hw_get_module_by_class()的源码。

    #define HAL_LIBRARY_PATH1 "/system/lib/hw"
    #define HAL_LIBRARY_PATH2 "/vendor/lib/hw"

    static const char *variant_keys[] = {
        "ro.hardware",
        "ro.product.board",
        "ro.board.platform",
        "ro.arch"
    };

    static const int HAL_VARIANT_KEYS_COUNT =
        (sizeof(variant_keys)/sizeof(variant_keys[0]));


    ...

    int hw_get_module_by_class(const char *class_id, const char *inst,
                               const struct hw_module_t **module)
    {
        int status;
        int i;
        const struct hw_module_t *hmi = NULL;
        char prop[PATH_MAX];
        char path[PATH_MAX];
        char name[PATH_MAX];

        if (inst)
            snprintf(name, PATH_MAX, "%s.%s", class_id, inst);
        else
            strlcpy(name, class_id, PATH_MAX);

        snprintf(prop, sizeof(path), "wmt.%s.so.path", name);
        if (property_get(prop, path, NULL) != 0) {
            if (access(path, R_OK) == 0) {
                i = 0;
                goto find_out;
            }
            else {
            }
        }

        for (i=0 ; i<HAL_VARIANT_KEYS_COUNT+1 ; i++) {
            if (i < HAL_VARIANT_KEYS_COUNT) {
                if (property_get(variant_keys[i], prop, NULL) == 0) {
                    continue;
                }
                snprintf(path, sizeof(path), "%s/%s.%s.so",
                         HAL_LIBRARY_PATH2, name, prop);
                if (access(path, R_OK) == 0) break;

                snprintf(path, sizeof(path), "%s/%s.%s.so",
                         HAL_LIBRARY_PATH1, name, prop);
                if (access(path, R_OK) == 0) break;
            } else {
                snprintf(path, sizeof(path), "%s/%s.default.so",
                         HAL_LIBRARY_PATH2, name);
                if (access(path, R_OK) == 0) break;

                snprintf(path, sizeof(path), "%s/%s.default.so",
                         HAL_LIBRARY_PATH1, name);
                if (access(path, R_OK) == 0) break;
            }
        }

    find_out :
        status = -ENOENT;
        if (i < HAL_VARIANT_KEYS_COUNT+1) {
            status = load(class_id, path, module);
        }

        return status;
    }

说明：该函数会依次在目录"/system/lib/hw"和"/vendor/lib/hw"中查找一个名称为"<class_id>.<variant_key>.so"的文件。  
(01) <class_id>表示模块id。例如，对于Gralloc而言，class_id就是"gralloc"。  
(02) <variant_key>表示"ro.hardware"、"ro.product.board"、"ro.board.platform"和"ro.arch"这四个系统属性值之一。例如，对于Gralloc来说，该函数会查找是否存在 gralloc.ro.hardware.so, gralloc.ro.product.board.so, gralloc.ro.board.platform.so, gralloc.<ro.arch>.so这四个库文件。   
  只要找到一个库文件就停止查找；然后，通过load()来加载该库文件。但是，如果没有找到所需要的库文件，则会在目录/system/lib/hw中查找是否存在一个名称为gralloc.default.so的默认文件。如果存在的话，那么也会调用函数load()将它加载到内存中来。



<a name="anchor2_4"></a>
## 2.4 load()

	static int load(const char *id,
			const char *path,
			const struct hw_module_t **pHmi)
	{
		int status;
		void *handle;
		struct hw_module_t *hmi;

		handle = dlopen(path, RTLD_NOW);
		...

		const char *sym = HAL_MODULE_INFO_SYM_AS_STR;
		hmi = (struct hw_module_t *)dlsym(handle, sym);
		if (hmi == NULL) {
			...
		}   

		if (strcmp(id, hmi->id) != 0) {
			...
		}   

		hmi->dso = handle;

		status = 0;

		...

		*pHmi = hmi;

		return status;
	}

说明：该函数的作用是加载.so库，并将加载后的HAL模块映射到hw_module_t中。  
(01) dlopen()的作用是打开.so库。  
(02) dlsym(handle, sym)的作用是在.so库中查找符号为sym的变量，并返回该变量的地址；然后将其返回的地址转换为hw_module_t\*类型的指针。先将load()介绍完，后面再详细说说说说dlsym()的原理。  
(03) 在得到hw_module_t*类型的变量hmi之后，就将其赋值给pHmi然后返回。



<a name="anchor2_5"></a>
## 2.5 HAL_MODULE_INFO_SYM_AS_STR符号的原理

先看看HAL_MODULE_INFO_SYM_AS_STR的定义

	#define HAL_MODULE_INFO_SYM         HMI

	#define HAL_MODULE_INFO_SYM_AS_STR  "HMI"

说明：该宏定义在hardware/libhardware/include/hardware/hardware.h中。HAL_MODULE_INFO_SYM_AS_STR的名称是字符串HMI，而HAL_MODULE_INFO_SYM的值正好是HMI(符号)。 Gralloc中的HAL_MODULE_INFO_SYM符号对应的是private_module_t类型的变量，定义如下。


	static struct hw_module_methods_t gralloc_module_methods = {
			open: gralloc_device_open
	};  
			
	struct private_module_t HAL_MODULE_INFO_SYM = {
		base: { 
			common: {
				tag: HARDWARE_MODULE_TAG,
				version_major: 1,
				version_minor: 0,
				id: GRALLOC_HARDWARE_MODULE_ID,
				name: "Graphics Memory Allocator Module",
				author: "The Android Open Source Project",
				methods: &gralloc_module_methods
			},
			registerBuffer: gralloc_register_buffer,
			unregisterBuffer: gralloc_unregister_buffer,
			lock: gralloc_lock,
			unlock: gralloc_unlock,
		},
		framebuffer: 0,
		flags: 0,
		numBuffers: 0,
		bufferMask: 0,
		lock: PTHREAD_MUTEX_INITIALIZER,
		currentBuffer: 0,
	};

说明：该代码在hardware/libhardware/modules/gralloc/gralloc.cpp中。HAL_MODULE_INFO_SYM就是我们要查找的符号。  
在load()中会将该符号对应的地址转换为hw_module_t类型；即，将private_module_t转换为hw_module_t类型。

但是，private_module_t类型的变量是如何转换为hw_module_t类型的呢？

[skywang-todo]

如上图所示，private_module_t的第一个成员变量base指向一个gralloc_module_t结构体，而gralloc_module_t的第一个成员变量common又指向了一个hw_module_t结构体。这意味着，private_module_t结构体的指针可以用作一个gralloc_module_t或者hw_module_t结构体提针来使用。事实上，这是使用C语言来实现的一种继承关系，等价于结构体private_module_t继承结构体gralloc_module_t，而结构体gralloc_module_t继承hw_module_t结构体。   
这样，我们就可以把在Gralloc模块中定义的符号HAL_MODULE_INFO_SYM看作是一个hw_module_t结构体。



<a name="anchor2_6"></a>
## 2.6 private_module_t和hw_module_t的代码

### 2.6.1 private_module_t

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

说明：该代码在hardware/libhardware/modules/gralloc/gralloc_priv.h中。private_module_t，顾名思义，它是Gralloc的私有结构体。它是Gralloc用来描述私有信息的结构体，这些私有信息主要包括"缓冲区"和"显示屏的信息"。  
下面，我们就先看看base对应的gralloc_module_t结构体，然后再分析private_handle_t和buffer_handle_t。


### 2.6.2 gralloc_module_t

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

说明：该代码在hardware/libhardware/include/hardware/gralloc.h中。它是一个对外结构体，外部可以通过该它的注册缓冲区/注销缓冲区/锁定缓冲区/解锁缓冲区等API对缓冲区进行操作。



### 2.6.3 private_handle_t 

下面，看看private_module_t中包含的private_handle_t和buffer_handle_t这两种结构体。先来看buffer_handle_t，它的定义如下：

    typedef const native_handle_t* buffer_handle_t;

说明：该代码在system/core/include/system/window.h中。由此可见，buffer_handle_t实际上是native_handle_t*类型。


    typedef struct native_handle
    {
        int version;        // 版本标识。实际上大小等于size_of(native_handle_t)
        int numFds;         // data[0]中文件描述符的个数
        int numInts;        // data[0]中整型数的个数
        int data[0];        // 数据(包括文件描述符和整型数)
    } native_handle_t;


说明：该代码在system/core/include/cutils/native_handle.h中。native_handle_t主要包含"文件描述符"和"整型数"这些内容，但是它们是干吗的呢？暂时还无从得知；但看过它的子类private_handle_t之类，就能了解这些成员的用途了。


接下来，就看看private_handle_t结构体，代码如下：

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

说明：该代码在hardware/libhardware/modules/gralloc/gralloc_priv.h中。private_handle_t在C编译器和C++编译器中编译得到的结果是不一样的。假设是在C++环境中编译的gralloc_priv.h，即编译环境定义有宏__cplusplus。那么，结构体private_handle_t就是从结构体native_handle_t继承下来的，它包含有1个文件描述符以及6个整数，以及三个静态成员变量。  
(01) fd是图形缓冲区的句柄。要么指向帧缓冲区设备，要么指向一块匿名共享内存，取决于它的宿主结构体private_handle_t描述的一个图形缓冲区是在帧缓冲区分配的，还是在内存中分配的。  

> 这里简要说明一下"帧缓冲区 和 内存中分配的图形缓冲区的差别"。




<a name="anchor3"></a>
# 3. Gralloc的打开流程

在SurfaceFlinger中加载Gralloc之后，就会调用gralloc_open()来打开Gralloc。

    GraphicBufferAllocator::GraphicBufferAllocator()
        : mAllocDev(0)
    {
        hw_module_t const* module;
        int err = hw_get_module(GRALLOC_HARDWARE_MODULE_ID, &module);
        ALOGE_IF(err, "FATAL: can't find the %s module", GRALLOC_HARDWARE_MODULE_ID);
        if (err == 0) {
            gralloc_open(module, &mAllocDev);
        }   
    }

<a name="anchor3_1"></a>
## 3.1 gralloc_open

下面就看看gralloc_open()的流程。

    #define GRALLOC_HARDWARE_GPU0 "gpu0"

    static inline int gralloc_open(const struct hw_module_t* module, 
            struct alloc_device_t** device) {
        return module->methods->open(module, 
                GRALLOC_HARDWARE_GPU0, (struct hw_device_t**)device);
    }

说明：该代码在hardware/libhardware/include/hardware/gralloc.h中。  
(01) module就是前面介绍的Gralloc对应的module，module->methods->open()对应会执行gralloc_device_open()方法。  
(02) GRALLOC_HARDWARE_GPU0的值是字符串"gpu0"。


<a name="anchor3_2"></a>
## 3.2 gralloc_device_open

    int gralloc_device_open(const hw_module_t* module, const char* name,
            hw_device_t** device)
    {
        int status = -EINVAL;
        if (!strcmp(name, GRALLOC_HARDWARE_GPU0)) {
            gralloc_context_t *dev;
            dev = (gralloc_context_t*)malloc(sizeof(*dev));

            memset(dev, 0, sizeof(*dev));

            dev->device.common.tag = HARDWARE_DEVICE_TAG;
            dev->device.common.version = 0;
            dev->device.common.module = const_cast<hw_module_t*>(module);
            dev->device.common.close = gralloc_close;

            dev->device.alloc   = gralloc_alloc;
            dev->device.free    = gralloc_free;

            *device = &dev->device.common;
            status = 0;
        } else {
            status = fb_device_open(module, name, device);
        }
        return status;
    }       

说明：该代码在hardware/libhardware/modules/gralloc/gralloc.cpp中。由于name的值等于GRALLOC_HARDWARE_GPU0，因此，该函数会先创建一个gralloc_context_t变量dev，然后对dev进行初始化。




<a name="anchor4"></a>
# 4. FB的打开流程

前面介绍了SurfaceFlinger中Gralloc的加载和打开流程。下面看看SurfaceFlinger中打开fb设备的流程。

    int HWComposer::loadFbHalModule()
    {
        hw_module_t const* module;

        int err = hw_get_module(GRALLOC_HARDWARE_MODULE_ID, &module);
        if (err != 0) {
            ALOGE("%s module not found", GRALLOC_HARDWARE_MODULE_ID);
            return err;
        }

        return framebuffer_open(module, &mFbDev);
    }

说明：该代码在frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp。我们暂不深究loadFbHalModule()是何时被调用的，只需要知道SurfaceFlinger中会通过该接口打开fb设备即可！  
(01) hw_get_module()是加载Gralloc。这在前面已经详细介绍过了。  
(02) framebuffer_open()是打开fb设备，并将fb设备的相关信息保存到mFbDev中。


<a name="anchor4_1"></a>
## 4.1 framebuffer_open

    #define GRALLOC_HARDWARE_FB0 "fb0"

    static inline int framebuffer_open(const struct hw_module_t* module,                            
            struct framebuffer_device_t** device) {
        return module->methods->open(module,
                GRALLOC_HARDWARE_FB0, (struct hw_device_t**)device);
    }

说明：该代码在hardware/libhardware/include/hardware/fb.h中。  
(01) module->methods->open()对应会执行gralloc_device_open()方法。  
(02) GRALLOC_HARDWARE_FB0的值是字符串"fb0"。



<a name="anchor4_2"></a>
## 4.2 gralloc_device_open

    int gralloc_device_open(const hw_module_t* module, const char* name,
            hw_device_t** device)
    {
        int status = -EINVAL;
        if (!strcmp(name, GRALLOC_HARDWARE_GPU0)) {
            gralloc_context_t *dev;
            dev = (gralloc_context_t*)malloc(sizeof(*dev));

            memset(dev, 0, sizeof(*dev));

            dev->device.common.tag = HARDWARE_DEVICE_TAG;
            dev->device.common.version = 0;
            dev->device.common.module = const_cast<hw_module_t*>(module);
            dev->device.common.close = gralloc_close;

            dev->device.alloc   = gralloc_alloc;
            dev->device.free    = gralloc_free;

            *device = &dev->device.common;
            status = 0;
        } else {
            status = fb_device_open(module, name, device);
        }
        return status;
    }       

说明：由于GRALLOC_HARDWARE_GPU0的值是"gpu0"，而name的值是"fb0"；因此，该函数会调用fb_device_open()来打开fb设备。



<a name="anchor4_3"></a>
## 4.3 fb_device_open

    int fb_device_open(hw_module_t const* module, const char* name,
            hw_device_t** device)
    {
        int status = -EINVAL;
        if (!strcmp(name, GRALLOC_HARDWARE_FB0)) {
            fb_context_t *dev = (fb_context_t*)malloc(sizeof(*dev));
            memset(dev, 0, sizeof(*dev));

            /* initialize the procs */
            dev->device.common.tag = HARDWARE_DEVICE_TAG;
            dev->device.common.version = 0;
            dev->device.common.module = const_cast<hw_module_t*>(module);
            dev->device.common.close = fb_close;
            dev->device.setSwapInterval = fb_setSwapInterval;
            dev->device.post            = fb_post;
            dev->device.setUpdateRect = 0;

            private_module_t* m = (private_module_t*)module;
            status = mapFrameBuffer(m);
            if (status >= 0) {
                int stride = m->finfo.line_length / (m->info.bits_per_pixel >> 3); 
                int format = (m->info.bits_per_pixel == 32) 
                             ? HAL_PIXEL_FORMAT_RGBX_8888
                             : HAL_PIXEL_FORMAT_RGB_565;
                const_cast<uint32_t&>(dev->device.flags) = 0;
                const_cast<uint32_t&>(dev->device.width) = m->info.xres;
                const_cast<uint32_t&>(dev->device.height) = m->info.yres;
                const_cast<int&>(dev->device.stride) = stride;
                const_cast<int&>(dev->device.format) = format;
                const_cast<float&>(dev->device.xdpi) = m->xdpi;
                const_cast<float&>(dev->device.ydpi) = m->ydpi;
                const_cast<float&>(dev->device.fps) = m->fps;
                const_cast<int&>(dev->device.minSwapInterval) = 1;
                const_cast<int&>(dev->device.maxSwapInterval) = 1;
                *device = &dev->device.common;
            }   
        }   
        return status;
    }

说明：该函数在hardware/libhardware/modules/gralloc/framebuffer.cpp中。此时的name等于GRALLOC_HARDWARE_FB0；因此，该函数会新建一个fb_context_t*类型的变量dev，然后对其进行初始化。


<a name="anchor4_4"></a>
## 4.4 mapFrameBuffer




前面分别了解Gralloc的加载和打开流程：在SurfaceFlinger中会加载Gralloc，加载完毕之后会打开Gralloc。而且，我们知道Gralloc是显示屏的硬件抽象层，它封装了操作显示屏的接口，通过调用Gralloc的注册帧缓冲/注销帧缓冲等待接口可以方便的对显示屏的帧缓冲区进行操作。
