---
title: Gralloc
date: 2025-03-11T09:46:48Z
lastmod: 2025-03-11T09:46:48Z
categories: ["Android"]
tags: ["Camera", "Android"]
---

# Gralloc

# Gralloc

## Linux显示

Linux内核在启动的过程中会创建一个类别和名称分别为"graphics"和"fb0"(0表示从设备号)的设备, 用来描述系统中的第一个帧缓冲区, 即第一个显示屏.
init进程在启动的过程中, 会启动另外一个进程ueventd来管理系统的设备文件. 当ueventd进程启动起来之后, 会通过netlink接口来Linux内核通信, 以便可以获得内核中的硬件设备变化通知. 而当ueventd进程发现内核中创建了一个类型和名称分别为"graphics"和"fb0"的设备的时候, 就会这个设备创建一个/dev/graphics/fb0设备文件.
这样, 用户空间的应用程序就可以通过设备文件/dev/graphics/fb0来访问内核中的帧缓冲区, 即在设备的显示屏中绘制指定的画面.
注: 用户空间的应用程序一般是通过内存映射的方式来访问设备文件/dev/graphics/fb0的.

## HAL模块

Gralloc模块是一个HAL模块, 以下对HAL模块的概念做简要介绍.

Android是基于Linux进行开发的, Linux对硬件的操作基本都在内核中. Android把对硬件的操作分为了HAL和内核驱动两部分, HAL实现在用户空间, 驱动在内核空间. 这是因为Linux内核中的代码是需要开源的, 如果把对硬件的操作放在内核这会损害硬件厂商的利益. 有了HAL层位于用户空间, 硬件厂商就可以将自己的核心算法之类的放在HAL层, 保护了自己的利益, 这样Android系统才能得到更多厂商的支持.

### HAL模块实现的规则

每一种硬件都对应一个HAL模块, 要实现HAL, 必须满足HAL的相关规则.
hardware/libhardware/include/hardware/hardware.h中定义了几个结构体:

```c
typedef struct hw_module_t {
    /** tag must be initialized to HARDWARE_MODULE_TAG */
    uint32_t tag;
    uint16_t module_api_version;

    /** Identifier of module */
    const char *id;

    /** Name of this module */
    const char *name;

    /** Author/owner/implementor of the module */
    const char *author;

    /** Modules methods */
    struct hw_module_methods_t* methods;

    /** module's dso */
    void* dso;
    uint64_t reserved[32-7];
} hw_module_t;
```

结构体`hw_module_t`代表HAL模块, 自己定义的HAL模块必须包含一个自定义struct, 且必须包含`hw_module_t`(第一个变量必须为 `hw_module_t`), 且模块的tag必须指定为`HARDWARE_MODULE_TAG`, 代表这是HAL模块的结构体.

```c
typedef struct hw_module_methods_t {
    /** Open a specific device */
    int (*open)(const struct hw_module_t* module, const char* id,
            struct hw_device_t** device);

} hw_module_methods_t;
```

结构体`hw_module_methods_t`代表模块的操作方法列表, 它内部只有一个函数指针`open`, 用来打开该模块下的设备.

```c
/**
 * Every device data structure must begin with hw_device_t
 * followed by module specific public methods and attributes.
 */
typedef struct hw_device_t {
    /** tag must be initialized to HARDWARE_DEVICE_TAG */
    uint32_t tag;
    uint32_t version;

    /** reference to the module this device belongs to */
    struct hw_module_t* module;
    uint64_t reserved[12];

    /** Close this device */
    int (*close)(struct hw_device_t* device);

} hw_device_t;
```

结构体`hw_device_t`代表该模块下的设备, 自己定义的HAL模块必须包含一个结构体, 且必须“继承”`hw_device_t`(即第一个变量必须为`hw_device_t`)
每个自定义HAL模块还有一个模块名和N个设备名(一个模块可以有多个设备). 在模块定义后需要导出`HAL_MODULE_INFO_SYM`(HMI), 指向此模块. 以下为一个camera hal的HMI:

```c
/******************************************************************************
 * Implementation of camera_module
 ******************************************************************************/
camera_module_t HAL_MODULE_INFO_SYM __attribute__((visibility("default"))) = {
  .common = {
    .tag = HARDWARE_MODULE_TAG,
    .module_api_version = CAMERA_MODULE_API_VERSION_2_5,
    .hal_api_version = HARDWARE_HAL_API_VERSION,
    .id = CAMERA_HARDWARE_MODULE_ID,
    .name = "MediaTek Camera Module",
    .author = "MediaTek",
    .methods = get_module_methods(),
    .dso = NULL,
    .reserved = {0},
  },
  .get_number_of_cameras = get_number_of_cameras,
  .get_camera_info = get_camera_info,
  .set_callbacks = set_callbacks,
  .get_vendor_tag_ops = get_vendor_tag_ops,
  .open_legacy = NULL,
  .set_torch_mode = set_torch_mode,
  .init = init,
  .get_physical_camera_info = get_physical_camera_info,
  .is_stream_combination_supported = is_stream_combination_supported,
  .notify_device_state_change = notify_device_state_change,
  .reserved = {0},
};

```

要想自定义的HAL模块被外部使用, 需要导出符号`HAL_MODULE_INFO_SYM`, 导出的这个模块结构体最重要的其实就是tag, id和methods.

- tag必须定义为`HARDWARE_MODULE_TAG`
- id就是定义的模块名
- methods指向自己实现的(包含`hw_module_methods_t`)的结构体, 其中的`open`用于打开模块下的设备, 指向自己实现的`open`函数.

HAL模块需要实现的部分总结如下:

- 实现`hw_module_t`, `hw_module_methods_t`中的`open()`, `hw_device_t`中的`close()`
- 使用`open()`获取到device的device对应操作
- 导出HMI

HAL模块最后会被编译层一个so库, 放到手机的/system/lib64下.

## Gralloc简介

Android设备的显示屏被抽象为一个帧缓冲区, 而Android系统中的SurfaceFlinger服务就是通过向这个帧缓冲区写入内容来绘制应用程序的用户界面的.
Android系统在HAL层中提供了一个Gralloc模块 (hardware/libhardware/modules/gralloc), 封装了对帧缓冲区的所有访问操作.

用户空间的应用程序在使用帧缓冲区之前, 首先要加载Gralloc模块, 并且获得一个gralloc设备和一个fb设备. 有了gralloc设备之后, 用户空间中的应用程序就可以申请图形缓冲区, 并且将图形缓冲区映射到应用程序的地址空间来, 向里面写入要绘制的画面的内容.
之后, 用户空间中的应用程序就通过fb设备来将已经准备好了的图形缓冲区渲染到帧缓冲区中去, 即将图形缓冲区的内容绘制到显示屏中去.
相应地, 当用户空间中的应用程序不再需要使用一块图形缓冲区的时候, 就可以通过gralloc设备来释放它, 并且将它从地址空间中解除映射.
Gralloc模块负责申请和释放GraphicBuffer的HAL层模块, 由硬件驱动提供实现, 为BufferQueue机制提供了基础. gralloc分配的图形Buffer是进程间共享的, 且根据其Flag支持不同硬件设备的读写.

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/g1-20241008184351-a5aeho6.png)

- 最底层是gralloc HAL模块.
- 向上是libui库, 主要功能是封装对gralloc模块的HAL层的调用. 代码目录是frameworks/native/include/ui和frameworks/native/libs/ui.
- 再向上是libgui库, 主要功能是封装BufferQueue, 向下依赖于于libui. 代码目录是frameworks/native/include/gui和frameworks/native/libs/gui.
- 最上面是使用方, OpenGL ES是BufferQueue的生产方, SurfaceFlinger是BufferQueue的消费方.

## Gralloc模块的实现

每一个HAL模块都有一个ID, 以ID为参数来调用硬件抽象层提供的函数`hw_get_module()`就可以将指定的模块加载到内存来, 并且获得一个`hw_module_t`接口来打开相应的设备.
Gralloc模块的ID定义在hardware/libhardware/include/hardware/gralloc.h文件中, `GRALLOC_HARDWARE_MODULE_ID`实际上为字符串"gralloc".
以下为Gralloc模块的HMI, 对其进行分析以剖析Gralloc模块的具体实现.

```c
struct private_module_t HAL_MODULE_INFO_SYM = {
    .base = {
        .common = {
            .tag = HARDWARE_MODULE_TAG,
            .version_major = 1,
            .version_minor = 0,
            .id = GRALLOC_HARDWARE_MODULE_ID,
            .name = "Graphics Memory Allocator Module",
            .author = "The Android Open Source Project",
            .methods = &gralloc_module_methods
        },
        .registerBuffer = gralloc_register_buffer,
        .unregisterBuffer = gralloc_unregister_buffer,
        .lock = gralloc_lock,
        .unlock = gralloc_unlock,
    },
    .framebuffer = 0,
    .flags = 0,
    .numBuffers = 0,
    .bufferMask = 0,
    .lock = PTHREAD_MUTEX_INITIALIZER,
    .currentBuffer = 0,
};
```

可以看到Gralloc模块的HMI类型为`private_module_t`, 其定义(hardware/libhardware/modules/gralloc/gralloc_priv.h)如下:

```c
struct private_module_t {
    gralloc_module_t base;

    private_handle_t* framebuffer;
    uint32_t flags;
    uint32_t numBuffers;
    uint32_t bufferMask;
    pthread_mutex_t lock;
    buffer_handle_t currentBuffer;
    int pmem_master;
    void* pmem_master_base;

    struct fb_var_screeninfo info;
    struct fb_fix_screeninfo finfo;
    float xdpi;
    float ydpi;
    float fps;
};
```

结构体private_module_t, 主要是用来描述帧缓冲区的属性, 简要描述如下:

- private_handle_t framebuffer: 一个指向系统帧缓冲区的指针
- flags： 标志系统帧缓冲区是否支持双缓冲。如果支持的话，那么它的PAGE_FLIP位就等于1，否则的话，就等于0。
- numBuffers: 表示系统帧缓冲区包含图形缓冲区数量。一个帧缓冲区包含图形缓冲区数量与它的可视分辨率以及虚拟分辨率的大小有关。例如，如果一个帧缓冲区的可视分辨率为800x600，而虚拟分辨率为1600x600，那么这个帧缓冲区就可以包含有两个图形缓冲区。
- bufferMask: 记录系统帧缓冲区中的图形缓冲区的使用情况(按位记录)
- lock: 一个互斥锁，用来保护结构体private_module_t的并行访问。
- buffer_handle_t currentBuffer: 描述当前正在被渲染的图形缓冲区
- fb_var_screeninfo info: 保存设备显示屏的可变属性信息
- fb_fix_screeninfo finfo: 保存设备的只读属性信息
  注: 这info和finfo的值可以通过IO控制命令FBIOGET_VSCREENINFO和FBIOGET_FSCREENINFO来从帧缓冲区驱动模块中获得。
- xdpi, ydpi: 分别用来描述设备显示屏在宽度和高度上的密度，即每英寸有多少个像素点。
- fps: 描述刷新率，即帧率。

其中`gralloc_module_t`(也就是`.base`部分)封装了`hw_module_t`实现了HAL模块的需要. 其定义(hardware/libhardware/include/hardware/gralloc.h)如下:

```c
typedef struct gralloc_module_t {
    struct hw_module_t common;
    int (*registerBuffer)(struct gralloc_module_t const* module,
            buffer_handle_t handle);
    int (*unregisterBuffer)(struct gralloc_module_t const* module,
            buffer_handle_t handle);
    int (*lock)(struct gralloc_module_t const* module,
            buffer_handle_t handle, int usage,
            int l, int t, int w, int h,
            void** vaddr);
    int (*unlock)(struct gralloc_module_t const* module,
            buffer_handle_t handle);
    int (*perform)(struct gralloc_module_t const* module,
            int operation, ... );
    int (*lock_ycbcr)(struct gralloc_module_t const* module,
            buffer_handle_t handle, int usage,
            int l, int t, int w, int h,
            struct android_ycbcr *ycbcr);
    int (*lockAsync)(struct gralloc_module_t const* module,
            buffer_handle_t handle, int usage,
            int l, int t, int w, int h,
            void** vaddr, int fenceFd);
    int (*unlockAsync)(struct gralloc_module_t const* module,
            buffer_handle_t handle, int* fenceFd);
    int (*lockAsync_ycbcr)(struct gralloc_module_t const* module,
            buffer_handle_t handle, int usage,
            int l, int t, int w, int h,
            struct android_ycbcr *ycbcr, int fenceFd);
    int32_t (*getTransportSize)(
            struct gralloc_module_t const* module, buffer_handle_t handle, uint32_t *outNumFds,
            uint32_t *outNumInts);
    int32_t (*validateBufferSize)(
            struct gralloc_module_t const* device, buffer_handle_t handle,
            uint32_t w, uint32_t h, int32_t format, int usage,
            uint32_t stride);
    void* reserved_proc[1];
} gralloc_module_t;
```

其具体实现在hardware/libhardware/modules/gralloc/mapper.cpp

- `registerBuffer`和`unregisterBuffer`: 分别用来注册和注销一个指定的图形缓冲区, 这个指定的图形缓冲区使用一个buffer_handle_t指针来描述. 所谓注册图形缓冲区, 实际上就是将一块图形缓冲区映射到一个进程的地址空间去, 而注销图形缓冲区就是执行相反的操作.
- `lock`和`unlock`: 分别用来锁定和解锁一个指定的图形缓冲区, 这个指定的图形缓冲区同样是使用一个buffer_handle_t指针来描述. 在访问一块图形缓冲区的时候, 需要将该图形缓冲区锁定, 用来避免访问冲突. 在锁定一块图形缓冲区的时候, 可以通过参数l, t, w, h来指定要锁定的图形绘冲区的位置以及大小. 其中, 参数l和t指定的是要访问的图形缓冲区的左上角位置, 而参数w和h指定的是要访问的图形缓冲区的宽度和长度. 锁定之后, 就可以获得由参数l, t, w, h所圈定的一块缓冲区的起始地址, 保存在参数vaddr中. 在访问完成一块图形缓冲区之后, 需要解除这块图形缓冲区的锁定.

更多具体解释见原文注释

在Gralloc模块中, `HAL_MODULE_INFO_SYM`指向的gralloc结构体的成员函数`registerBuffer`, `unregisterBuffer`, `lock`, `unlock`分别被指定为函数`gralloc_register_buffer`, `gralloc_unregister_buffer`, `gralloc_lock`, `gralloc_unlock`.

至此, 简要介绍了Gralloc模块HMI中的内容.

## Gralloc模块的使用

待补充
