---
title: BufferQueue概述
date: 2025-03-11T09:46:48Z
lastmod: 2025-03-11T09:46:48Z
categories: ["Android"]
tags: ["Camera", "Android"]
---

BufferQueue可以将可生成图形数据buffer的组件 (生产方) 连接到接受数据以便进行显示或进一步处理的组件 (使用方) . 几乎所有在系统中移动图形数据buffer的内容都依赖于BufferQueue. BufferQueue是Android显示系统的核心, 实际上Surface, ImageReader, ImageWriter是对BufferQueue的生产者消费者的封装, 而BufferQueue是一个生产者消费者模型又是GraphicBuffer管理者, 它和显示系统以及Camera流媒体紧密关系着.

定义在frameworks/native/libs/gui/

## BufferQueue中的生产者与消费者框架

BufferQueue的核心逻辑是生产者消费者逻辑. 在BufferQueue这个生产者消费者框架中,  BufferQueueCore可以理解为buffer的管理者, 它的消费者是BufferQueueConsumer, 它的生产者是BufferQueueProducer.

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/image-20241023192453-dunqqph.png)

上图描述了BufferQueue中的生产者与消费者的工作流程, 根据个人理解, 可以做一个比喻:

大厨在做菜, 顾客在吃菜, 菜用盘子装, 服务员送盘子.

Producer就是大厨, consumer是顾客, 盘子是buffer, 而服务员是binder.

大厨在做菜需要盘子装就是dequeue的过程, 装好菜后服务员上菜是queue的过程.

菜端给顾客吃是acquire的过程, 顾客吃完菜然后服务员把盘子送回去是release的过程.

在此过程中, buffer有5种状态, 分别为: free, dequeued, queue, acquired, shared.

状态变化如下图:

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/image-20241023192501-eap8xsz.png)

* FREE: 表示缓冲区可供生产者出队. 该slot由BufferQueue拥有. 当调用`dequeueBuffer()`​时, 它会转换为 DEQUEUED.
* DEQUEUED: 表示缓冲区已由生产者出队, 但尚未排队或取消. 生产者可以在相关释放栅栏发出信号后立即修改缓冲区的内容. 该槽位由生产者拥有. 它可以转换为QUEUED (通过`queueBuffer()`​或`attachBuffer()`​) 或返回FREE (通过`cancelBuffer()`​或`detachBuffer()`​) .
* QUEUED: 表示缓冲区已被生产者填充并排队供消费者使用. 缓冲区内容可能会在有限的时间内继续被修改, 因此在相关栅栏发出信号之前, 不得访问内容. 该slot(buffer, 下同)由BufferQueue拥有. 它可以转换为ACQUIRED (通过`acquireBuffer()`​) 或FREE (如果另一个缓冲区以异步模式排队) .
* ACQUIRED: 表示缓冲区已被消费者获取. 与QUEUED一样, 在获取栅栏信号发出之前, 消费者不得访问内容. 该slot由消费者拥有. 当调用`releaseBuffer()`​或`detachBuffer()`​时, 它会转换为FREE. 分离的缓冲区也可以通过`attachBuffer()`​进入ACQUIRED状态.
* SHARED: 表示此缓冲区正在共享缓冲区模式下使用. 它可以同时处于其他状态的任意组合 (FREE 除外, 因为这排除了处于任何其他状态) . 它也可以多次被出队、排队或获取.

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/image-20241023192508-xtndc76.png)

## BufferQueue的构成

BufferQueueCore是由BufferQueue调用`BufferQueue::createBufferQueue()`​创建的. BufferQueue中主要定义了`BufferQueue::createBufferQueue()`​和ProxyConsumerListener. 封装的消费者调用`BufferQueue::createBufferQueue()`​创建BufferQueueCore, 然后根据BufferQueueCore创建原始消费者生产者BufferQueueConsumer和BufferQueueProducer. BufferQueueCore负责维护BufferQueue的基本数据结构, 而BufferQueueProducer和BufferQueueConsumer则负责提供操作BufferQueue的基本接口.

消费者BufferQueueConsumer在调用`BufferQueueConsumer::connect()`​的时候把ConsumerListener相关的回调接口注册进BufferQueueCore供ProxyConsumerListener回调使用.

生产者BufferQueueProducer也有一个`BufferQueueProducer::connect()`​接口, 其注册IProducerListener到BufferQueueCore中, 在消费者使用完GraphicBuffer释放的时候通过IProducerListener通知生产者.

同时这个IProducerListener会注册Binder死亡通知函数, 在死亡的时候回调BufferQueueProducer的`binderDied()`​, 取消链接. 至此由BufferQueue, BufferQueueCore, BufferQueueConsumer,  BufferQueueProducer组成的核心的生产者消费者模型就建立起来了.

### BufferQueue

在BufferQueue中首先定义了ProxyConsumerListener. ProxyConsumerListener类是ConsumerListener类的实现, 主要用来保存一个指向真正的consumer object的weak pointer. ProxyConsumerListener的主要目的是避免BufferQueue对象和consumer对象的循环引用. ProxyConsumerListener的使用在ConsumerBase中. ProxyConsumerListener的实现比较简单, 几个函数的功能主要是监听一些动作, 当这些对应的事件发生后, 及时通知另一端, 做数据同步 (producer和consumer两端的关系) .

然后定义了`createBufferQueue()`​, 前文中提到过此函数用于创建BufferQueueCore等BufferQueue的基本组成.

### BufferQueueCore

* slots相关, 用于关联数据核心GraphicBuffer, 包括mSlots mQueue mFreeSlots mfreeBuffers mUnusedSlots,mActiveBuffers
* listener相关, 用于通知和回调, 包括mConsumerListener mLinkedToDeath mConnectedProducerListener
* Buffercount相关, 用于定BufferQuue中的Buffer数量, 包括mMaxBufferCount mMaxAcquiredBufferCount mMaxDequeuedBufferCount
* 一些设置项, 包括 mConsumerName mDefaultWidth mDefaultHeight mDefaultBufferFormat mDefaultBufferDataSpace. 主要是name, 宽高信息, format信息, dataspace信息等.

BufferQueueCore正常会放在消费者中去创建,两点原因:

1. 出于consumer准备好消费了再去生产的思想考虑, 因为生产的目的为了消费.
2. 以consumer作为核心端去管理, consumer创建方便统一管理.

### BufferSlot与BufferItem

BufferQueueCore管理buffer, 而其核心GraphicBuffer关联在BufferSlot中. BufferQueueCore定义了`BufferQueueDefs::SlotsType mSlots`​, 其本质是一个BufferSlot数组, 而BufferSlot定义如下:

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

其中重点是`sp<GraphicBuffer> mGraphicBuffer`​, 也就是实际使用的buffer. BufferQueue框架中, 消费者和生产者对buffer操作的单元核心就是一个BufferSlot, 也就是说所有对GraphicBuffer的操作都是针对BufferSlot来完成的.

具体的BufferQueueCore对BufferSlot的操作是通过BufferQueueCore中的`Fifo mQueue`​. 其实际类型是`typedef Vector<BufferItem> Fifo`​. 所以Fifo中存的是BufferItem, BufferItem中定义了mslots, 表示此BufferItem中的buffer的slot序号, 以此找到对应的BufferSlot.

可以简单理解成BufferQueueCore, 生产者从mQueue上获取BufferItem从而找到了对应的BufferSlot, 并对它完成一系列的操作, 之后, 放回到mQueue中供消费者使用, 消费者也是从mQueue上获取BufferItem从而找到对应的BufferSlot来消费, 消费完成之后放回mQueue.

注: 实际上不是真正的把BufferSlot取出放回mQueue, 而是mSlots索引值的传递过程.

### Surface与BufferItemConsumer

Surface是ANativeWindow的实现, 作用是将buffer送到BufferQueue. ANativeWindow 表示图像队列的生产者端. 它是Java中android.view.Surface对象的C对应物, 并且可以双向转换.

根据消费者的不同, 提交给ANativeWindow的图像可以显示在显示屏上或发送给其他消费者, 例如视频编码器.

Surface将buffer送到BufferQueue, 通常通过某种方式 (可能是OpenGL、软件渲染器或硬件解码器) 渲染帧并将其创建的帧转发到SurfaceFlinger进行合成的程序使用. 例如, 视频解码器可以渲染帧并调用`eglSwapBuffers()`​, 这会调用在Surface中实现的ANativeWindow回调. 然后, Surface通过Binder将buffer转发到BufferQueue的生产者接口, 将新帧提供给消费者 (例如GLConsumer) .

BufferItemConsumer是一个BufferQueue消费者端点, 允许客户端访问BufferQueue中的整个BufferItem. 可以一次获取多个buffer供客户端同时使用. 此消费者可以采用同步或异步模式运行.

其中, BufferItemConsumer封装并简化了BufferQueueConsumer的操作, Surface封装并简化了BufferQueueProducer的操作 (两者都与BufferQueue相连) .

### ImageWriter与ImageReader

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/image-20241023192515-n18x8e9.png)

ImageReader (frameworks/base/media/java/android/media/ImageReader.java) 是消费者, 消费者承担创建BufferQueue的责任. 所以在ImageReader初始化的时候创建了BufferQueue以及最原始的生产者BufferQueueProducer和消费者BufferItemConsumer. ImageReader是java层的概念, 其调用JNI接口`ImageReader_init()`​ (frameworks/base/media/jni/android\_media\_ImageReader.cpp) 来初始化.

ImageReader工作的时候调用`acquirenextImage()`​经过层层调用获取到BufferQueue中的buffer来使用, 并将状态改成acquired状态. 使用完成之后调用release接口放回到freeBuffers队列中并通知生产者释放Buffer.

ImageReader的基本操作流程如下:

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/image-20241023192522-onbxqrw.png)

ImageWriter (frameworks/base/media/java/android/media/ImageWriter.java ) 是生产者, 根据消费者的创建的原始生产者封装并创建自己, 然后link到消费者进程. 然后调用`dequeueInputImage()`​最终调用`BufferQueueProducer:dequeueBuffer()`​拿到buffer进行处理. 处理完成之后调用`queueInputImage()`​最终调用`BufferQueueProducer:queueBuffer()`​把buffer放回队列供消费者使用.

ImageReader的基本操作流程如下:

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/image-20241023192526-5hbc56w.png)

## BufferQueue中类关系总结

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/image-20241023192533-d0gg913.png)

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/image-20241023192538-nu4ovat.png)

具体说明待补充 (TODO)

## BufferQueue在相机中的应用

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/image-20241023192542-seo7rar.png)

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/image-20241023192547-gog5vso.png)

TODO

### 补充

OpenGL能进行高效得渲染图形图像, 并支持各种复杂的特效和动画. 在Android当中, 运用的是OpenGL ES, 是OpenGL的一个轻量级版本, 其工作流程基本如下:

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/image-20241023192555-yzvcsim.png)

首先Camera把摄像头收集到的数据传给SurfaceTexture, SurfaceTexture将数据转换为OpenGL ES可用的纹理, 并将其更新到绑定的纹理对象中, OpenGL ES根据纹理ID获取到纹理对象后, 渲染成帧数据, 再通过EGL显示在 Surface上.

* 纹理: 是一种可以被OpenGL识别和使用的渲染数据源.
* 纹理对象: 是一个用于存储和处理纹理数据的对象.
* 纹理ID: 用于标识和操作这个纹理对象的唯一编号.
* SurfaceTexture: 将数据转换为OpenGL ES可用的纹理, 并将其更新到绑定的纹理对象中, 以供OpenGL ES进行渲染和显示.
* EGL: OpenGL ES与底层窗口系统 (如Android、Linux等) 之间的中间层, 用于管理与窗口系统的交互以及OpenGL ES渲染表面的创建、绑定和显示等操作.
