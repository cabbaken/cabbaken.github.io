---
title: Android相机架构概览
date: 2025-03-11T09:46:48Z
lastmod: 2025-03-11T09:46:48Z
categories: ["Camera"]
tags: ["Camera", "Android"]
---

# 安卓相机架构

Android系统将相机架构分层, 各层的接口与实现分开, 使用接口连接各层, Android相机的架构共被分为5层. 如下图, 包括APP, framework, Service, HAL, driver.

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203164642-20250311094648-1a43amv.png)

## Camera APP

Camera APP是最上层, 不同的APP有不同的界面, 负责与用户直接交互. 通过Camera2 API将需求发送到Camera Framework层, 以此控制相机, 再将接收到的结果反馈给用户.

## Camera Framework

Camera Framework层封装了Camera2 API的实现细节, 并将接口提供给Camera APP层, 供其调用. 收到APP层的请求后通过AIDL实现IPC, 将请求发送给Camera Service层, 并等待其回传结果, 然后将结果发送给Camera APP. 而Camera2作为jar包供Camera APP调用.

Camera2 API的实现: frameworks/base/core/java/android/hardware/camera2/

## Camera Service

Camera Service层会作为一个独立的进程(cameraserver)在系统启动后运行, 封装了AIDL接口供Framework层进行调用. 接收到Framework层的请求后通过HIDL或AIDL接口将请求发送至Camera HAL层, 并在收到结果后将结果回传给Framework层.

Camera Service的相关实现: frameworks/av/services/camera/libcameraservice/

## Camera HAL

Camera HAL层处于Driver与Service之间. Camera HAL内部分为两层, 分别为Camera Provider和Camera HAL Module. 而HAL层由于不同厂商的硬件差异, 所以HAL的实现也有所不同, HAL的实现由厂商提供.

高通的Camera HAL实现: vendor/qcom/proprietary/camx/

MTK的Camera HAL实现: vendor/mediatek/proprietary/hardware/mtkcam3/

### Camera Provider

Camera Provider作为HAL的上层, 是一个独立的进程, 在系统启动的初期被启动, 提供HIDL或AIDL接口供Camer Service调用, 而HAL3标准的实现在HAL Module中. Provider负责接收来自Camera Service的请求, 通过回调调用HAL Module, 然后等待其结果回传并将结果通过HIDL/AIDL接口发送至Camera Service.

高通的Camera Provider实现: vendor/qcom/proprietary/camx/camx-service/provider/

MTK的Camera Provider实现: vendor/mediatek/proprietary/hardware/mtkcam3/main/HAL/entry/hidl/provider/

### Camera HAL Module

Camera HAL Module是Cmaera HAL的一部分, 被编译成动态链接库并被加载至Camera Provider. 与Camera Provider类似, 由于硬件厂商的不同, 实现也不同, 主要分析Qcom和MTK的实现. 其中Qcom使用cmax-chi架构, MTK使用TinyMW架构.

#### Qcom CamX-CHI

高通对Camera HAL的架构为CamX-CHI, 以so库的形式被加载至Camera Provider, 提供HAL3标准的接口供Provider调用, 接收Provider的请求. 其内部实现了HAL3接口, 通过V4L2标准框架控制相机驱动层, 将请求下发至驱动部分, 并等待结果然后回传给Provider.

CamX-CHI分为CamX和CHI两个部分, CamX中基本是架构的核心跳转及处理业务的代码, 一般不需要更改, 相关实现在vendor/qcom/proprietary/camx/中. CHI中一般是手机厂商定制功能的地方, 代码目录在vendor/qcom/proprietary/chi-cdk/. 一个请求会先交给CamX处理, 然后经过CHI进行对请求的重新定制化打包后交给CamX实际执行或与Camera Driver进行交互.

#### MTK HAL

MTK HAL内部实现了HAL3接口, 接收Provider的请求并通过V4L2标准框架控制相机驱动层, 将请求下发至驱动部分, 并等待结果然后回传给Provider.

MTK HAL分为Core Device, PostProc Device, PipelineModel, MTK HAL Interface四部分. Core Device提供Interface来连接Custom Zone, 作为客制化代码区域. 在Custom Zone中集成三方算法, 然后通过控制Core Device和PostProc Device完成相机处理流程. MTK HAL Interface是MTK在HAL层定义的全新的接口, 用于三方算法集成, 更完整的完成客制化代码区与MTK HAL代码的解耦. PostProc Device为Custom Zone提供独立的平台硬件能力或算法, 通过MTK HAL Interface开放给Custom Zone. PipelineModel是MTK Camera HAL pipeline的实际处理的模块.

其相关代码实现: vendor/mediatek/proprietary/hardware/mtkcam3/

## Camera Driver

Camera Driver负责驱动相机. Linux为摄像头制定了V4L2 (Video For Linux 2) 接口, 这为驱动和应用程序提供了一套统一的接口规范, 内核中实现了其基础框架. Camera Driver按照I2C协议与硬件通信, 根据MIPI CSI协议传递数据, 硬件相关驱动代码的编写, 除了编写具体硬件的控制代码外, 主要是将驱动与V4L2框架绑定, 包括V4L2中的数据结构与驱动的数据结构和ioctl的相关函数指针. 用户空间进程可以通过V4L2接口调用相关设备功能, 而不用考虑其实现细节.

V4L2 Core代码实现: kernel/drivers/media/v4l2-core/

Qcom Camera Driver代码实现: vendor/qcom/opensource/ais-kernel/drivers/

MTK Camera Driver代码实现: vendor/mediatek/kernel_modules/mtkcam/

## Camera Hardware

相机硬件处在整个相机体系的最底层, 是相机系统的物理实现部分. 包括镜头、感光器、ISP三个主要模块, 还有对焦马达、闪光灯、滤光片、光圈等辅助模块. 镜头的作用是汇聚光线, 利用光的折射性把射入的光线汇聚到感光器上. 感光器的作用是负责光电转换, 通过内部感光元件将接收到的光信号转换为电子信号进而通过数电转换模块转为数字信号, 并最后传给ISP. ISP负责对数字图像进行一些算法处理, 如白平衡、降噪、去马赛克等.

## 架构总结

通过上述对相机架构各层的介绍, 我们可以发现Android系统中相机架构中层与层之间由不同的接口沟通, 其中大致可以理解为上层通过标准接口将请求发送给下层, 下层根据自身对标准接口的实现处理请求后发送给更下层, 最后将下层收到的结果回传给上层. 从整体上看, 分层利用标准接口让整个框架结构分明, 减少耦合, 使整个框架更利于维护.

总体符合下图

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/image-20250311094648-f3esd4j.png)

# Camera APP

Camera APP通常是一个java程序, 为了满足各种场景, 会加入很多业务处理逻辑, 而其核心是调用谷歌制定的Camera2 API对相机进行控制. Camera2 API引入了Session和Reques,

## Camera2 API

谷歌提出Camera Api v2接口, 引入了Session以及Request概念, 将控制逻辑统一成一个视图, 因此在使用上更加复杂, 同时也支持了更多特性, 比如逐帧控制曝光、感光度以及支持Raw格式的输出等. 并且由于对控制逻辑的高度抽象化, 使得该接口具有很高的灵活性. 总得来说, 该接口极大地提高了对于相机框架的控制能力, 同时也进一步大幅度提升了其整体性能. 谷歌将其具体实现放入了Camera Framework中来完成, Framework内部负责解析来自App的请求, 并且通过AIDL跨进程接口下发到Camera Service中进行处理, 并且等待结果的回传.

Camera2 API主要包含以下类:

- CameraManager: 用于检测(getCameraIdList())并打开(openCamera(String, CameraDevice.StateCallback, Handler))可见的相机, 也可以用于获取当前Camera设备支持的属性信息(使用getCameraCharacteristics(String)返回一个CameraCharacteristics)

  - CameraDevice.StateCallback: 一个由开发者实现的抽象类, 当openCamera()成功打开相机时, 执行CameraDevice.StateCallback.onOpened(CameraDevice)
- CameraDevice: 通过CameraDevice.StateCallback.onOpened获得CameraDevice, 表示一个相机设备, 使用此类的方法可以控制相机并与其交互

  - createCaptureSession(SessionConfiguration): 用于获取一个CaptureSession
  - createCaptureRequest(int): 用于获取一个CaptureRequest.Builder, 创建一个Request
- CameraCaptureSession: CaptureSession可被用于拍摄照片, 它需要配置相机设备的内部设置并分配内存缓冲区(Surface)以将图像发送到目标位置

  - capture(CaptureRequest, CameraCaptureSession.CaptureCallback, Handler): 用于进行一次照片拍摄. 其中CaptureRequest一般来自于CaptureRequest.Builder.build(), 表示此次拍摄的设置. CameraCaptureSession.CaptureCallback与前文提到的CameraDevice.StateCallback类似, 在图片拍摄完成时调用CameraCaptureSession.CaptureCallback.onCaptureCompleted(CameraCaptureSession, CaptureRequest, TotalCaptureResult). capture的全部输出数据存储在result中, 其中包含了最终拍摄设置和输出数据
- CaptureRequest: 由CaptureRequest.Builder创建, 包含了拍摄的硬件设置, 处理流程和算法, 也包含数据要发送到的Surface列表. Surface是图像的数据缓冲区
- CaptureResult: 包含了部分拍摄设置和输出数据
- TotalCaptureResult: 包含了最终拍摄设置和输出数据

Camera APP是Android相机架构的最上层, 为了保证整个框架的稳定性以及高效性, 谷歌重新设计了Camera2 API, 将控制逻辑高度抽象成了一个控制视图, 所以可以逐帧控制硬件参数, 进而实现一系列强大的功能. Api v2接口的实现在Camera Framework中完成, 内部并没有采用十分复杂的控制逻辑, 保证了该层的稳定以及高效.

# Camera Framework

基于接口与实现相分离的基本设计原则, 谷歌通过Camera2 API接口的定义, 搭建起了APP与相机系统的桥梁, 而具体实现便是由Camera Framework来负责完成的. Camera Framework通过AIDL机制让Camera2 API得以直接与Camera Service通信, 简化了框架. 以下对Camera2 API中的主要实现进行介绍:

## CameraManager.openCamera()

当用户打开相机应用时, 会去调用该方法打开一个相机设备, openCamera()的内部实现如下:

```Plain
openCamera()->
    openCameraForUid()
    openCameraDeviceUserAsync()->
        getCameraCharacteristics()
        getPhysicalIdToCharsMap()
        new CameraDeviceImpl()
        ICameraService.connectDevice()
        CameraDeviceImpl.setRemoteDevice()
```

openCameraDeviceUserAsync()主要工作为:

1. 创建一个CameraDevice实例 (new CameraDeviceImpl()) , 其中APP定义的CameraDevice.StateCallback将会被保存在此CameraDeviceImpl中.
2. 调用Camera Service的AIDL接口, 与摄像头连接 (ICameraService.connectDevice()) .

连接完成后, 来自APP的CameraDevice.StateCallback.onOpen()会被调用, 把CameraDevice对象返回给APP.

## CameraDevice.createCaptureSession()

打开相机后需要创建一个CaptureSession才可以开始发送图像传输的请求, 具体过程如下:

```Plain
createCaptureSession()->
    createCaptureSessionInternal()->
        configureStreamsChecked()->
            ICameraDeviceUserWrapper.beginConfigure()
            ICameraDeviceUserWrapper.deleteStream()
            ICameraDeviceUserWrapper.createStream()
            ICameraDeviceUserWrapper.endConfigure()
        ICameraDeviceUserWrapper.getInputSurface()
        new CameraCaptureSessionImpl()
```

createCaptureSessionInternal()主要工作为:

1. 使用configureStreamsChecked()对流进行配置
2. 实例化一个CameraCaptureSessionImpl, 其中APP定义的CameraCaptureSession.StateCallback将会被保存在此CameraCaptureSessionImpl中.

## CameraCaptureSession.capture()

有了CameraCaptureSession就可以使用capture()进行拍照, capture()的具体过程如下:

```Plain
capture()->
    CameraDevice.capture()->
        CameraDevice.submitCaptureRequest()->
            ICameraDeviceUserWrapper.submitRequestList()->
                ICameraDeviceUser.submitRequestList()
```

capture()最终调用CameraDevice.submitCaptureRequest(), 其通过AIDL接口调用Camera Service层的ICameraDeviceUser.submitRequestList().

Camera Framework负责处理来自Camera APP对于Camera2 API的调用, 并根据APP所调用的接口使用AIDL接口对Camera Service层进行调用. 其中主要的AIDL接口定义于如下两个文件:

frameworks/av/camera/aidl/android/hardware/ICameraService.aidl frameworks/av/camera/aidl/android/hardware/camera2/ICameraDeviceUser.aidl

其中ICameraService表示Camera Service, 而ICameraDeviceUser被ICameraDeviceUserWrapper封装, 也用于调用Camera Service层.

# Camera Service

Camera Service是一个独立的进程, 作为服务端处理来自Camera Framework通过AIDL发送的请求, 并将对应请求发送给Camera Provider, 整个流程涉及两个跨进程操作, 都是进程间通信, 前者通过AIDL实现, 后者为AIDL/HIDL实现. 我们将首先介绍这两种机制.

## AIDL

AIDL (Android Interface Definition Language) 定义了让客户端和服务端为了能够进行进程间通讯 (IPC) 而能够达成一致的接口. 在Android中, 使用Binder进行IPC, 需要将对象分解为操作系统可以理解的原始类型, 再进行数据传输. 编写这种编组的代码很繁琐, 所以Android用AIDL处理.

AIDL利用binder驱动完成调用. 进行接口调用时, 函数标识 (method identifier) 和所有对象 (参数) 被打包到缓冲区中, 并复制到远程进程中, 在远程进程中, 有binder线程等待读取数据. binder线程读取数据后, 会在本进程找到对应对象存储数据并调用本地接口 (client调用的实际执行的函数) .

## HIDL

HIDL (HAL interface definition language) 描述了HAL层与调用HAL层的接口, HIDL使HAL层能通过binder来进行IPC.

待补充

## Camera Service的启动

在系统启动时, 运行init进程, init会识别并读取frameworks/av/camera/cameraserver/cameraserver.rc, 并启动cameraserver, 而cameraserver进程就是cameraservice.

Camera server: frameworks/av/camera/cameraserver/main_cameraserver.cpp

```Plain
CameraService::instantiate()->
    BinderService::publish()
```

最终会创建一个Camera Service实例 (在frameworks/av/services/camera/libcameraservice/CameraService.cpp中定义) . 实例创建后, 首次被强指针引用会调用`CameraService::onFirstRef()` (`CameraService`继承自`BinderService`) . 对Camera Service启动的详细介绍会在下文详细介绍.

## Camera Service主程序

- CameraService: 相机服务实例, 手机启动时会生成该类的对象. 应用对ICameraService的函数调用, 会访问到CameraService.
- CameraDeviceClient: 当应用调用`openCamera()`打开指定相机设备时, 会生成一个对应的CameraDeviceClient. 应用通过对ICameraDeviceUser的函数调用, 会访问到CameraDeviceClient.
- CameraProviderManager: CameraService对象生成时创建的对象, 用于同HAL交互.
- Camera3Device: 应用调用`openCamera()`时生成该类的对象, 即具体的Camera设备.

Camera3Device相关类图如下:

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203164702-20250311094648-by5x9w9.png)

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203164710-20250311094648-v02lau3.png)

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203164717-20250311094648-pcayn3c.png)
具体说明待补充

# Camera HAL

Camera HAL是硬件抽象层, 所以是硬件依赖的, 不同的CPU需要使用不同的HAL. 为了让Camera Service能够调用HAL层, 我们需要让HAL提供一致的API, 也就是HAL3接口.

而只有HAL3接口还不够, 还需要有API的具体实现, 包括硬件厂商自己的实现, 如高通和MTK.

## Camera HAL3

HAL3定义了自己的一套通用标准接口 (hardware/libhardware/include/hardware/camera3.h, hardware/libhardware/include/hardware/camera_common.h) , 平台厂商要根据以下规则实现自己的module.

- 每个硬件都通过`hw_module_t`来描述, 被称为HMI.
- 每一个HMI都要实现自身的`open()`, 用于打开设备, 并返回对应的操作接口.
- 使用`hw_device_t`来描述`open()`返回的操作接口.

以下, 我们先来介绍几个数据结构.

### hw_module_t

其中`hw_module_t` (hardware/libhardware/include/hardware/hardware.h) :

```C++
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

`hw_module_t`中的主要内容是`hw_module_methods_t`:

```C++
typedef struct hw_module_methods_t {
    /** Open a specific device */
    int (*open)(const struct hw_module_t* module, const char* id,
            struct hw_device_t** device);

} hw_module_methods_t;
```

`hw_module_methods_t`的作用为打开设备, 也就是上文提到的`open()`.

接下来我们看看`hw_device_t`:

```C++
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

`hw_module_t`代表模块, 通过其`open()`用来打开一个设备, 而该设备是用`hw_device_t`来表示, 其中除了用来关闭设备的`close()`外, 并无其它方法, 由此可见谷歌定义的HAL接口, 并不能满足绝大部分HAL模块的需要, 所以谷歌想出了一个比较好的解决方式, 那便是将这两个基本结构嵌入到更大的结构体内部, 同时在更大的结构内部定义了各自模块特有的方法, 用于实现模块的功能, 保持了HAL的统一规范的同时扩展了模块的功能.

这些结构体用于描述最基础的硬件操作. 而相机相关的Camera HAL3接口定义了另外的结构体, 包括`camera_module_t`、`camera3_device_t`、`camera3_stream_configuration`、`camera3_stream`、`camera3_stream_buffer`等.

### camera_module_t

`camera_module_t`的数据结构如下:

```C++
typedef struct camera_module {
    hw_module_t common;
    int (*get_number_of_cameras)(void);
    int (*get_camera_info)(int camera_id, struct camera_info *info);
    int (*set_callbacks)(const camera_module_callbacks_t *callbacks);
    void (*get_vendor_tag_ops)(vendor_tag_ops_t* ops);
    int (*open_legacy)(const struct hw_module_t* module, const char* id,
            uint32_t HALVersion, struct hw_device_t** device);
    int (*set_torch_mode)(const char* camera_id, bool enabled);
    int (*init)();
    int (*get_physical_camera_info)(int physical_camera_id,
            camera_metadata_t **static_metadata);
    int (*is_stream_combination_supported)(int camera_id,
            const camera_stream_combination_t *streams);
    void (*notify_device_state_change)(uint64_t deviceState);
    int (*get_camera_device_version)(int camera_id, uint32_t *version);
    void* reserved[1];
} camera_module_t;
```

由于以上结构体中的注释篇幅较长, 故在此对其简要说明:

- `hw_module_t common`: 相机模块的常用方法. 这必须为`camera_module` 的第一个成员, 使用此结构体时会利用`hw_module_t common`获取`camera_module`指针.
- `int (*get_number_of_cameras)(void)`: 返回能够通过`camera_module`访问的相机数量. 相机设备的编号为0～N-1, `common.methods->open()`使用的id就是对应的数字转化为字符串得来的.
- `int (*get_camera_info)(int camera_id, struct camera_info *info)`: 返回相机的静态信息 (不会改变的信息)
- `int (*set_callbacks)(const camera_module_callbacks_t *callbacks)`: 向HAL提供回调函数指针, 通知framework层异步相机事件, framework层会在初始化相机 HAL module加载、首次调用 `get_number_of_cameras()`方法后以及对模块进行任何其他调用之前调用此函数.
- `void (*get_vendor_tag_ops)(vendor_tag_ops_t* ops)`: 获取查询vendor extension metadata tag (包含用于枚举一组不可变的vendor定义的相机元数据标签以及查询有关其结构/类型的静态信息的基本函数. 用途是验证相机 HAL 返回的元数据的结构, 并允许vendor定义的元数据标签在面向应用程序的相机 API 中可见. ) 的方法. HAL 应填写所有vendor tag方法, 如果未定义则`ops`不变. (`vendor_tag_ops`在system/media/camera/include/system/vendor_tags.h中定义)
- `int (*open_legacy)(const struct hw_module_t* module, const char* id, uint32_t HALVersion, struct hw_device_t** device)`: 如果此相机 HAL 模块支持多个HAL的API版本, 打开特定的旧版的HAL模块. (一般不需要)
- `int (*set_torch_mode)(const char* camera_id, bool enabled)`: 打开或关闭与`camera_id`关联的闪光灯单元的torch mode, 若成功, HAL需要通知framework层torch状态被改变 (通过`camera_module_callbacks.torch_mode_status_change()`) .
- `int (*init)()`: 此函数被camera service在调用其他函数前调用, 将会加载camera HAL库.
- `int (*get_physical_camera_info)(int physical_camera_id, camera_metadata_t **static_metadata)`: 返回一个物理摄像头的静态元数据作为逻辑摄像头的一部分, 此函数只有在物理摄像头的id没有公开时被调用, 也就是说camera_id会大于等于`get_number_of_cameras()`的返回值.
- `int (*is_stream_combination_supported)(int camera_id, const camera_stream_combination_t *streams)`: 查看设备对特定相机流组合的支持.
- `void (*notify_device_state_change)(uint64_t deviceState)`: 通知相机模块整个设备的状态已经以某种方式发生了变化.
- `int (*get_camera_device_version)(int camera_id, uint32_t *version)`: 返回给定相机设备的设备版本. 对于相机设备, 此值可能不会改变. 此处返回的版本必须与`get_camera_info()`中的版本相同.

`camera_module_t`包含了`hw_module_t`, 用于表示相机模块, 定义了拓展方法.

### camera3_device_t

`camera3_device_t`的定义如下:

```C++
typedef struct camera3_device {
    hw_device_t common;
    camera3_device_ops_t *ops;
    void *priv;
} camera3_device_t;
```

`camera3_device_t`包含了`hw_device_t`, 主要用来表示Camera设备. 其中, `camera3_device_ops_t *ops`中存放一系列对`camera3_device`的操作, 用来实现正常获取图像数据以及控制Camera的功能, 是核心接口, 将会在下文重点介绍.

### camera3_stream_configuration_t

`camera3_stream_configuration_t`是一个包含了流定义的结构体, 被用与`configure_streams()`. 其定义了所有的输出流和重处理输入流, 其中包括了:

```C++
typedef struct camera3_stream_configuration {
    uint32_t num_streams;
    camera3_stream_t **streams;
    uint32_t operation_mode;
} camera3_stream_configuration_t;
```

- `uint32_t num_streams`: 表示framework层需要的流的总数量, 包含input和output流. (至少有一条可以output的流)
- `camera3_stream_t **streams`: 相机流指针数组, 定义相机 HAL 设备的输入/输出配置. 单个`camera3_stream_configuration`中最多可定义一个支持输入的流 (INPUT 或 BIDIRECTIONAL) 和至少一个支持输出的流 (OUTPUT 或 BIDIRECTIONAL) .
- `uint32_t operation_mode`: 是`camera3_stream_configuration_mode_t`中的值, 表示这个配置中流的运行模式.

### camera3_stream_t

`camera3_stream_t`用于描述单个摄像头输入或输出流的属性. Framework通过流的buffer format (缓冲区中数据的组织和编码方式, 决定了图像数据如何在内存中表示) 和buffer resolution (buffer resolution通常指的是在图像处理或者视频流处理中, 用于存储图像或视频帧的缓冲区的尺寸, 决定了能够处理的最大图像尺寸和质量) , gralloc usage flag (内存分配和访问权限指示, 告诉gralloc如何分配内存 (比如内存是用于CPU访问、GPU访问还是同时用于两者) 指定了内存区域可以被哪些组件访问 (如CPU、GPU、Camera等) 以及访问的方式 (读、写、渲染等) ) 和in-flight buffer count (多个request在HAL中等待处理, 这些request携带的buffer被称为in-flight buffer) 定义流.

stream的所有者是framework, 但是指向`camera3_stream_t`的指针通过`configure_streams()`被传入HAL, 这个指针在下一个不包含`camera3_stream`的`configure_streams()`或`close()`前都有效.

一旦将`camera3_stream`传递到`configure_streams()`, 所有`camera3_stream`中由framework管理的成员都不可变. HAL只能在`configure_streams()`调用期间更改HAL控制的参数 (私有指针的内容除外) .

`camera3_stream_t`中包含:

```C++
typedef struct camera3_stream {
    int stream_type;
    uint32_t width;
    uint32_t height;
    int format;
    uint32_t usage;
    uint32_t max_buffers;
    void *priv;
    android_dataspace_t data_space;
    int rotation;
    const char* physical_camera_id;
    void *reserved[6];
} camera3_stream_t;
```

- `int stream_type`: 表示此流的种类 (在`camera3_stream_type_t`中定义)
- `uint32_t width`: 表示buffer的宽度 (像素为单位)
- `uint32_t height`: 表示buffer的高度 (像素为单位)
- `int format`: 表示此流中buffer的像素格式. 格式来自system/core/include/system/graphics.h中的HAL_PIXEL_FORMAT_*列表的值, 或者是来自设备特定头文件的值. 如果定义了HAL_PIXEL_FORMAT_IMPLEMENTATION_DEFINED, 则平台的gralloc模块将根据相机设备和流的另一端提供的使用标志来选择一个格式.

以上成员由framework设置.

- `uint32_t usage`: 表示此流中HAL需要的gralloc usage flag (在gralloc.h或是设备特定头文件中定义的GRALLOC_USAGE_*) . 对于输出流, 这些是HAL的生产者usage flag；对于输入流, 这些是HAL的消费者usage flag. 来自生产者和消费者的使用标志将被组合在一起, 然后传递给gralloc HAL模块, 为每个流分配gralloc buffer.
- `uint32_t max_buffers`: HAL可能需要同时出队的缓冲区的最大数量. HAL在此流中正在运行的缓冲区数量不得超过此值.
- `void *priv`: 指向HAL对此流的私有信息, 不会被framework查看.
- `android_dataspace_t data_space`: 描述buffer中的内容 (用途与含义) , 包括颜色空间等信息. 具体可见system/core/include/system/graphics.h
- `int rotation`: 流所需要旋转的角度 (在`camera3_stream_rotation_t`中定义) , 与流宽度和高度一起由HAL检查, 相机HAL忽略输入流的旋转字段.
- `const char* physical_camera_id`: 表示此流对应的相机的id.

### camera3_stream_buffer_t

`camera3_stream_buffer_t`来自camera3流的单个buffer. buffer未指定其用于输入还是输出, 取决于其父流类型以及buffer如何传递到HAL.

`camera3_stream_buffer_t`中包含:

```C++
typedef struct camera3_stream_buffer {
    camera3_stream_t *stream;
    buffer_handle_t *buffer;
    int status;
    int acquire_fence;
    int release_fence;
} camera3_stream_buffer_t;
```

- `camera3_stream_t *stream`: 一个指向此buffer对应的stream的指针.
- `buffer_handle_t *buffer`: 一个指向此buffer的native指针, 用于访问实际的图像数据.
- `int status`: buffer此刻的状态 (在camera3_buffer_status_t中定义) . 如果HAL无法填充缓冲区, 它必须在调用 process_capture_result() 时将状态设置为CAMERA3_BUFFER_STATUS_ERROR.
- `int acquire_fence`: 用于同步, HAL必须等待此fence的文件描述符变为可用, 然后才能尝试读取或写入buffer. 如果不需要等待, framework会将其设置为-1.
- `int release_fence`: 用于同步, HAL在返回缓冲区给framework时必须设置这个栅栏, 或者写-1表示不需要等待. 此后, HAL不能再尝试访问buffer中的内容, 因为buffer的所有权已经完全被转移给了framework.

### camera3_device_ops_t

`camera3_device_ops_t`是HAL3的核心接口, 其定义如下:

```C++
typedef struct camera3_device_ops {
    int (*initialize)(const struct camera3_device *,
            const camera3_callback_ops_t *callback_ops);
    int (*configure_streams)(const struct camera3_device *,
            camera3_stream_configuration_t *stream_list);
    int (*register_stream_buffers)(const struct camera3_device *,
            const camera3_stream_buffer_set_t *buffer_set);
    const camera_metadata_t* (*construct_default_request_settings)(
            const struct camera3_device *,
            int type);
    int (*process_capture_request)(const struct camera3_device *,
            camera3_capture_request_t *request);
    void (*get_metadata_vendor_tag_ops)(const struct camera3_device*,
            vendor_tag_query_ops_t* ops);
    void (*dump)(const struct camera3_device *, int fd);
    int (*flush)(const struct camera3_device *);
    void (*signal_stream_flush)(const struct camera3_device*,
            uint32_t num_streams,
            const camera3_stream_t* const* streams);
    int (*is_reconfiguration_required)(const struct camera3_device*,
            const camera_metadata_t* old_session_params,
            const camera_metadata_t* new_session_params);
    void *reserved[6];
} camera3_device_ops_t;
```

- `int (*initialize)(const struct camera3_device *, const camera3_callback_ops_t *callback_ops)`: 一次性初始化, 将框架回调函数指针传递给HAL. 在成功调用`open()`后, 在对 `camera3_device_ops`结构调用任何其他函数之前, 将调用一次. 该函数是非阻塞的, 需要在5ms内返回, 最长不能超过10ms.
- `int (*configure_streams)(const struct camera3_device *, camera3_stream_configuration_t *stream_list)`:
- `int (*register_stream_buffers)(const struct camera3_device *, const camera3_stream_buffer_set_t *buffer_set)`:
- `const camera_metadata_t* (*construct_default_request_settings)(const struct camera3_device *, int type)`:
- `int (*process_capture_request)(const struct camera3_device *, camera3_capture_request_t *request)`: 向HAL发送capture请求. HAL不应从此调用返回, 直到它准备好接受下一个要处理的请求. Framework一次只会对`process_capture_request()`进行一次调用, 并且所有调用都将来自同一线程. 一旦有新请求及其相关buffer可用, 就会对`process_capture_request()`进行下一次调用. 在正常的预览场景中, framework将不停调用该函数. 对请求的处理是异步的, 捕获结果由HAL通过`process_capture_result()`调用返回. 此调用要求元数据可用, output buffer可能要等待同步. 多个请求同时进行时, 保持完整的输出帧速率. framework保留请求的所有权, 它只保证在此调用期间有效, HAL必须复制需要的信息. HAL负责等待并关闭buffer的栅栏并将缓冲区句柄返回给framework. 如果input_buffer不为NULL, 则HAL必须将input buffer的释放同步栅栏的文件描述符写入input_buffer->release_fence. 如果HAL为input buffer释放同步栅栏返回-1, 则框架可以立即重新使用input buffer, 否则framework将等待同步栅栏, 然后再重新填充使用input buffer.
- `void (*get_metadata_vendor_tag_ops)(const struct camera3_device*, vendor_tag_query_ops_t* ops)`:
- `void (*dump)(const struct camera3_device *, int fd)`:
- `int (*flush)(const struct camera3_device *)`:
- `void (*signal_stream_flush)(const struct camera3_device*, uint32_t num_streams, const camera3_stream_t* const* streams)`:
- `int (*is_reconfiguration_required)(const struct camera3_device*, const camera_metadata_t* old_session_params, const camera_metadata_t* new_session_params)`:

待补充

太多了建议自己看文档

### camera3_callback_ops_t

`camera3_callback_ops_t`包含供HAL调用到framework中的回调方法. 这些方法用于返回已完成或失败的capture的元数据和buffer, 并通知framework异步事件 (例如错误) . framework不会从这些回调中回调到 HAL, 并且这些调用不会长时间阻塞.

```C++
typedef struct camera3_callback_ops {
    void (*process_capture_result)(const struct camera3_callback_ops *,
            const camera3_capture_result_t *result);
    void (*notify)(const struct camera3_callback_ops *,
            const camera3_notify_msg_t *msg);
    camera3_buffer_request_status_t (*request_stream_buffers)(
            const struct camera3_callback_ops *,
            uint32_t num_buffer_reqs,
            const camera3_buffer_request_t *buffer_reqs,
            /*out*/uint32_t *num_returned_buf_reqs,
            /*out*/camera3_stream_buffer_ret_t *returned_buf_reqs);
    void (*return_stream_buffers)(
            const struct camera3_callback_ops *,
            uint32_t num_buffers,
            const camera3_stream_buffer_t* const* buffers);
} camera3_callback_ops_t;
```

具体说明待补充

### HAL3总结

上文中提到的HAL3相关的数据结构之间的关系不够直观, 对其进行整理如下:

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203164733-20250311094648-1linxxf.png)
(箭头含义: A->B表示A中有指向B的指针或A包含B)

可以看到, HAL3中的这些数据结构基本可以分为两部分, 通过`camera3_device_ops_t`进行连接.

## Camera Provider

(MTK平台叫做camerahalserver)

谷歌为了将系统框架和平台厂商的自定义部分相分离, 将平台厂商的实现部分放入vendor分区中进行管理, 进而与system分区保持隔离, 这样便可以在相互独立的空间中进行各自的迭代升级, 而互不干扰.

而在相机框架体系中, 将Camera HAL Module从Camera Service中解耦出来, 放入独立进程Camera Provider中进行管理, 而为了更好的进行跨进程访问, 谷歌针对Provider提出了HIDL机制用于Camera Service对于Camera Provider的访问, 而HIDL接口的实现是在Camera Provider中实现, 针对Camera HAL Module的控制又是通过谷歌制定的Camera HAL3接口来完成. 由此看来, Camera Provider的职责也比较简单, 通过HIDL机制保持与Camera Service的通信, 通过HAL3接口控制着Camera HAL Module.

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203164745-20250311094648-8wr8ysa.png)
上图中, 实线箭头是调用关系. 左边是cameraserver进程中的动作, 右边则是cameraprovider进程中的动作, 它们之间通过ICameraProvider联系在了一起.

Cameraserver一侧, CameraService类依旧是主体. 它通过CameraProviderManager来管理对CameraProvider进行操作. 此处初始化的最终目的是连接上CameraProvider. 而CameraProvider一侧, 最终主体是CameraProvider类. 初始化最终目的是得到一个mModule, 通过它可以直接与HAL接口定义层进行交互.

与CameraService的启动相同, CameraProvider的启动也由init进程读取rc文件开始.

```Plain
android::ProcessState::initWithDriver()
registerPassthroughServiceImplementation()->
     getRawServiceInternal()
```

CameraProvider的启动很简单, 主要动作是 (通过system/libhidl/transport/LegacySupport.cpp中定义的`registerPassthroughServiceImplementation()`) 注册该CameraProvider, 以便CameraServer启动时能找到它. 这里的PassThrough服务是指将请求转发给另一个服务, 而不进行任何处理的服务. 需要注意的是, 此时CameraProvider还未实例化.

上文中提到, CameraService在实例化后会调用`CameraService::onFirstRef()`. 其中出现的HidlCameraService继承于ICameraService (在frameworks/av/services/camera/libcameraservice/hidl/HidlCameraService.h中定义) .

```Plain
CameraService::onFirstRef()->
    CameraService::enumerateProviders()->
        new CameraProviderManager()
        mCameraProviderManager::initialize(this)->
            CameraProviderManager::addProviderLocked()->
                getServiceInternal()->
                    (IServiceManager)PassthroughServiceManager::get()->
                        HIDL_FETCH_ICameraProvider()
    HidlCameraService::getInstance(this)
    ICameraService::registerAsService()->    //这里是注册cameraserver而不是camera provider
        registerAsServiceInternal()
```

在CameraService初始化时, 会调用`CameraService::enumerateProviders()`, 通过`CameraProviderManager`连接CameraProvider. 其中, `CameraProviderManager`在frameworks/av/services/camera/libcameraservice/common/CameraProviderManager.h中定义. 在`PassthroughServiceManager::get()` (定义在system/libhidl/transport/ServiceManagement.cpp) 中, CameraProvider与CameraService连接起来. 而`get()`中加载了动态链接库, 调用了`HIDL_FETCH_ICameraProvider()` (在hardware/interfaces/camera/provider/2.4/default/CameraProvider_2_4.cpp中定义) . 其中调用`getProviderImpl()`创建CameraProvider实例并初始化.

下图是Camera Service视角的初始化时序图:

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203164758-20250311094648-50nrlzn.png)
Camera Provider在Qcom和MTK上的实现不同.

MTK平台中Camera Provider实例化时调用了:

```Plain
new CameraProvider()
provider->initialize()->
    setupVendorTags()
    LegacyCameraModule::getDevicesNameList()->
        CameraDeviceManagerBase::getDeviceNameList()
```

MTK平台通过加载动态库调用`HIDL_FETCH_ICameraProvider()`创建实例.

`camera_module_t`类型的变量`HAL_MODULE_INFO_SYM`在vendor/mediatek/proprietary/hardware/mtkcam3/main/mtkHAL/hidl/depend/entry.cpp中定义.

Qcom平台中Camera Provider实例化时调用了:

```Plain
CameraProvider::Create()->
    provider->initialize()->
        hw_get_module()
        mModule->init()
        setUpVendorTags()
        mModule->setCallbacks()
```

下图是Camera Provider视角的初始化时序图 (随着代码更新, 后半部分不太准确, 看起来前半段像MTK, 后半段像高通) :

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203164808-20250311094648-bqectfj.png)
![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203164814-20250311094648-64smw6b.png)
上图描述了CameraService与CameraProvider进行沟通的类的关系. 黄色的部分是HAL层接口类, 而绿色部分为CameraService用于调用HAL的类.

下图对几个重要流程进行描述.

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203164820-20250311094648-rnqemtc.png)
当CameraService和CameraProvider初始化完成了之后, CameraProvider进程 (在MTK平台上叫做CameraHALServer) 便一直便存在于系统中, 等待来自Camera Service的调用.

Camera provider interface: hardware/interfaces/camera/provider/

Camera provider(Qcom): vendor/qcom/proprietary/camx/camx-service/provider/

Camera provider(MTK): vendor/mediatek/proprietary/hardware/mtkcam3/main/mtkHAL/custom/provider/

vendor/mediatek/proprietary/hardware/mtkcam3/main/mtkHAL/hidl/provider/

## MIVI

MiVi: vendor/xiaomi/proprietary/mivifwk

### 算法上移1.0

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203164832-20250311094648-jhwuise.png)
算法上移的根本逻辑是通过预览camera将拍照的request下发下去后, 通过相应的回调获取拍摄的图像以及对应的meta信息, 然后将这些信息打包通过并行化服务, JNI送给算法框架. 算法框架内部是异步处理, 此时预览camera可以正常close, 相机也可以退出后台. 当算法框架算法处理完成, 会通过回调接口将数据返回给App, App再进行水印滤镜处理, 然后通过另一个虚拟camera做Jpeg编码. 生成的Jpeg图像再通过ImageSaver进行存储. 这就是整个拍摄图片的处理流程.

好处:

- 降低shot_2_shot的时间
- 通过通用算法集成框架可以复用APP控制逻辑, 节省新项目适配成本
- 通过算法框架管理算法, 降低算法集成与平台的耦合

### MIVI1.0

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203164840-20250311094648-jl25alh.png)
与算法上移1.0相比, 最大的变化是将算法框架运行进程从APP层下移到HAL层, 同时添加了小米图像后处理服务. APP与算法框架通过后处理服务的客户端和服务端交互. MIVI1.0提供了高通硬件算法管理模块给算法框架使用, 让其拥有raw2yuv的能力. 无法应用到MTK平台.

### MIVI2.0

MIVI2.0的架构如下 (vendor/xiaomi/proprietary/mivifwk)

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203164849-20250311094648-gjtqxxr.png)
重构了小米相机HAL, 目前mihal运行在camera provider进程中, 位于legacy camera hal implementation下, soc hal之上. mihal与mialgoengine一同构成了小米自研的相机中间层服务. mihal承载了miui相机app的下沉业务, 所有的拍照策略与后处理都由mihal主导控制与任务调度.

Mihal 核心组件如下:

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203164858-20250311094648-sgpt67w.png)
mihal可以根据上图划分为3大部分:

- CameraHALImpl: 基于HAL3标准封装mi camera hal impl, 对hal3标准中的camera_module和camera_device进行扩展, 由CameraManager管理.()
- Mivi Foundation: MIVI的基础组件
  - Camera3Plus: 对HAL3标准中的部分定义进行封装、扩展, 以便在mihal中通用.
  - Stream: 针对camera3_stream进行扩张, 包含了BufferManager用于管理internal buffer (GraphicBuffer)
  - BGService: 是一个HIDL serice, 搭配MockCamera实现异步的大图回调
- SessionManager: 以Session为调度中心的后处理管理器.

#### CameraHALImpl

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203164909-20250311094648-fqqu2om.png)

CameraHALImpl是mihal用于兼容google camera3的接口, 沟通android和vendor的适配层.

- CameraManager: 管理CameraModule和CameraDevice, 提供获取CameraInfo和CameraDevice的接口. (vendor/xiaomi/proprietary/mivifwk-cn/hal/micam-core/CameraManager.cpp)
- CameraModule: 用于包装CameraDevice, 其派生类持有google camera_module.
  - VendorModule: 对camera_module的封装, 内部此有一个camera_module, 借助CameraDevice的封装此有camera3_device实例.
  - MockModule: 是mihal中单独实现的虚拟相机模块, 用于和app配合拍照后处理收大图
- CameraDevice: 是对camera3中camera_device的扩展思想, 自身此有hal3的camera3_device实例. (vendor/xiaomi/proprietary/mivifwk-cn/hal/micam-core/CameraDevice.cpp)
  - VendorCamera: 每个VendorCamera对应一个camera3_device实例, 后续app对camera的操作都可以在VendorCamera中进行捕获与扩展处理. (vendor/xiaomi/proprietary/mivifwk-cn/hal/micam-core/VendorCamera.cpp)
  - MockCamera: 对MockModule的camera3_device实现, app调用它来完成拍照后处理收大图. 其暴露给framework, 在拍照后处理在算法框架运行到最后一个plugin时需要获得app下发的output buffer, 这个output buffer就是app通过MockCamera下发. (vendor/xiaomi/proprietary/mivifwk-cn/hal/micam-core/MockCamera.cpp)

#### Mivi Foundation

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203165025-20250311094648-bcedz0c.png)

##### Camera3Plus

Camera3Plus对hal3中的数据结构进行拓展.

- StreamConfig
  - RemoteConfig: 封装mihal捕捉客制化的StreamConfig, 直接下发给SOC HAL
  - LocalConfig: 对特定场景的stream进行客制化, 包括stream信息和buffer
- StreamBuffer
  - RemoteStreamBuffer: 封装无需mihal客制化的buffer (来自CameraService的buffer)
  - LocalStreamBuffer: 封装mihal需要后处理的StreamBuffer
- CaptureRequest
  - RemoteRequest: 封装mihal不作处理的request, 直接下发给SOC HAL
  - LocalRequest: 封装需要后处理的request, 发送给SOC HAL或darkroom
- CaptureResult
  - RemoteResult: 封装mihal不作处理的result, 返回给framework
  - LocalResult: 对mihal需要处理的result进行修改和控制, 返回给framework
- NotifyMessage: 用于控制notify message的内容
- Metadata: 为mihal提供metadata相关操作接口

##### Stream

mihal使用StreamWraper类来管理streams. 每一个StreamWraper都包含一个BufferManager.

BufferManager用于申请graphic buffer. Mihal在CameraMode中构建internal stream, 规定buffer的规格, 后续通过StreamWraper中的BufferManager申请internal buffer.

注: 这里的buffer并不是hal3中的`camera3_stream_buffer_t`. mihal的buffer有两种, 分为LocalStream和RemoteStream. 其中CameraService传入的buffer属于RemoteStream, 会原封不动的发送到vendor hal. mihal中申请的buffer属于LocalStream, 用于封装mihal客制化的buffer.

BufferManager管理buffer的模型如下:

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203165037-20250311094648-rwhr6ty.png)
BufferManager通过Android接口(在mihal中以ImageBuffer的形式被封装)申请GraphicBuffer并对其管理.

上图不同颜色的箭头展示了BufferManager的主要功能.

- requestBuffer: 请求graphic buffer
- releaseBuffer: 释放graphic buffer, 先从mBusyBuffers中出名, 再添加到mIdleBuffers中供后续使用
- reclaim: 回收Buffer, 在request

##### BGService

vendor/xiaomi/proprietary/mivifwk/interfaces/bgservice/

MIVI2.0独立实现了一个名为IBGService的HIDL接口服务, 同时运行在camera app和camera provider中, 让两者直接通信. BGService由IBGService和IEventCallback两个部分组成.

- IBGService用于在SDK场景下获取云控信息. Camera app使用此接口获取`setEventCallback()`接口.
- IEventCallback中四个回调函数:
  - notifyCallback: mihal通过此接口将MockCamera下发的request的信息回调给app, 包含request buffer中的size, format等信息.
  - notifySnapshotAvailability(): mihal通过此接口控制app的拍照频率.
  - onCaptureCompleted: mihal会在合适的阶段通知app拍照完成, app会开放拍照点击, 而此次拍照处理并未真正完成, 将在后台继续进行后处理.
  - onCaptureFailed: soc hal取帧阶段出现错误或需要drop调这次拍照任务都会调用此接口通知app, mihal会取消后续后处理任务.

##### AppSnapshotController与MiAlgoTaskMonitor

AppSnapshotController是进行拍照控制的模块, 目标是实现平滑拍照，提升用户体验. 其为单例实现，包含三个关键接口和一个常驻的指令处理线程.

- onRecordUpdate: 外部调用，将最新的是否可以拍照的状态，更新给指令队列
- onRecordDelete: 在切换模式or退出相机时触发，清空指令队列，默认允许app拍照
- onRecordChangedLocked: 确认指令，唤醒指令处理线程并更新app状态，被onRecordUpdate和onRecordDelete所调用
- processCommandLoop: 常驻的指令处理线程，会将最新指令发送给app，最终实现继续拍照/停止拍照
- mBufferCapacity: 当前模式下stream的最大buffer数量的限定
- mBufferLimitCnt: 用来限定当前模式下最少保留拥有下一次拍照的buffer数量，该值与当前模式下可能需要的最大帧数相等。

mBufferCapacity和mBufferLimitCnt可通过mivivendorstreamconfig.json进行配置, 其中的VendorSnapshotBufferQueueSize对应BufferCapacity.

MiAlgoTaskMonitor针对算法框架的后处理任务并发情况, 结合AppSnapshotController对拍照频率进行补充调控. app能否进行下一次拍照由AppSnapshotController控制，受buffer数量及后处理任务数共同影响.

#### SessionManager

vendor/xiaomi/proprietary/mivifwk-cn/hal/micam-core/Session.cpp

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203165058-20250311094648-rgifg3c.png)
SessionManager (单例) 用于管理Session生命周期. Session通过对CameraMode和Photographer的管理实现整体业务控制. 其中分为3种Session:

- AsyncAlgoSession: 用于管理MIUICamera的各个拍照模式, 以异步方式实现拍照收图, 即quickview小图+大图异步后处理.
- SyncAlgoSession: 用于管理和小米合作的三方相机的拍照功能, 采取同步收图方案, 即app拍照直接收取最终jpeg大图, 不支持quickview, 属于原始android camera方案.
- NonAlgoSession: 无需任何拍照后处理, 通过此Session直接与SOC HAL沟通.

主要关注AsyncAlgoSession, 每次configure_stream对应一次Session创建.

##### CameraMode

vendor/xiaomi/proprietary/mivifwk-cn/hal/micam-core/CameraMode.cpp

CameraMode是Session的一部分, 客制化不同场景的stream控制策略, 创建RequestExecutor实现后处理管控. 其主要功能有:

- 搭配json客制化内容, 为不同场景构建相应的stream, 对发往vendor的streamConfig进行调整
- 创建并管理各个internal stream及其internal buffer
- 根据request和stream信息, 构建正确的Photographer, 实现具体拍照逻辑
- 依赖PostProcessorManager对mialgoengine完成stream配置和初始化, 构建后续MockCamera的配流信息.
- 以loop线程为核心实现quickview的主干逻辑

CameraMode针对场景模式, 完成大量stream客制化管理与分配.

##### RequestExecutor

vendor/xiaomi/proprietary/mivifwk-cn/hal/micam-core/RequestExecutor.cpp

RequestExecutor对request任务进行包装, mihal使用RequestExecutor进行任务与资源的管理, 每一个RequestExecuetor既是资源单位也是执行单位.

拍照的后处理需要利用RequestExecutor的派生类Photographer对PostProcessorManager进行任务分发.

也就是说, 各个Photographer(和Previewer)都是RequestExecutor的子类.

##### PostProcessorManager

vendor/xiaomi/proprietary/mivifwk/hal/micam-core/PostProcessor.h

PostProcessorManager是单例实现, 管理所有PostProcessor. 一个PostProcessor对应一个mialgoengine, 也就是说PostProcessorManager本质上是在管理算法框架实例.

首先来介绍以下PostProcessor.

PorstProcessor管理一个算法框架实例, 其主要接口为:

- PostProcessor(): PostProcessor的构造函数会创建新的算法框架实例.
- preProcessCaptureRequest: 算法框架实例进行算法预处理.
- processInputRequest: 将input frame输入算法框架.
- InviteForMockRequest: 通过bgservice的notifyCallback接口见算法框架的output buffer的规格通知camera app, 让app能够通过MockCamera下发对应规格的request.
- processOutputRequest: MockCamera下发的request将buffer通过此接口传给算法框架, 作为算法框架输出大图的output buffer.
- processMialgoResult: 算法框架输出的大图的output buffer通过此接口回调给MockCamera, 再以result返回给camera app, 完成收大图.
- flush: 对算法框架进行fush, 清空冗余任务.

而PostProcessorManager的大多数操作都通过调用PostProcessor来完成, 管理PostProcessor的接口如下:

- configureStreams: 传入对算法框架input/output stream的规格信息, 构建PostProcessor.
- quickFinishTask: 快速终止后处理任务.
- clear: 析构所有的PostProcessor.

PostProcessor与PostProcessorManager是mihal后处理的核心.

TODO, 待补充

## MTK HAL

MTK Camera HAL代码目录如下

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203165109-20250311094648-kkdck1p.png)
MTK Camera HAL3架构如下

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203165114-20250311094648-2ff406q.png)
MTK通过实现Android的ICameraProvider, ICameraDevice, ICameraDeviceSession和ICameraDeviceCallback等接口来实现自己的功能.

ICameraProvider对应CameraProviderImpl, 包在camera device manager外围, 是一个adapter, 适配不同版本的camera device interface. Camera Service可以通过ICameraProvider得到ICameraDevice.

ICameraDevice对应CameraDevice3Impl, ICameraDeviceSession对应CameraDevice3SessionImpl. 两者用于操作每一个camera. 还包含了 IVirtualDevice 的实现. 用于camera device manager去跟camera device交互.

### PipelineModel

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203165122-20250311094648-ovpez3b.png)
![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203165128-20250311094648-c346cqt.png)
![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203165135-20250311094648-lkjmblk.png)
PipelineModel是HAL3核心架构, 对上需要开放对Pipeline创建与操作的API, 对下需要建立Pipeline并管理Pipeline的生命周期. 上图描述了PipelineModel的在不同时期的任务. PipelineModel在config阶段根据APP的`createCaptureSession()`里面带下来的surface list, 推测Output, 按照request所需要的经过的Node推测Pipeline各个Node是否需要创建以及各个Node的I/O buffer, 建立整条PipelineModel. 接到上层传输下来的request, 转化为Pipleline统一的IPipelineFrame, 决定这个request的I/O buffer, Topological, sub frame, dummy frame, feature set等信息.

- Topological是指这个request要经过的处理节点 (Node) , 这个节点的拓扑图只能是config阶段创建的PipelineModel拓扑图的子集.
- sub frame指是否需要多帧frame来做出这个request, 例如4 frame MFNR拍照, 上层只会给1个request, 但做出1张MFNR结果需要4个frame叠合, 所以另外3个就是sub frame.
- dummy frame是指是否需要多帧来产生一张3A收敛的frame结果, 例如HDR一般需要EV +1/0/-1, 需要重新做AE计算, 但AE可能需要2-3 frame才能做完收敛, 多出的frame就叫做dummy frame.
- feature set是指这个request要用的feature, 例如HDR、MFNR, 美颜, 或者MFNR+美颜.

### PipelineContext

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203165144-20250311094648-yf58zsr.png)
PipelineContext主要包含有创建与管理Pipeline内部组件 (IPipelineNode (即各个HwNode) 和IPipelineFramework) 相关的对象以及Utils. 其中包括:

- PipelineDAG: 保存并管理Pipeline以及Node全部的Topological关系
- PipelineNodeMap: 保存并管理Pipeline以及Node全部IO map以及StreamInfo
- StreamBuilder: Pipeline Stream的构建
- PipelineBuilder: Pipeline的构建
- NodeBuilder: 各个HWNode的构建
- RequestBuilder: IPipelineFrame的构建
- InFlightRequest: 注册在Pipeline执行中的IPipelineFrame

### HWNode

HWNode负责整合MTK硬件功能, 软件算法, 3A算法. 以下对各个Node进行介绍.

#### P1Node

P1Node是pipeline的根节点, 输入数据来自于上层. P1Node的主要功能是输出raw buffer. P1Node会与CamIO, HAL3A, P2进行交互. 通过HAL3A get和set 3A信息, CamIO enque和deque buffer, 最后输出RAW图给P2Node处理.

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203165151-20250311094648-hrez5r4.png)
![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203165156-20250311094648-rz53c89.png)

#### P2CaptureNode

P2CaptureNode继承自IPipelineNode, 数据来源为P1Node, 主要负责Capture frame的处理, 其中包含:

- CaptureProcessor: 封装P2/Capture request, 决定request要做什么处理 (执行哪些node)
- CaptureFeaturePipe: 串联各个pipe node

过程如下, 其中CFP是Capture Feature Pipe的缩写

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203165207-20250311094648-89qjblf.png)
P2CaptureNode转换raw数据为yuv数据, 支持缩放与裁减.

更具体的说明可见[Capture Framework](https://online.mediatek.com/apps/quickstart/QS00137#QSS01441)

#### P2StreamingNode

P2StreamingNode继承自IPipelineNode, 数据来源为P1Node.

- P2StreamingNode: 接收Pipeline Frame, 转成MWFrame .
- DispatchProcessor: 控制feature的dispatch. 将一般的Streaming request dispatch给StreamingProcessor 处理. 将slow motion和debug的request dispatch给BasiceProcessor处理. 还可以控制debug机制(dump buffer, sanline等).
- StreamingProcessor: 准备feature on/off/combination parameters, 准备output buffer的crop calculation, enque到StreamingFeaturePipe中做feature处理.
- StreamingFeaturePipe: 做Raw转YUV, 结合MTK HW和三方algo实现各种feature.

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203165219-20250311094648-kmi2096.png)
P2StreamingNode转换raw数据为yuv数据, 支持缩放与裁减.

SFP Node (Streaming Feature Pipe) 是SFP下的各个node. request传到SFP后, SFP会往下丢到SFP Node处理. SFP Node的flow总是从RootNode开始, 中间根据不同的处理需求, 流经各个不同的node处理, 最终都会流到HelperNode处理. HelperNode会将处理结果callback回StreamingProcessor. TPINode是用于挂载三方算法的Node.

更具体的说明可见[Streaming Framework](https://online.mediatek.com/apps/quickstart/QS00137#QSS01440)

#### JpegNode

JpegNode转换YUV数据为Jpeg数据. 数据来源为P2CaptureNode或P2StreamingNode, 输出数据到app.

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203165229-20250311094648-7h0pw1e.png)
![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203165234-20250311094648-adnt0dx.png)

#### FDNode

生成FD信息.

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203165242-20250311094648-axu57y4.png)

其中MTK HAL相关总结如下:

- P1Node主要功能是负责出raw buffer, YUV Buffer. 通常与camIO,Hal3A,P2C,P2S打交道. Pass1Node 通过camIO enque, deque buffer送给P2C或者P2S处理.
- P2StreamingNode接收P1Node传过来的Pipeline Frame, 转成MWFrame, MWFrame封装了mNodeID, mNodeName以及Pipeline Frame.
- DispatchProcessor主要将一般的Streaming request dispatch给StreamingProcessor处理,还可以生成dumplugin来控制P2 dump buffer.
- StreamingProcessor get到FeaturePipeParam, 准备 feature on/off/combination parameters, 注册mCallback 给到后面helpnode直接回调, 然后enque到StreamingFeaturePipe中做feature处理.
- StreamingFeaturePipe做Raw转YUV, 串各个TPI Node实现第三方preview算法的各种feature
- TPINode主要是接收P2A输出的yuv, 通过TPIMgr call到plugin去执行process处理第三方算法
- TPIMgr主要是管理各个plugin,及对各个plugin进行排序等操作, 最后把各个plugin, property, Selection封装到PluginMap
- P2CaptureNode接收P1Node传过来的Pipeline Frame, 转成MWFrame
- CaptureProcessor通过获取MTK_FEATURE_CAPTURE这个tag来得到feature的参数, 并加到mFeatures, 并把当前request enque到CaptureFeaturePipe做多帧算法或单帧算法的处理
- CaptureFeaturePipe在构造的时候会创建RootNode, P2Node, MultiFrameNode, YUVNode, MDPNode等node及分配node id, 串联各个pipe node.
- JpegNode主要负责将yuv转换成Jpeg
- FDNode主要call到FD Algo去做 Face detection, smile, gestures, asd等detection
- AppStreamManager

  - config : 主要是创建createMetaStreamInfo, createImageStreamInfo并做mtk的转换
  - submitRequest: 主要是createMTKRequests, 还有通过CallbackHandler把result进行返回
- PipelineModelSession

  - open/close:主要通过HalDeviceAdapter中间类调power on/off
  - config: 根据app带下来的surfaceList及相关设定决定Pipeline各个Node是否需要创建以及各个Node的I/O buffer, 建立整条PipelineModel
  - request: 接到上层queue下来的request, 转化为Pipleline统一的IPipelineFrame, 然后通过PipelineContext send到RootNode处理
- Policy

  - config: 主要是做pipeline相关设定的配置, 主要是需要创建哪些HWNode, HWNode中各node的可能所有连接
  - request: 主要是决定实际的HWNode的Topology, 决定feature的相关设定, PipeineModelSession会根据解析出Policy设定再去 build 出 PipelineFrame queue到pipeline
- PipelineContext主要包含有创建/管理Pipeline内部组件, 内部主要包含PipelineDAG, PipelineNodeMap等
- MTK ISP Hidl主要就是将三方算法的整合, 由MTK HAL层内进行整合, 上移到app层, 以降低与MTK HAL的耦合度. 藉由app取得input buffer后, 通过isp hidl操作来做拍照jpeg等动作.

各个流程如下:

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203165251-20250311094648-8v70r5d.png)
![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203165258-20250311094648-iwbcnsq.png)
(Face Detection)

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203165304-20250311094648-ceflkmo.png)

#### PipelinePlugin

PipelinePlugin是HAL3统一的三方算法挂载界面, 无论是Streaming/Capture/DualCam, 还是RAW/YUV算法, 都是在PipelinePlugin里面挂载并运行. PipelinePlugin主要有2个部分:

- IInterface部分: 提供三方算法的接入, 会用selection详细描述可以提供的buffer与metadata形式；
- IProvider部分: 提供三方算法的挂载 (将三方算法能力提供出来) , 同样也会用selection来声明可以使用什么buffer与metadata形式；

例:

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203165310-20250311094648-etsv2as.png)
其中REGISTER_PLUGIN_INTERFACE与REGISTER_PLUGIN_PROVIDER分别为IInterface与IProvider注册.

TODO: 待补充三方算法挂载等内容[PipelinePlugin](https://online.mediatek.com/apps/quickstart/QS00137#QSS01437)

### 关键流程

### TinyMW

新一代MTK Camera架构

TODO: 待补充

## Qcom HAL

# Camera Driver

# Camera Hardware

# Camera Buffer

Camera的buffer在用途上主要分为两种类型:

- Input buffer: Input buffer是有实际内容的buffer, 通过request流程送到HAL中, 由HAL读取并进行一系列处理.
- Output buffer: Output buffer是一块内存, 其中不包含具体数据, HAL将处理得到的数据存入output buffer.

在Camera2 API和HAL3中, buffer被交由stream管理. 对stream的定义在前文介绍`camera3_stream_t`的时候已经说明, 其有四种属性, 分别为:

- resolution: 在图像处理或者视频流处理中, 用于存储图像或视频帧的缓冲区的尺寸, 决定了能够处理的最大图像尺寸和质量
- format: 缓冲区中数据的组织和编码方式, 决定了图像数据如何在内存中表示
- gralloc usage flag: 内存分配和访问权限指示, 告诉gralloc如何分配内存 (比如内存是用于CPU访问、GPU访问还是同时用于两者) 指定了内存区域可以被哪些组件访问 (如CPU、GPU、Camera等) 以及访问的方式 (读、写、渲染等) .
- maximum in-flight buffer count: 最大in-flight buffer数量 (多个request在HAL中等待处理, 这些request携带的buffer被称为in-flight buffer)

## Buffer的流向

`configure_stream()`配置的stream定义了HAL可以处理的buffer的类型, 每一个stream对应一种的buffer. 由于camera的buffer都随着request和result流动, 下文将通过梳理request和result的流程来对buffer的流向进行分析.

注: request和result携带的buffer类型是buffer_handle_t *, buffer_handle_t实际上是一个native_handle_t的指针, 而native_handle是对Gralloc分配的buffer的描述. camera_capture_request_t与camera_capture_result_t定义在frameworks/av/services/camera/libcameraservice/device3/Camera3OutputUtils.h

request与result是相机子系统中的重要概念, app向framework发送request来获取result.

- 一个request可以对应一系列的result
- request包含所有必要的配置信息, 存放于metadata中. 如: 分辨率和像素格式；sensor、镜头、闪光等的控制信息；3A 操作模式；RAW 到 YUV 处理控件；以及统计信息的生成等.
- request携带对应的surface (也就是框架里面的stream) , 用于接收返回的图像.
- Framework发送异步的request到HAL, 也就是说, 上一个request没有处理完, 也可以submit新的request. 多个request可以同时处于in-flight状态.
- HAL必须顺序处理request (总是按照FIFO (队列) 的形式) , 对于每一个request都要返回timestamp (shutter, 也就是帧的生成时间) , metadata, image buffers.
- HAL需要的信息都通过request携带的metadata接收, HAL需要返回的信息都通过result携带的metadata返回.
- snapshot的request拥有更高的优先级.

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203165339-20250311094648-7h62zc5.png)
上图为request的整体处理流程

- open 流程 (黑色箭头线条)

  - CameraManager注册AvailabilityCallback回调, 用于接收相机设备的可用性状态变更的通知.
  - CameraManager通过调用`getCameraIdList()`获取到当前可用的camera id, 通过`getCameraCharacteristcs()`函数获取到指定相机设备的特性.
  - CameraManager调用`openCamera()`打开指定相机设备, 并返回一个CameraDevice对象, 后续通过该CameraDevice对象操控具体的相机设备.
  - 使用CameraDevice对象的`createCaptureSession()`创建一个session, 数据请求 (预览、拍照等) 都是通过session进行. 在创建session时, 需要提供Surface作为参数, 用于接收返回的图像.
- configure stream流程 (蓝色箭头线条)

  - 申请Surface, 如上图的OUTPUT STREAMS DESTINATIONS框, 用于在创建session时作为参数, 接收session返回的图像.
  - 创建session后, surface会被配置成框架的stream. 在框架中, stream定义了图像的size及format.
  - 每个request都需要携带target surface用于指定返回的图像是归属到哪个被configure的stream的.
- request处理流程 (橙色箭头线条)

  - CameraDevice对象通过`createCaptureRequest()`来创建request, 每个request都需要有surface和settings (settings就是metadata, request包含的所有配置信息都是放在metadata中的) .
  - 使用session的`capture()`, `captureBurst()`, `setStreamingRequest()`, `setStreamingBurst()`等api可以将request发送到框架.
  - 预览的request, 通过`setStreamingRequest()`, `setStreamingBurst()`发送, 仅调用一次. 将request set到repeating request list里面. 只要pending request queue里面没有request, 就将repeating list里面的request copy到pending queue里面.
  - 拍照的request, 通过`capture()`, `captureBurst()`发送, 每次需要拍照都会调用. 每次触发, 都会直接将request入到pending request queue里面, 所以拍照的request比预览的request的优先级更高.
  - in-progress queue代表当前正在处理的request的queue, 每处理完一个, 都会从pending queue里面拿出来一个新的request放到这里.
- 数据返回流程 (紫色箭头线条)

  - 硬件层面返回的数据会放到result里面返回, 会通过session的capture callback回调响应.

注意到request进入IN-PROGRESS QUEUE后的流程在HAL中完成, 其具体过程如下图

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203165348-20250311094648-w7rcgts.png)

- request处理流程 (黑色箭头线条)

  - framework异步地submit request到HAL, HAL依次处理, 并返回result.
  - 每个被submit到HAL的request都必须携带stream. stream分为input stream和output stream: input stream对应的buffer是已有图像数据的buffer, HAL对这些buffer进行reprocess；output stream对应的buffer是empty buffer, HAL将生成的图像数据填充的这些buffer里面.
- input stream处理流程 (图中的INPUT STREAM)

  - request携带input stream及input buffer到HAL.
  - HAL进行reprocess, 然后新的图像数据重新填充到buffer里面, 返回到framework.
- output stream处理流程 (图中的OUTPUT STREAM)

  - result携带output stream及output buffer到HAL.
  - HAL经过一系列模块的的处理, 将图像数据写到buffer中, 返回到framework.

对buffer的流向有了一个总体的认识, 为了详细介绍buffer的管理, 需要先了解BufferQueue的概念, 见下文.

## CameraService中的buffer

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203165356-20250311094648-ipq2wdq.png)

- camera3_stream: stream基础类型, 定义了buffer的四个基本属性: resolution, format, gralloc usage flags, maximum in-flight buffer count.
- Camera3StreamInterface: 定义了一系列操作stream的接口, 如: `startConfiguration()`, `getBuffer()`, `returnBuffer()`等.
- Camera3Stream: 用于管理来自摄像机设备的单个输入或输出数据流的类, 并定义了一个内部状态机用于追踪stream的状态, 如: connected、configured等状态.
- Camera3IOStreamBase: 维护了一系列用于管理buffer的成员, 如: mTotalBufferCount (该stream允许的最大buffer数量) , mHandoutTotalBufferCount (发送给HAL的buffer总数, 包括input buffer和output buffer) 等.
- Camera3InputStream: 用于管理相机设备的单一输入数据流的类, 该类充当HAL的消费者适配器 (HAL是消费者, 该类用于获取Input Buffer, 然后分发给HAL使用) , 当HAL消费完成后, Input Buffer则会通过该类释放.
- Camera3OutputStream: 用于管理相机设备的单一输出数据流的类.
- Camera3BufferManager: 管理相机output stream使用的buffer的类. 它根据请求分配并分发Gralloc buffer给客户端 (例如Camera3OutputStream) . 当客户端请求buffer时, 如果已经分配了一些可用的buffer, buffer manager将选择一个buffer, 否则将分配一个buffer. 当buffer manager维护的已分配buffer过多时, 它将动态释放一些仅由该buffer manager拥有的buffer. 这样做可以减少内存占用, 除非内存占用已经很小, 而不会影响性能.

下图描述了request在Camera Service中的关键流程

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203165404-20250311094648-z4zg9xe.png)

- threadLoop: open camera时启动的线程函数, 用于向HAL发request, close camera时结束该线程. 当App没有下发request到framework时, 该函数会wait在waitForNextRequestLocked中, 等待request的到来.
- getInputBuffer: 仅当request携带input stream才会触发, 用于获取input buffer.
- getBuffer: 用于获取request携带的output stream的buffer.

首先`waitForNextRequestBatch()`获取到NextRequest, 而NextRequest包含了CameraApp在执行Request的时候带下来的CaptureRequest, 给CameraHardwareInterface使用的halRequest, outputBuffers.

接着`waitForNextRequestLocked()`从RequestQueue中拿出一个request并返回, 如果RequestQueue中没有request, 则从RepeatingRequestQueue中拿出一个request.

然后在`prepareHalRequests()`的过程中, 会通过CaptureRequest的setting设置, stream信息, buffer信息对halRequest进行填充. 通过CaptureRequest找到mOutputStreams,然后通过Camera3OutputStream的getBuffer方法获取output buffers, 这个是buffer的dequeue过程.

之后申请的buffer以buffer_handle_t的形式存于output buffers中. 同时这个output buffer会赋值到halRequest->output_buffers中.

最终填充好NextRequest之后调用`sendRequestsBatch()`提取halRequest然后调用`Camera3Device::HalInterface::processBatchCaptureRequests()`. 其中会调用`wrapAsHidlRequest()`把halRequest再次封装成device::Vx_x::CaptureRequest, 之后调用`processCaptureRequest_x_x()`交给CameraProvider进程处理. (x_x表示hidl版本号)

注: 无论是input buffer还是output buffer, 都有最大数量限制, 由stream的max_buffers来确定, 即上文提到的maximum in-flight buffer count, 这个值由HAL设置. 当buffer数量不够时, 会触发wait等待.

下图描述了result在Camera Service中的关键流程

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203165412-20250311094648-03zmsrz.png)

- processCaptureResult: HAL返回result到framework会触发该函数. 该函数会将metadata返回给app, 以及将output buffer和input buffer返回给stream.
- returnBuffer: 将output buffer还给stream.
- returnInputBuffer: 将input buffer还给stream.

### InputBuffer与OutputBuffer的申请与释放

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203165417-20250311094648-g42wjyn.png)
上图描述了通过Camera3Stream申请与释放InputBuffer的过程, 从图中可以看出, 最终buffer的实际申请和释放都是通过BufferItemConsumer (也就是BufferQueue中的消费者端) 的acquireBuffer和releaseBuffer来实现的.

- `acquireBuffer()`: 从生产者处获取buffer.
- `releaseBuffer()`: 将获取的buffer返回到队列, 以允许重复使用.

BufferItemConsumer是BufferQueue的消费者端点, 允许客户端访问BufferQueue中的整个BufferItem entry, 消费者可以采用同步或异步模式一次获取多个缓冲区, 以供其同时使用.

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203165423-20250311094648-1iwibk9.png)
对应上述流程图, 可以看出OutputBuffer的申请有三种情况:

- 当mUseBufferManager为TRUE时, 从Camera3BufferManager中申请.
- 当mUseBufferManager为TRUE, 但是无法从Camera3BufferManager中申请时, 从ANativeWindow中dequeue buffer. 这个buffer实际上也是Camera3BufferManager申请的, 然后将其attach到ANativeWindow.
- 当mUseBufferManager为FALSE时, 直接从ANativeWindow中dequeue buffer, 这个buffer则是由 ANativeWindow申请.

总而言之, output buffer要么从Camera3BufferManager中申请, 要么从ANativeWindow申请. 从前者申请的话, CameraService会拥有对output buffer更多的控制权, 如动态申请释放buffer来省内存.

将上面的四张图合成即可得到完整的buffer在CameraService中流转的过程.

## HAL中的buffer

继续上文中对request与result的分析, 对传入HAL层的buffer流转进行分析.

### MTK

vendor/mediatek/proprietary/hardware/mtkcam3/main/mtkhal/hidl/device/3.x/HidlCameraDeviceSession.cpp::processCaptureRequest()

### Qcom

TODO

# BufferQueue概览

BufferQueue可以将可生成图形数据buffer的组件 (生产方) 连接到接受数据以便进行显示或进一步处理的组件 (使用方) . 几乎所有在系统中移动图形数据buffer的内容都依赖于BufferQueue. BufferQueue是Android显示系统的核心, 实际上Surface, ImageReader, ImageWriter是对BufferQueue的生产者消费者的封装, 而BufferQueue是一个生产者消费者模型又是GraphicBuffer管理者, 它和显示系统以及Camera流媒体紧密关系着.

定义在frameworks/native/libs/gui/

## BufferQueue中的生产者与消费者框架

BufferQueue的核心逻辑是生产者消费者逻辑. 在BufferQueue这个生产者消费者框架中, BufferQueueCore可以理解为buffer的管理者, 它的消费者是BufferQueueConsumer, 它的生产者是BufferQueueProducer.

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203165431-20250311094648-hwr5t04.png)
上图描述了BufferQueue中的生产者与消费者的工作流程, 根据个人理解, 可以做一个比喻:

大厨在做菜, 顾客在吃菜, 菜用盘子装, 服务员送盘子.

Producer就是大厨, consumer是顾客, 盘子是buffer, 而服务员是binder.

大厨在做菜需要盘子装就是dequeue的过程, 装好菜后服务员上菜是queue的过程.

菜端给顾客吃是acquire的过程, 顾客吃完菜然后服务员把盘子送回去是release的过程.

在此过程中, buffer有5种状态, 分别为: free, dequeued, queue, acquired, shared.

状态变化如下图:

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203165437-20250311094648-n72bx1x.png)

- FREE: 表示缓冲区可供生产者出队. 该slot由BufferQueue拥有. 当调用`dequeueBuffer()`时, 它会转换为 DEQUEUED.
- DEQUEUED: 表示缓冲区已由生产者出队, 但尚未排队或取消. 生产者可以在相关释放栅栏发出信号后立即修改缓冲区的内容. 该槽位由生产者拥有. 它可以转换为QUEUED (通过`queueBuffer()`或`attachBuffer()`) 或返回FREE (通过`cancelBuffer()`或`detachBuffer()`) .
- QUEUED: 表示缓冲区已被生产者填充并排队供消费者使用. 缓冲区内容可能会在有限的时间内继续被修改, 因此在相关栅栏发出信号之前, 不得访问内容. 该slot(buffer, 下同)由BufferQueue拥有. 它可以转换为ACQUIRED (通过`acquireBuffer()`) 或FREE (如果另一个缓冲区以异步模式排队) .
- ACQUIRED: 表示缓冲区已被消费者获取. 与QUEUED一样, 在获取栅栏信号发出之前, 消费者不得访问内容. 该slot由消费者拥有. 当调用`releaseBuffer()`或`detachBuffer()`时, 它会转换为FREE. 分离的缓冲区也可以通过`attachBuffer()`进入ACQUIRED状态.
- SHARED: 表示此缓冲区正在共享缓冲区模式下使用. 它可以同时处于其他状态的任意组合 (FREE 除外, 因为这排除了处于任何其他状态) . 它也可以多次被出队、排队或获取.

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203165445-20250311094648-sbe33h9.png)

## BufferQueue的构成

BufferQueueCore是由BufferQueue调用`BufferQueue::createBufferQueue()`创建的. BufferQueue中主要定义了`BufferQueue::createBufferQueue()`和ProxyConsumerListener. 封装的消费者调用`BufferQueue::createBufferQueue()`创建BufferQueueCore, 然后根据BufferQueueCore创建原始消费者生产者BufferQueueConsumer和BufferQueueProducer. BufferQueueCore负责维护BufferQueue的基本数据结构, 而BufferQueueProducer和BufferQueueConsumer则负责提供操作BufferQueue的基本接口.

消费者BufferQueueConsumer在调用`BufferQueueConsumer::connect()`的时候把ConsumerListener相关的回调接口注册进BufferQueueCore供ProxyConsumerListener回调使用.

生产者BufferQueueProducer也有一个`BufferQueueProducer::connect()`接口, 其注册IProducerListener到BufferQueueCore中, 在消费者使用完GraphicBuffer释放的时候通过IProducerListener通知生产者.

同时这个IProducerListener会注册Binder死亡通知函数, 在死亡的时候回调BufferQueueProducer的`binderDied()`, 取消链接. 至此由BufferQueue, BufferQueueCore, BufferQueueConsumer, BufferQueueProducer组成的核心的生产者消费者模型就建立起来了.

### BufferQueue

在BufferQueue中首先定义了ProxyConsumerListener. ProxyConsumerListener类是ConsumerListener类的实现, 主要用来保存一个指向真正的consumer object的weak pointer. ProxyConsumerListener的主要目的是避免BufferQueue对象和consumer对象的循环引用. ProxyConsumerListener的使用在ConsumerBase中. ProxyConsumerListener的实现比较简单, 几个函数的功能主要是监听一些动作, 当这些对应的事件发生后, 及时通知另一端, 做数据同步 (producer和consumer两端的关系) .

然后定义了`createBufferQueue()`, 前文中提到过此函数用于创建BufferQueueCore等BufferQueue的基本组成.

### BufferQueueCore

- slots相关, 用于关联数据核心GraphicBuffer, 包括mSlots mQueue mFreeSlots mfreeBuffers mUnusedSlots,mActiveBuffers
- listener相关, 用于通知和回调, 包括mConsumerListener mLinkedToDeath mConnectedProducerListener
- Buffercount相关, 用于定BufferQuue中的Buffer数量, 包括mMaxBufferCount mMaxAcquiredBufferCount mMaxDequeuedBufferCount
- 一些设置项, 包括 mConsumerName mDefaultWidth mDefaultHeight mDefaultBufferFormat mDefaultBufferDataSpace. 主要是name, 宽高信息, format信息, dataspace信息等.

BufferQueueCore正常会放在消费者中去创建,两点原因:

1. 出于consumer准备好消费了再去生产的思想考虑, 因为生产的目的为了消费.
2. 以consumer作为核心端去管理, consumer创建方便统一管理.

### BufferSlot与BufferItem

BufferQueueCore管理buffer, 而其核心GraphicBuffer关联在BufferSlot中. BufferQueueCore定义了`BufferQueueDefs::SlotsType mSlots`, 其本质是一个BufferSlot数组, 而BufferSlot定义如下:

```C++
struct BufferSlot {
    sp<GraphicBuffer> mGraphicBuffer;
    EGLDisplay mEglDisplay;
    BufferState mBufferState;
    bool mRequestBufferCalled;
    uint64_t mFrameNumber;
    EGLSyncKHR mEglFence;
    sp<Fence> mFence;
    bool mAcquireCalled;
    bool mNeedsReallocation;
};
```

其中重点是`sp<GraphicBuffer> mGraphicBuffer`, 也就是实际使用的buffer. BufferQueue框架中, 消费者和生产者对buffer操作的单元核心就是一个BufferSlot, 也就是说所有对GraphicBuffer的操作都是针对BufferSlot来完成的.

具体的BufferQueueCore对BufferSlot的操作是通过BufferQueueCore中的`Fifo mQueue`. 其实际类型是`typedef Vector<BufferItem> Fifo`. 所以Fifo中存的是BufferItem, BufferItem中定义了mslots, 表示此BufferItem中的buffer的slot序号, 以此找到对应的BufferSlot.

可以简单理解成BufferQueueCore, 生产者从mQueue上获取BufferItem从而找到了对应的BufferSlot, 并对它完成一系列的操作, 之后, 放回到mQueue中供消费者使用, 消费者也是从mQueue上获取BufferItem从而找到对应的BufferSlot来消费, 消费完成之后放回mQueue.

注: 实际上不是真正的把BufferSlot取出放回mQueue, 而是mSlots索引值的传递过程.

### Surface与BufferItemConsumer

Surface是ANativeWindow的实现, 作用是将buffer送到BufferQueue. ANativeWindow 表示图像队列的生产者端. 它是Java中android.view.Surface对象的C对应物, 并且可以双向转换.

根据消费者的不同, 提交给ANativeWindow的图像可以显示在显示屏上或发送给其他消费者, 例如视频编码器.

Surface将buffer送到BufferQueue, 通常通过某种方式 (可能是OpenGL、软件渲染器或硬件解码器) 渲染帧并将其创建的帧转发到SurfaceFlinger进行合成的程序使用. 例如, 视频解码器可以渲染帧并调用`eglSwapBuffers()`, 这会调用在Surface中实现的ANativeWindow回调. 然后, Surface通过Binder将buffer转发到BufferQueue的生产者接口, 将新帧提供给消费者 (例如GLConsumer) .

BufferItemConsumer是一个BufferQueue消费者端点, 允许客户端访问BufferQueue中的整个BufferItem. 可以一次获取多个buffer供客户端同时使用. 此消费者可以采用同步或异步模式运行.

其中, BufferItemConsumer封装并简化了BufferQueueConsumer的操作, Surface封装并简化了BufferQueueProducer的操作 (两者都与BufferQueue相连) .

### ImageWriter与ImageReader

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203165457-20250311094648-sm9qmgl.png)
ImageReader (frameworks/base/media/java/android/media/ImageReader.java) 是消费者, 消费者承担创建BufferQueue的责任. 所以在ImageReader初始化的时候创建了BufferQueue以及最原始的生产者BufferQueueProducer和消费者BufferItemConsumer. ImageReader是java层的概念, 其调用JNI接口`ImageReader_init()` (frameworks/base/media/jni/android_media_ImageReader.cpp) 来初始化.

ImageReader工作的时候调用`acquirenextImage()`经过层层调用获取到BufferQueue中的buffer来使用, 并将状态改成acquired状态. 使用完成之后调用release接口放回到freeBuffers队列中并通知生产者释放Buffer.

ImageReader的基本操作流程如下:

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203165503-20250311094648-ktc931c.png)
ImageWriter (frameworks/base/media/java/android/media/ImageWriter.java ) 是生产者, 根据消费者的创建的原始生产者封装并创建自己, 然后link到消费者进程. 然后调用`dequeueInputImage()`最终调用`BufferQueueProducer:dequeueBuffer()`拿到buffer进行处理. 处理完成之后调用`queueInputImage()`最终调用`BufferQueueProducer:queueBuffer()`把buffer放回队列供消费者使用.

ImageReader的基本操作流程如下:

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203165512-20250311094648-xdl7pmp.png)

## BufferQueue中类关系总结

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203165517-20250311094648-evfz84s.png)

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203165526-20250311094648-xfg4fzd.png)
具体说明待补充 (TODO)

## BufferQueue在相机中的应用

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203165533-20250311094648-jwy4umb.png)

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203165541-20250311094648-zqqvvke.png)

TODO

### 补充

OpenGL能进行高效得渲染图形图像, 并支持各种复杂的特效和动画. 在Android当中, 运用的是OpenGL ES, 是OpenGL的一个轻量级版本, 其工作流程基本如下:

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203165552-20250311094648-mmoc2ad.png)
首先Camera把摄像头收集到的数据传给SurfaceTexture, SurfaceTexture将数据转换为OpenGL ES可用的纹理, 并将其更新到绑定的纹理对象中, OpenGL ES根据纹理ID获取到纹理对象后, 渲染成帧数据, 再通过EGL显示在 Surface上.

- 纹理: 是一种可以被OpenGL识别和使用的渲染数据源.
- 纹理对象: 是一个用于存储和处理纹理数据的对象.
- 纹理ID: 用于标识和操作这个纹理对象的唯一编号.
- SurfaceTexture: 将数据转换为OpenGL ES可用的纹理, 并将其更新到绑定的纹理对象中, 以供OpenGL ES进行渲染和显示.
- EGL: OpenGL ES与底层窗口系统 (如Android、Linux等) 之间的中间层, 用于管理与窗口系统的交互以及OpenGL ES渲染表面的创建、绑定和显示等操作.

# sp与wp

sp: system/core/libutils/StrongPointer.cpp

TODO

# 对Binder的一些理解

Binder通信是一种client-server的通信结构

- 从表面上来看, 是client通过获得一个server的代理接口, 对server进行直接调用
- 代理接口中定义的方法与server中定义的方法一一对应
- client调用某个代理接口中的方法时, 代理接口的方法会将client传递的参数打包成为Parcel对象；
- 代理接口将该Parcel发送给内核中的binder driver.
- server会读取binder driver中的请求数据, 如果是发送给自己的, 解包Parcel对象, 处理并将结果返回；
- 整个的调用过程是一个同步过程, 在server处理的时候, client会block住.

Bp, Bn同时实现一个接口I. Bp主要用于处理服务请求, 然后通过`transact()`通过binder将处理请求传给Bn. BpINTERFACE是client端的代理接口, BnINTERFACE是server端的代理接口.

BpBinder (Binder Proxy) 负责向Bnbinder (Binder Native) 发送调用请求的数据. BpBinder是client端binder通信的核心对象, 通过调用`transact()`, client就可以向Bnbinder发送调用请求和数据.

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241203165602-20250311094648-ft3jtwv.png)

1. client端通过`transact()`对binder驱动发起操作请求, 并等待返回结果.
2. binder驱动将该请求转发给对应的server端.
3. server端通过`onTransact()`来响应client的操作请求, 并将结果返回给client端.
