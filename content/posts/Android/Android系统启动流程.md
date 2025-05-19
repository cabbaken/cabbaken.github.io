---
title: Android系统启动流程
date: 2025-03-11T09:46:48Z
lastmod: 2025-03-11T09:46:48Z
categories: ["Android"]
tags: ["Android"]
---

# Android系统启动流程

Android系统基于Linux，因此启动Android的第一步是启动Linux。

# Linux kernel startup

## Bootloader

Bootloader是在计算机或嵌入式系统启动时运行的一段小程序。其主要功能是初始化系统硬件和加载操作系统。Bootloader在电源开启后由系统的固件（BIOS/UEFI）调用，并准备系统硬件（如内存控制器、处理器、外设）以及软件环境，使其准备好加载并启动操作系统。

1. 电源开启：按下开机键，主板发送信号给电源。设备电源开启后，固件（BIOS/UEFI）执行自检（Power-On Self Test, POST）并识别启动设备。
2. 加载Bootloader：固件从预设的启动设备（如硬盘、SSD、USB驱动器或网络）中加载Bootloader到内存中执行。
3. 硬件初始化：Bootloader配置必要的硬件资源，如设置内存控制器、检测外部存储设备等。
4. 加载操作系统：Bootloader从存储设备读取操作系统的核心映像（如Linux的vmlinuz，Windows的bootmgr），并将其加载到内存中。
5. 传递控制权：一旦操作系统核心被加载且初始化完成，Bootloader将控制权传递给操作系统，完成启动过程。

## Linux kernel

kernel处理所有操作系统进程，例如内存管理（MM）、任务调度（schedule）、I/O、进程间通信（IPC）和整个系统控制。kernel启动分为两个阶段：

### Stage 1

在第一阶段，kernel还只是一个压缩镜像文件，所以第一阶段的工作就是将kernel压缩文件加载进内存。kernel会进行自解压（在ARM64架构中，kernel由bootloader解压），并初始化一些基本功能，例如基本内存管理、最基本的硬件初始化。基本完成了基础的CPU架构相关的工作，为第二阶段非架构相关的启动过程做准备。

### Stage 2

在第二阶段（start\_kernel()），kernel会设置中断处理（中断向量表），内存相关设置，挂载内存为initramfs（initial ram file system）或initrd（initial RAM disk），作为临时文件系统。

initrd让kernel能够完全启动并加载驱动（不需要依赖其他硬件设备），initrd包含了大部分可能会需要的驱动，让kernel更小。initramfs是早期用户空间，用于检测并加载需要的设备驱动到用户空间的文件系统。完成后，临时文件系统被真正的用户空间取代。内存被释放，kernel将会启动PID\=1的用户空间进程init。

## Idle

```Plain
start_kernel()->
    rest_init()->
        cpu_startup_entry()                  PID=0
```

Idle是PID\=0的进程。在start\_kernel()中，第一个执行的函数set\_task\_stack\_end\_magic(&init\_task)将自身进程与init\_task链接起来，而init\_task是一个全局变量，已经被初始化。也就是说，此时执行start\_kernel()的进程是系统中第一个进程，PID被初始化为0，也是唯一一个没有通过fork或kernel\_thread创建的进程。在start\_kernel()的最后，通过cpu\_startup\_entry()变为idle进程，CPU在空转时运行此进程。

## Init

```Plain
start_kernel()->
    rest_init()->
        user_mode_thread(kernel_init)->      寻找init的文件
            kernel_clone()->
                copy_process()->             PID=1，虽然此时PID=1，但是依旧是内核线程
                    run_init_process()->
                        kernel_execve()->
                            bprm_execve()->
                                exec_binprm()->
                                    search_binary_handler()->
                                        fmt->load_binary()->
                                            load_elf_binary()/load_flat_binary()->
                                                start_thread()->
                                                    start_thread_common()    这里转为用户态进程
```

Init是第一个被kernel启动的用户空间进程，所以PID为1。这是一个后台进程，不与用户直接交互。Init会引导用户空间（挂载文件系统，启动其它程序）。Init拥有多个runlevel（一般是7个，为0\~6），不同的runlevel对应init不同的运行状态。Android实现的init的具体功能如下：

FirstStageMain():

1. 挂载[tmpfs](https://docs.kernel.org/filesystems/tmpfs.html), proc, sysfs生成/proc, /dev, /sys
2. 加载kernel modules   `LoadKernelModules()`​
3. 切换新的根目录（依旧在ramdisk中），释放旧的根文件系统使用的内存（过去创造的临时文件系统）  `SwitchRoot()`​
4. 挂载设备树的fstab中描述的分区  `DoFirstStageMount()`​
5. 在firststage的最后重新运行init并加上参数"selinux\_setup"  `execv()`​

SetupSelinux():

1. 安卓旧版本会在这里为了兼容老机器挂载system\_ext.img  `MountMissingSystemPartitions()`​
2. 加载selinux相关设置  `ReadPolicy()`​  `LoadSelinuxPolicy()`​
3. 在最后重新运行init并加上参数"second\_stage"  `execv()`​

SecondStageMain():

1. 设置信号处理  `sigaction()`​
2. 挂载/apex，/linkerconfig    `MountExtraFilesystems()`​
3. 初始化并启动property服务  `PropertyInit()  `​  `StartPropertyService()`​
4. 挂载vendor overlay    `fs_mgr_vendor_overlay_mount_all()`​
5. 加载并解析rc文件    `LoadBootScripts()`​
6. 进入循环等待事件发生（通常是用户态进程的启动和结束，电源信号，通过/usr/sbin/telinit向init发出的请求） `auto pending_functions = epoll.Wait(epoll_timeout);`​

注：Init不在Linux kernel代码中实现，而Android使用的Init的实现在system/core/init

## Kthreadd

```Plain
start_kernel()->
    rest_init()->
        kernel_thread(kthreadd)              PID=2
```

kthreadd（kernel thread daemon）的PID为2，用于创建内核线程。无论是进程，还是线程，在kernel看来都是任务（Task），都使用相同的数据结构，平放在同一个链表中。这里的函数kthreadd，负责所有内核态的线程的管理，是内核态所有线程运行的祖先。事实上，init需要kthreadd才能完成进程创建，所以在代码中init需要调度器合理的调度两个进程的执行顺序才能够正常运行。

kernel\_thread()与user\_mode\_thread()的差别仅有一个.name和.kthread的args初始化差别。

注：Linux kernel startup部分基于Linux v6.11-rc3

至此，Linux启动完成。

# Zygote

Zygote是Android中的一个关键概念，它作为Android应用程序启动过程的启动点。Zygote用来孵化和管理Android系统中所有应用进程，在Init的第二阶段被创建，其主要作用是预加载和共享Android系统的核心库和常用资源，以减少每个独立应用程序启动时的时间和资源消耗。

在init运行的`SecondStageMain()`​中会解析rc文件，在`LoadBootScripts()`​中使用`Parser`​解析并rc文件。其中，会解析手机的system/etc/init/hw/zygote64\_32.rc（system/core/rootdir/init.zygote64\_32.rc），这正是zygote启动所用到的文件。其中

```Plain
service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server --socket-name=zygote
```

表示zygote服务启动的命令行参数，可以看到，实际上时启动了app\_process64这个程序，此程序源码位于frameworks/base/cmds/app\_process。以下为启动zygote的时序图：

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/image-20241023192313-p3eqxt3.png)

对此图的详细说明可见此[链接](https://elinux.org/Android_Zygote_Startup)。

ZygoteInit的代码在近十年间变化不大，这里不展开做详细说明。

其中AndroidRuntime::start()通过JNI启动了java虚拟机，并根据参数调用其main()函数（需要启动的java程序）。此时，zygoteInit开始运行，其源码位于frameworks/base/core/java/com/android/internal/os/ZygoteInit.java，这个时候，我们已经处于java虚拟机中。

ZygoteInit的任务为

* 预加载资源
* 创建并启动ZygoteServer，等待请求并创建新的进程

  ```Java
              zygoteServer = new ZygoteServer(isPrimaryZygote);

              if (startSystemServer) {
                  Runnable r = forkSystemServer(abiList, zygoteSocketName, zygoteServer);
                  // system_server是Zygote第一个fork出来的进程，负责启动各种服务，设置上下文，系统属性，环境等
                  // {@code r == null} in the parent (zygote) process, and {@code r != null} in the
                  // child (system_server) process.
                  if (r != null) {
                      r.run();        //启动system_service
                      return;
                  }
              }

              Log.i(TAG, "Accepting command socket connections");

              // The select loop returns early in the child process after a fork and
              // loops forever in the zygote.
              caller = zygoteServer.runSelectLoop(abiList);
  ```

其中，SystemServer(frameworks/base/services/java/com/android/server/SystemServer.java)是第一个被zygote创建的进程，负责设置系统属性、环境、库、context，启动并管理各种服务。

启动SystemServer后，ZygoteInit通过ZygoteServer.runSelectLoop()来等待请求并创建新的应用程序。

# SystemService

上文中提到的SystemServer创建了SystemServiceManager，这就是SystemService的启动者。

SystemServiceManage用来管理系统中的各种用于C/S架构中的Binder通信机制的Service，Client端使用某个Service，则需要先到ServiceManager中先查询Service的相关信息，然后根据Service的相关信息和Service的进程建立连接，这样Client端就可以使用Service了。

上述的SystemServer, SystemServiceManager, SystemService的关系如下:

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/image-20241023192320-terxr4u.png)

至此，系统启动完成。其总体过程如下图:

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/image-20241023192324-jpt34vi.png)

# APP启动

Android APP启动过程如下:

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/image-20241023192328-dwptpn4.png)

‍
