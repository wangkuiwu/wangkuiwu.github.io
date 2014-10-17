---
layout: post
title: "Android UI系统(三) 硬件抽象层的Gralloc模块和FB模块"
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

Android设备的显示屏被抽象为一个帧缓冲区。而且在硬件抽象层，Android提供了2个抽象硬件模块：Gralloc模块和FB模块。通过这2个抽象模块，就可以实现对帧缓冲区的操作。 Gralloc模块中封装了申请/释放图形缓冲区的操作，而FB模块封装了将图形缓冲区中的内容更新到显示屏的操作。

举例来说，当应用程序要在显示屏上显示图像时，它会先通过Gralloc模块来申请一个图形缓冲区，并将这个图形缓冲区映射到应用程序的地址空间上。接着，应用程序将要绘制的内容填入到图形缓冲区。当绘制内容被写入到图形缓冲区之后，应用程序就可以通过FB模块来将图形缓冲区的内容渲染到帧缓冲区中去，即将图形缓冲区的内容绘制到显示屏中去。


了解了Gralloc模块和FB模块的作用之后，下面说说它们在HAL层的数据结构。先看看它们的类图。

[skywang-todo]



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

说明：该代码在frameworks/native/libs/ui/GraphicBufferAllocator.cpp中。GraphicBufferAllocator是SurfaceFlinger服务中用来分配GraphicBuffer的类，至于SurfaceFlinger是调用到GraphicBufferAllocator的，这点不用深究。我们只需要知道在SurfaceFlinger中会通过hw_get_module()加载Gralloc模块即可！

下面，我们就看看hw_get_module()的实现。


<a name="anchor2_2"></a>
## 2.2 hw_get_module()

    int hw_get_module(const char *id, const struct hw_module_t **module)
    {
        return hw_get_module_by_class(id, NULL, module);
    }

说明：该代码在hardware/libhardware/hardware.c中。它是通过调用hw_get_module_by_class()来实现的，在介绍hw_get_module_by_class()之前，先说说hw_get_module()的作用和参数。

  hw_get_module(id, module)的作用是加载Android中的硬件抽象层，即HAL层(Hardware Abstraction Layer)。  
  顾名思义，硬件抽象层是用硬件的抽象表示，将硬件的功能都封装到软件接口中；它是以.so库的形式存在于系统中的。hw_get_module(id, module)就是加载id对应的.so库，并将.so库映射到结构体变量module中；之后，通过操作module，就相当于操作硬件。其中，id是硬件抽象层模块的身份标识；例如，Gralloc的id是GRALLOC_HARDWARE_MODULE_ID；而module则是hw_module_t**类型的指针，hw_module_t是HAL中描述硬件模块的结构体，它的详细介绍可以参考[Android UI系统(二) UI中的数据结构][link_ui_02_datastruct]。

Gralloc模块的id定义在hardware/libhardware/include/hardware/gralloc.h中，代码如下：

    #define GRALLOC_HARDWARE_MODULE_ID "gralloc"



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

不同的厂商可能会提供不同的Gralloc库。它们的原理都是一样的，因此在后面的介绍中，我们会以默认的Gralloc库，即gralloc.default.so来进行介绍！


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
(02) dlsym(handle, sym)的作用是在.so库中查找符号为sym的变量，并返回该变量的地址；然后将其返回的地址转换为hw_module_t*类型的指针。先将load()介绍完，后面再详细说说说说dlsym()的原理。  
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
在load()中会将该符号对应的地址转换为hw_module_t类型；即，将private_module_t转换为hw_module_t类型。关于private_module_t和hw_module_t的介绍，请参考[Android UI系统(二) UI中的数据结构][link_ui_02_datastruct]。

至于private_module_t类型的变量是如何转换为hw_module_t类型的呢。很简单，如下图所示。

[skywang-todo]

private_module_t的第一个成员变量base指向一个gralloc_module_t结构体，而gralloc_module_t的第一个成员变量common又指向了一个hw_module_t结构体。这意味着，private_module_t结构体的指针可以用作一个gralloc_module_t或者hw_module_t结构体提针来使用。事实上，这是使用C语言来实现的一种继承关系，等价于结构体private_module_t继承结构体gralloc_module_t，而结构体gralloc_module_t继承hw_module_t结构体。   
这样，我们就可以把在Gralloc模块中定义的符号HAL_MODULE_INFO_SYM看作是一个hw_module_t结构体。


<br/>
截止到目前，我们成功加载了Gralloc模块，并且将获取到的Gralloc模块信息保存在hw_module_t结构体变量module中。  
下面，我们接着看看打开Gralloc模块的流程。



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

<br/>
也就是说，在打开Gralloc模块时，我们获取到了表示Gralloc这个抽象硬件设备的信息，并将其保存到mAllocDev中。




<a name="anchor4"></a>
# 4. FB的打开流程

前面介绍Gralloc模块的加载以及打开过程，下面再开开FB模块的打开过程。

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

说明：该代码在frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp。我们暂不深究loadFbHalModule()是何时被调用的，只需要知道SurfaceFlinger中会通过该接口打开FB模块即可！  
(01) hw_get_module()是加载so库。Gralloc模块和FB模块都是在同一个so库中，而在前面已经介绍过了加载该so库的过程。  
(02) framebuffer_open()是打开FB模块，并将FB模块的相关信息保存到mFbDev中。


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
            ...
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

说明：该函数在hardware/libhardware/modules/gralloc/framebuffer.cpp中。此时的name等于GRALLOC_HARDWARE_FB0；因此，该函数会新建一个fb_context_t*类型的变量dev，然后对其进行初始化。fb_context_t是描述FB模块的上下文环境的结构体，它仅仅只包含了一个成员framebuffer_device_t。在初始化dev之后，还会通过mapFrameBuffer()来读取显示屏的相关数据，并将系统缓冲区映射到用户空间。

下面是fb_context_t的代码。

    struct fb_context_t {
        framebuffer_device_t  device;
    };


<a name="anchor4_4"></a>
## 4.4 mapFrameBuffer

    static int mapFrameBuffer(struct private_module_t* module)
    {
        pthread_mutex_lock(&module->lock);
        int err = mapFrameBufferLocked(module);
        pthread_mutex_unlock(&module->lock);
        return err;
    }

说明：该函数会调用mapFrameBufferLocked()。



<a name="anchor4_5"></a>
## 4.5 mapFrameBufferLocked

    #define NUM_BUFFERS 2

    int mapFrameBufferLocked(struct private_module_t* module)
    {
        // 帧缓冲区已经初始化，则返回
        if (module->framebuffer) {
            return 0;
        }

        char const * const device_template[] = {
                "/dev/graphics/fb%u",
                "/dev/fb%u",
                0 };

        int fd = -1;
        int i=0;
        char name[64];

        // 按照先后顺序尝试打开"/dev/graphics/fb0"和"/dev/fb0"；
        // 成功打开一个就退出循环
        while ((fd==-1) && device_template[i]) {
            snprintf(name, 64, device_template[i], 0);
            fd = open(name, O_RDWR, 0);
            i++;
        }
        if (fd < 0)
            return -errno;

        // 通过ioctl()获取显示屏的"只读信息"，并保存到finfo中。
        struct fb_fix_screeninfo finfo;
        if (ioctl(fd, FBIOGET_FSCREENINFO, &finfo) == -1)
            return -errno;

        // 通过ioctl()获取显示屏的"可读写信息"，并保存到info中。
        struct fb_var_screeninfo info;
        if (ioctl(fd, FBIOGET_VSCREENINFO, &info) == -1)
            return -errno;

        // 初始化info中的部分成员
        info.reserved[0] = 0;
        info.reserved[1] = 0;
        info.reserved[2] = 0;
        info.xoffset = 0;
        info.yoffset = 0;
        info.activate = FB_ACTIVATE_NOW;

        // NUM_BUFFERS的值为2
        // 设置"显示屏的虚拟分辨率" 为 "实际分辨率"的2倍
        info.yres_virtual = info.yres * NUM_BUFFERS;

        // 通过ioctl()设置显示屏的信息，包括设置显示屏的虚拟分辨率。
        uint32_t flags = PAGE_FLIP;
        if (ioctl(fd, FBIOPUT_VSCREENINFO, &info) == -1) {
            ...
        }

        // 判断虚拟分辨率是否设置成功(检查硬件是否支持双缓冲)
        if (info.yres_virtual < info.yres * 2) {
            ...
        }

        // 再次通过ioctl()获取显示屏的"可读写信息"
        if (ioctl(fd, FBIOGET_VSCREENINFO, &info) == -1)
            return -errno;

        uint64_t  refreshQuotient =
        (
                uint64_t( info.upper_margin + info.lower_margin + info.yres )
                * ( info.left_margin  + info.right_margin + info.xres )
                * info.pixclock
        );

        // 设置显示屏的刷新频率
        int refreshRate = refreshQuotient > 0 ? (int)(1000000000000000LLU / refreshQuotient) : 0;

        if (refreshRate == 0) {
            // bleagh, bad info from the driver
            refreshRate = 60*1000;  // 60 Hz
        }

        if (int(info.width) <= 0 || int(info.height) <= 0) {
            // the driver doesn't return that information
            // default to 160 dpi
            info.width  = ((info.xres * 25.4f)/160.0f + 0.5f);
            info.height = ((info.yres * 25.4f)/160.0f + 0.5f);
        }

        // 屏幕密度和帧率
        float xdpi = (info.xres * 25.4f) / info.width;
        float ydpi = (info.yres * 25.4f) / info.height;
        float fps  = refreshRate / 1000.0f;


        // 再次通过ioctl()获取显示屏的"只读信息"，并保存到finfo中。
        if (ioctl(fd, FBIOGET_FSCREENINFO, &finfo) == -1)
            return -errno;

        if (finfo.smem_len <= 0)
            return -errno;

        module->flags = flags;
        module->info = info;
        module->finfo = finfo;
        module->xdpi = xdpi;
        module->ydpi = ydpi;
        module->fps = fps;

        int err;
        // finfo.line_length * info.yres_virtual是计算的是整个系统帧缓冲区的大小。
        // 其中，info.yres_virtual等于"虚拟分辨率的高度"，finfo.line_length等于"每一行所占用的字节数"。
        // 而roundUpToPageSize()的作用是进行字节对齐。
        size_t fbSize = roundUpToPageSize(finfo.line_length * info.yres_virtual);
        // 根据缓冲区大小创建private_handle_t变量，private_handle_t是Gralloc中描述图形缓冲区的结构体。
        module->framebuffer = new private_handle_t(dup(fd), fbSize, 0);

        // 设置系统帧缓冲区的个数
        module->numBuffers = info.yres_virtual / info.yres;
        // 设置系统帧缓冲区的使用情况，0x00表示两个系统帧缓冲区都没有被使用。
        module->bufferMask = 0;

        // 通过mmap()将系统帧缓冲区映射到当前进程所在的用户空间地址中。
        void* vaddr = mmap(0, fbSize, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);
        if (vaddr == MAP_FAILED) {
            ALOGE("Error mapping the framebuffer (%s)", strerror(errno));
            return -errno;
        }
        module->framebuffer->base = intptr_t(vaddr);
        memset(vaddr, 0, fbSize);
        return 0;
    }

说明：mapFrameBufferLocked()的作用主要有两个：第一，将系统缓冲区映射到用户空间。第二，将映射到用户空间的缓冲区地址，以及显示屏的其他信息都保存到module变量中。而module是Gralloc模块的私有结构体private_module_t类型的变量。主要步骤如下：  
(01) 通过open()打开显示屏对应的文件节点，并将文件句柄保存到fd中。  
(02) 接着，获取显示屏的信息，包括显示屏的实际分辨率。然后根据显示屏的实际分辨率来设置它的虚拟分辨率，将虚拟分辨率设置为实际分辨率的2倍，并根据返回结果判断是否设置成功。即，查看显示屏是否支持双缓冲技术。本文的描述，都是假设显示屏是支持双缓冲技术，即虚拟分辨率是实际分辨率的2倍。  
(03) 然后，读取显示屏的显示密度和帧率等信息，并保存到module中。  
(04) 之后，将系统的帧缓冲区映射到用户空间。并将系统帧缓冲的相关信息，包括系统缓冲区个数、大小、在用户空间的地址等等信息都保存到module中。





<a name="anchor5"></a>
# 5. 分配图形缓冲区


<a name="anchor5_1"></a>
## 5.1 gralloc_alloc

    static int gralloc_alloc(alloc_device_t* dev,
            int w, int h, int format, int usage,
            buffer_handle_t* pHandle, int* pStride)
    {
        if (!pHandle || !pStride)
            return -EINVAL;

        size_t size, stride;

        int align = 4;
        int bpp = 0;
        switch (format) {
            case HAL_PIXEL_FORMAT_RGBA_8888:
            case HAL_PIXEL_FORMAT_RGBX_8888:
            case HAL_PIXEL_FORMAT_BGRA_8888:
                bpp = 4;
                break;
            case HAL_PIXEL_FORMAT_RGB_888:
                bpp = 3;
                break;
            case HAL_PIXEL_FORMAT_RGB_565:
            case HAL_PIXEL_FORMAT_RAW_SENSOR:
                bpp = 2;
                break;
            default:
                return -EINVAL;
        }
        size_t bpr = (w*bpp + (align-1)) & ~(align-1);
        size = bpr * h;
        stride = bpr / bpp;

        int err;
        if (usage & GRALLOC_USAGE_HW_FB) {
            err = gralloc_alloc_framebuffer(dev, size, usage, pHandle);
        } else {
            err = gralloc_alloc_buffer(dev, size, usage, pHandle);
        }

        if (err < 0) {
            return err;
        }

        *pStride = stride;
        return 0;
    }

说明：gralloc_alloc()的作用是分配图形缓冲区。   
(01) 先对各个参数进行说明。dev是描述Gralloc设备的变量。w和h是描述要分配的图形缓冲区的宽和高的。format是图形数据中每个像素的格式，例如对于HAL_PIXEL_FORMAT_RGBX_8888而言，每个像素要32位(R,G,B个8位，剩余8位表示透明度)。usage是图形缓冲区是用途，如果是用来在系统帧缓冲区中渲染的，即参数usage的GRALLOC_USAGE_HW_FB位等于1，那么就必须要系统帧缓冲区中分配，否则的话，就在内存中分配。pHandle和pStride是输出参数，pHandle是描述图形缓冲区的句柄，而pStride则是用来记录一行多少像素点的。  
(02) gralloc_alloc()会依据用途来分配图形缓冲区。如果图形缓冲区是用来在系统帧缓冲区中进行渲染的，即usage的GRALLOC_USAGE_HW_FB位等于1，则调用gralloc_alloc_framebuffer()在系统帧缓冲区中进行分配；否则，则调用gralloc_alloc_buffer()在内存中进行分配。


<a name="anchor5_2"></a>
## 5.2 gralloc_alloc_framebuffer

先看看在系统帧缓冲区中分配图形缓冲区的方法gralloc_alloc_buffer()。

    static int gralloc_alloc_framebuffer(alloc_device_t* dev,
            size_t size, int usage, buffer_handle_t* pHandle)
    {
        private_module_t* m = reinterpret_cast<private_module_t*>(
                dev->common.module);
        pthread_mutex_lock(&m->lock);
        int err = gralloc_alloc_framebuffer_locked(dev, size, usage, pHandle);
        pthread_mutex_unlock(&m->lock);
        return err;
    }

说明：该函数会调用gralloc_alloc_framebuffer_locked()分配图形缓冲区。


<a name="anchor5_3"></a>
## 5.3 gralloc_alloc_framebuffer_locked

    static int gralloc_alloc_framebuffer_locked(alloc_device_t* dev,
            size_t size, int usage, buffer_handle_t* pHandle)
    {
        private_module_t* m = reinterpret_cast<private_module_t*>(
                dev->common.module);

        // 如果系统帧缓冲区还没有初始化，则对系统缓冲区进行初始化
        if (m->framebuffer == NULL) {
            int err = mapFrameBufferLocked(m);
            if (err < 0) {
                return err;
            }
        }

        const uint32_t bufferMask = m->bufferMask;
        const uint32_t numBuffers = m->numBuffers;
        const size_t bufferSize = m->finfo.line_length * m->info.yres;
        // 如果系统帧缓冲区只有一个图形缓冲区大小，则该缓冲区不能分配给应用程序使用，而只能作为系统的主缓冲区；
        // 此时，在内存中分配图形缓冲区。
        if (numBuffers == 1) {
            int newUsage = (usage & ~GRALLOC_USAGE_HW_FB) | GRALLOC_USAGE_HW_2D;
            return gralloc_alloc_buffer(dev, bufferSize, newUsage, pHandle);
        }

        if (bufferMask >= ((1LU<<numBuffers)-1)) {
            return -ENOMEM;
        }

        // create a "fake" handles for it
        intptr_t vaddr = intptr_t(m->framebuffer->base);
        // hnd就是用来该图形缓冲区
        private_handle_t* hnd = new private_handle_t(dup(m->framebuffer->fd), size,
                private_handle_t::PRIV_FLAGS_FRAMEBUFFER);

        // 在系统帧缓冲区中找到空闲的区域
        for (uint32_t i=0 ; i<numBuffers ; i++) {
            if ((bufferMask & (1LU<<i)) == 0) {
                m->bufferMask |= (1LU<<i);
                break;
            }
            vaddr += bufferSize;
        }

        hnd->base = vaddr;
        // 偏移
        hnd->offset = vaddr - intptr_t(m->framebuffer->base);
        *pHandle = hnd;

        return 0;
    }

说明：gralloc_alloc_framebuffer_locked()是在系统帧缓冲区中分配一个空闲的图形缓冲区给应用程序使用。但是如果系统帧缓冲区本身就只有一个图形缓冲区大小，则它不能分配给应用程序使用，而只能作为系统的主缓冲区；此时，就调用gralloc_alloc_buffer()从内存中分配图形缓冲区给应用程序。如果系统帧缓冲区中的有空闲的图形缓冲区，则找到该空闲区域并分配给应用程序。  
此时，创建保存图形缓冲区的private_handle_t变量时，传入的参数是PRIV_FLAGS_FRAMEBUFFER；这个参数的意思表示该图形缓冲区是从系统帧缓冲区分配的。

介绍完了从系统帧缓冲区中分配图形缓冲区，下面看看从内存中分配图形缓冲区的方法gralloc_alloc_buffer()。


<a name="anchor5_4"></a>
## 5.4 gralloc_alloc_buffer

    static int gralloc_alloc_buffer(alloc_device_t* dev,
            size_t size, int usage, buffer_handle_t* pHandle)
    {
        int err = 0;
        int fd = -1;

        // 进行字节对齐
        size = roundUpToPageSize(size);

        // 在"共享内存"中分配一个区域，区域的名称是"gralloc-buffer"，大小是size。
        // 返回该区域的句柄。
        fd = ashmem_create_region("gralloc-buffer", size);
        if (fd < 0) {
            ALOGE("couldn't create ashmem (%s)", strerror(-errno));
            err = -errno;
        }

        if (err == 0) {
            private_handle_t* hnd = new private_handle_t(fd, size, 0);
            gralloc_module_t* module = reinterpret_cast<gralloc_module_t*>(
                    dev->common.module);
            err = mapBuffer(module, hnd);
            if (err == 0) {
                *pHandle = hnd;
            }
        }

        ALOGE_IF(err, "gralloc failed err=%s", strerror(-err));

        return err;
    }

说明：gralloc_alloc_buffer()的作用是从"共享内存"中分配一块区域作为图形缓冲区。该函数会先"共享内存"中划分一块区域，然后将该区域映射到用户空间，并将该区域的句柄等信息保存在private_handle_t中。

<a name="anchor5_5"></a>
## 5.5 mapBuffer

    int mapBuffer(gralloc_module_t const* module,
            private_handle_t* hnd)
    {
        void* vaddr;
        return gralloc_map(module, hnd, &vaddr);
    }

说明：该代码在hardware/libhardware/modules/gralloc/mapper.cpp中。该函数会调用gralloc_map()。


<a name="anchor5_6"></a>
## 5.6 gralloc_map

    static int gralloc_map(gralloc_module_t const* module,
            buffer_handle_t handle,
            void** vaddr)
    {   
        private_handle_t* hnd = (private_handle_t*)handle;
        if (!(hnd->flags & private_handle_t::PRIV_FLAGS_FRAMEBUFFER)) {
            size_t size = hnd->size;
            void* mappedAddress = mmap(0, size,
                    PROT_READ|PROT_WRITE, MAP_SHARED, hnd->fd, 0);
            if (mappedAddress == MAP_FAILED) {
                ALOGE("Could not mmap %s", strerror(errno));
                return -errno;
            }
            hnd->base = intptr_t(mappedAddress) + hnd->offset;
            //ALOGD("gralloc_map() succeeded fd=%d, off=%d, size=%d, vaddr=%p",
            //        hnd->fd, hnd->offset, hnd->size, mappedAddress);
        }
        *vaddr = (void*)hnd->base;
        return 0;
    }

说明：gralloc_map()的作用是将共享内存映射到用户空间，以便应用程序能够使用。



<a name="anchor6"></a>
# 6. 释放图形缓冲区

<a name="anchor6_1"></a>
## 6.1 gralloc_free

    static int gralloc_free(alloc_device_t* dev,
            buffer_handle_t handle)
    {
        if (private_handle_t::validate(handle) < 0)
            return -EINVAL;

        private_handle_t const* hnd = reinterpret_cast<private_handle_t const*>(handle);
        if (hnd->flags & private_handle_t::PRIV_FLAGS_FRAMEBUFFER) {
            // free this buffer
            private_module_t* m = reinterpret_cast<private_module_t*>(
                    dev->common.module);
            const size_t bufferSize = m->finfo.line_length * m->info.yres;
            int index = (hnd->base - m->framebuffer->base) / bufferSize;
            m->bufferMask &= ~(1<<index);
        } else {
            gralloc_module_t* module = reinterpret_cast<gralloc_module_t*>(
                    dev->common.module);
            terminateBuffer(module, const_cast<private_handle_t*>(hnd));
        }

        close(hnd->fd);
        delete hnd;
        return 0;
    }

说明：gralloc_free()的作用是释放图形缓冲区。当hnd->flags标记的PRIV_FLAGS_FRAMEBUFFER位是1时，表示该图形缓冲区是从系统帧缓冲区中分配的；否则，该图形缓冲区是从内存中分配的。  
(01) 当图形缓冲区是从系统帧缓冲区中分配的时候，就系统帧缓冲区找到被分配的这个图形缓冲区的序号index；然后，更改系统帧缓冲区的使用情况变量bufferMask的值。  
(02) 当图形缓冲区是从内存中分配的时候，则调用terminateBuffer()来释放图形缓冲区。  


<a name="anchor6_2"></a>
## 6.2 terminateBuffer

    int terminateBuffer(gralloc_module_t const* module,
            private_handle_t* hnd)
    {       
        if (hnd->base) {
            // this buffer was mapped, unmap it now
            gralloc_unmap(module, hnd);
        }       
                
        return 0;
    }       

说明：该代码在hardware/libhardware/modules/gralloc/mapper.cpp中。它会调用gralloc_unmap()来释放图形缓冲区。


<a name="anchor6_3"></a>
## 6.3 gralloc_unmap

    static int gralloc_unmap(gralloc_module_t const* module,
            buffer_handle_t handle)
    {           
        private_handle_t* hnd = (private_handle_t*)handle;
        if (!(hnd->flags & private_handle_t::PRIV_FLAGS_FRAMEBUFFER)) {
            void* base = (void*)hnd->base;
            size_t size = hnd->size;
            //ALOGD("unmapping from %p, size=%d", base, size);
            if (munmap(base, size) < 0) {
                ALOGE("Could not unmap %s", strerror(errno));
            }
        }
        hnd->base = 0;
        return 0;
    }

说明：gralloc_unmap()的作用是释放图形缓冲区。前面，在从内存中分配图形缓冲区时，是从"共享内存"中获取一块区域，然后通过mmap()将该区域映射到用户空间；现在要释放该区域，则调用munmap()解压映射即可。





<a name="anchor7"></a>
# 7. 注册图形缓冲区

    int gralloc_register_buffer(gralloc_module_t const* module,
            buffer_handle_t handle)
    {           
        if (private_handle_t::validate(handle) < 0)
            return -EINVAL;
            
        private_handle_t* hnd = (private_handle_t*)handle;
        ALOGD_IF(hnd->pid == getpid(),
                "Registering a buffer in the process that created it. "
                "This may cause memory ordering problems.");

        void *vaddr;
        return gralloc_map(module, handle, &vaddr);
    }

说明：该代码在hardware/libhardware/modules/gralloc/mapper.cpp中。
说明：gralloc_map()的作用是将共享内存映射到用户空间，以便应用程序能够使用。



[link_ui_02_datastruct]: /2014/09/11/UI-DataStruct/
