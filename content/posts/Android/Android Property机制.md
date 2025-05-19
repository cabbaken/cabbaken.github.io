---
title: Android Property机制
date: 2025-03-11T09:46:48Z
lastmod: 2025-03-11T09:46:48Z
categories: ["Android"]
tags: ["Android"]
---

# Android Property机制

[Google Android Property](https://source.android.com/docs/core/architecture/configuration/add-system-properties)

## Android Property简介

Android里有很多属性(property)每个属性都是一个键值对, 他们都是字符串格式, 定义了Android系统的一些公共系统属性.
这些属性多数在系统启动时设定, 也有一些是动态加载的, 后加载的重名属性会覆盖前面的.
可以使用在shell中使用`getprop`和`setprop`获取或设置property.
Android Property机制的运作原理大致如下

1. 系统一启动就会从若干属性脚本文件中加载属性内容
2. 系统中的所有属性（key/value）会存入同一块共享内存中
3. 系统中的各个进程会将这块共享内存映射到自己的内存空间，这样就可以直接读取属性内了
4. 系统中只有一个实体可以设置、修改属性值，它就是属性服务（Property Service）
5. 不同进程只可以通过socket方式，向属性服务发出修改属性值的请求，而不能直接修改属性值
   6. 共享内存中的键值内容会以一种字典树的形式进行组织

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241115190658-20250311094648-flv8dyq.png)

## Property命名规则

Android Property的命名可以是任意字符串, 推荐以下格式:

```plain
[{prefix}.]{group}[.{subgroup}]*.{name}[.{type}]
```

- prefix: 可以是"", "ro", "persist".
  - ro: 表示此属性是只读属性, 无法修改.
  - persist: 表示此属性在设备重启后不会被重新写入.
- group: 一般作为子系统名使用, 如`audio`, `sys`, `vendor`, `camera`等. 如果需要, 可以加上一个subgroup做为group的补充.
- name: 表示此属性的名字.
- type: 作为属性名字的补充, 可选.
