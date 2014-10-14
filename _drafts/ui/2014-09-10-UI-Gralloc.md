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


