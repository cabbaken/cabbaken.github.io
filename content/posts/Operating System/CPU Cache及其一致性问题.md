---
title: CPU Cache及其一致性问题
date: 2025-03-11T09:46:48Z
lastmod: 2025-05-06T19:41:16Z
---

## Cache简介

Cache作为CPU高速缓存, 用于减少处理器访问内存所需平均时间.  
在远古时期, CPU并没有Cache, 但随着CPU的快速发展, CPU访问其registers与访问内存速度差距越来越大, 所以就需要Cache这一中间层来提升性能.  

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020250117151028-20250311094648-cf17dqm.png)
上图可见, CPU访问registers的延迟为0.3ns, 而访问内存的延迟为62.9ns. 如果没有Cache, CPU将要花费漫长的时间从内存中获取数据. 引入多级Cache, 等级越高, 速度越慢, 容量越大. 这种设计既填补CPU registers与内存之间的空白, 提升性能, 又能够平衡成本(更快的Cache会更贵).  
现代CPU一般使用3级Cache, 在3级Cache的缓冲, 相邻Cache之间和Cache与memory之间的延迟差距越来越小.  

以Arm64-v8a架构的Cortex-A53为例, 其Cache的组织方式如下图.
![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020250117152525-20250311094648-s8khsxw.png)
其中, 每个CPU core(核心)独享L1 Cache, 而两个CPU core组成一个Cluster(集群), 一个Cluster共享一个L2 Cache, 所有的Cluster共享L3 Cache, 而L3 Cache通过Bus(总线)与内存相连.  

## Cache的工作方式

程序执行时, CPU会从内存中读取数据到Cache中, 而这些数据是根据其地址来标志的. 32位系统的地址空间是$2^{32}$B(4GB), 64位系统的地址空间为$2^{64}$B(基于成本(TLB缓存)和复杂度考虑, 目前的CPU设计的寻址根据其架构不同只使用64bit中的一部分).  
一种比较直观的Cache使用方式是, 在Cache中存储一些访问过的数据, CPU下次读取数据就可以直接去Cache中找, 如果找不到就去内存中读取, 然后将数据存进Cache.  

### Cache的存取方式

程序访问内存具有空间局部性的特征, 短时间内访问的数据或是指令大概率会在一段连续的地址范围内, 因此Cache和内存之间的传输却不是以单个字节进行的, 而是一次取特定大小的连续内存. 这个特定大小我们就称为Cache Line, 也是Cache的基本组成单位.    

Cache Line的大小在4-128字节, 一般为64B. 事实上, Cache Line作为Cache的基本组成单位, 包含两个部分: Metadata和真正需要的数据, 我们所说的Cache Line的大小是指第二部分的大小(即真正从内存中映射的数据大小).  

Cache Line的大小也是我们在开发程序的时候要重视内存对齐的底层原因. 假设Cache Line的大小为64B, 有一个64字节大小数据, 如果地址对齐过, 那一次内存IO就可以完成访问, 但如果未曾对齐, 那就要两次内存IO才行.  
下文以64位系统, 32GB($2^{35}$B)内存, 1MB($2^{20}$B) L1 Cache, 64B($2^{6}$B) Cache Line为例.  

最早的时候Cache使用一种叫做Direct Mapping的组织方式. 在这种模式下, 64位地址(CPU想要访问的数据的地址)被分为3个部分:Indes, Offset, tag.  
L1 Cache中一共有$1MB/64B=2^{20}B/2^{6}B=2^{14}$个Cache Line, 因此我们需要14位Index来标记Cache Line. 找到了Cache Line之后, 需要6位Offset来在64B的Cache Line中找出相应Byte, 还剩下$64-6-14=44$位我们将其作为标记位(tag). 我们只需要两个部分也就是Index和Offset就可以定位Cache中一个Cache Line中的某一个字节, 而tag用于校验当前Index相同的地址是否真的是CPU需要的地址(Cache Line的这些数据存在其Metadata内).  
所以CPU访问使用Direct Mapping方式的Cache的流程如下:  

1. CPU给出访问地址
2. 地址被分为3个部分: Index, Offset, tag
3. CPU使用Index找到对应Cache Line
4. 校验所需地址的tag与Cache Line的tag是否一致
5. 根据Offset获取数据

这种方式存在着问题: 当我们需要访问多个地址时, 如果这些地址包含着相同的Index, 那么就会频繁的发生冲突, 因为他们只能被mapping到唯一的Cache Line(因为这种Direct Mapping本质上是Cache Line - Index的映射), 这种现象叫做Cache Thrashing(缓存颠簸).  

Direct Mapping存在的问题可以使用新的方式改善, 也是现在普遍使用的组织方式, 即Set Associative.  
总的来说, Set Associative将内存分为多个set, 每个set中有n个Cache Line, 这样在遇到Index相同但是tag不同的情况, 可以将这些数据都存放在同组的Cache Line中.  
下文将每组Cache Line数量设为4个.  
L1 Cache会有$2^{20}/2^{6}/4=2^{12}$个组的Cache Line, 也就是说, 我们需要用12个bit去找到Index(理论上), 6个bit找到Offset, 使用剩下的$64-12-6=48$bit作为tag.  
所以CPU访问使用n-way Set Associative方式组织Cache Line的Cache流程如下:  

1. CPU给出访问地址
2. 地址被分为3个部分: Index, Offset, tag
3. CPU使用Index找到对应set
4. CPU在对应set中找到tag相同的Cache Line
5. 根据Offset获取数据

对比Direct Mapping和Set Associative, 可以发现Set Associative的set大小为1时, 就可以看作是Direct Mapping. 而Set Associative的set大小为最大, 也就是整个Cache的时候, 被成为Fully Associative.  
在Fully Associative的情况下, 因为整个Cache就是一个组, 那么我们就不需要Index来寻找set. 因此地址只剩下了tag和Offset. 因为在访问Cache时高速缓存电路必须并行匹配许多tag, 想要构造又大又快的Fully Associative Cache十分困难且昂贵, 因此Fully Associative Cache只适合做小容量的Cache, 如TLB.  

## 缓存的存取策略

Cache的写入分为两种:

- 写直达: 把数据同时写入内存和Cache中.
- 写回: 数据仅仅被写入Cache Line里，只有当修改过的Cache Line被替换时才需要写到内存中.

### 写直达

写直达是保持内存与Cache一致性最简单的方式, 把数据同时写入内存和Cache. 在使用此方法时, 写入前会先判断数据是否已经在Cache中:

- 如果数据在Cache存在, 先将数据写入Cache, 再写入到内存.
- 如果数据不在Cache, 直接把数据写入到内存里面.
  写直达很简单, 但是问题明显, 无论数据在不在Cache里面, 每次写操作都会写回到内存, 这样写操作将会花费大量的时间, 无疑性能会受到很大的影响.

### 写回

为了改善写直达的性能影响, 出现了写回的方法. 在写回机制中, 当发生写操作时, 新的数据仅仅被写入Cache Line, 当修改过的Cache Line被替换时才需要写到内存中, 这减少了数据写回内存的频率, 提高系统的性能.
其具体实现流程如下:

- 如果数据已经存在于CPU Cache中, 则直接将数据更新到该Cache Line中, 并将该Cache Line标记为脏. 脏标记表示此时Cache Line中的数据与主内存中的数据不一致, 在这种情况下, 不需要立即将数据写回内存, 而是等到合适的时机(例如Cache Line被替换时)再进行写回.
- 如果数据不在CPU Cache中, 即缓存未命中, 这时需要找到合适的Cache Line来存放新的数据, 流程大致如下:
  - 检查原Cache Line(被替换的Cache Line)的状态: 判断它是否被标记为脏.
    - 原Cache Line是脏的：
      1. 将Cache Line中的脏数据写回内存, 确保内存中的数据是最新的.
      2. 从内存中读取当前需要的数据, 加载到这个Cache Line中(注意, 这一步是为了把新数据加载到缓存中, 便于后续的写入操作).
      3. 将要写入的数据更新到该Cache Line中, 并将其标记为脏.
    - 原Cache Line不是脏的：
      1. 从内存中读取当前需要的数据到Cache Line中.
      2. 将要写入的数据更新到Cache Line中, 并将该Cache Line标记为脏.

写回方式的优势在于, 减少了直接写回内存的次数, 提高了系统性能, 尤其在大量写操作能够命中缓存的情况下, CPU不必频繁地与内存交互, 从而显著提升数据访问效率.

### 缓存的一致性问题

在使用写回机制时, 数据的修改并不会立即写回主内存, 而是延迟到合适的时机才进行写回. 这种延迟写回的机制在多核处理器系统中可能会引发数据一致性问题, 因为在数据被修改且尚未写回主内存之前, 若其他核心也需要访问该数据, 可能导致不同核心之间的数据版本不一致(由于各个core都有独立的Cache).

## 系统视角下的Cache

### 物理核与逻辑核

- 物理核: 物理核是处理器中实际存在的硬件核心, 负责执行计算任务.
- 逻辑核: 逻辑核是操作系统看到的处理单元, 在支持超线程的CPU上, 通常将一个物理核虚拟为多个(通常是2个)逻辑核, 提升使用率, 这些逻辑核共享物理核的资源.
  在Linux中, 可以通过`/proc/cpuinfo`或`lscpu`来获取CPU的相关数据. 其中的`processor`表示逻辑核序号, 其对应的`physical id`表示此逻辑核对应的物理核.
  超线程里的2个逻辑核实际上是在一个物理核上运行的, 模拟双核运作, 共享该物理核的L1和L2缓存.

### 多级Cache

上文中提到, 现代CPU一般3级Cache, 一般结构如下:  
​![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020250120212616-20250311094648-yz7gn97.png)  
由上图见

* L1 Cache距离CPU core最近, 具有最快的访问速度但容量较小. 通常, L1 Cache会分为两部分: L1D(L1 Data)和L1I(L1 Instructions). 这是因为指令和数据的更新策略不同, 需要分别进行缓存管理(指令通常是可预测的, 这种设计可以让CPU能够同时访问指令与数据, 提高CPU的并行能力). 在CISC架构中存在变长指令, L1I会进行特殊的优化, 以提高对变长指令的访问效率. 每个物理核心都有自己独立的L1数据缓存(L1D)和L1指令缓存(L1I). L1 Cache的访问速度极快, 通常可以在几个CPU时钟周期内完成数据的读取和写入, 为CPU核心提供最快的数据访问支持.
* L2 Cache的容量比L1 Cache大, 但访问速度稍慢. 它充当了中间缓冲区的作用, 当L1 Cache未命中时, CPU会尝试从L2 Cache中获取数据. 可以将L2 Cache比作一个小型的数据仓库, 存储着更多可能被CPU核心频繁使用的数据. 根据CPU架构不同, 一般由1个或2个物理核共享一个L2 Cache, 确保其在访问数据时可以减少对其他核心的依赖，提高整体访问效率.
* L3 Cache通常具有更大的容量, 但访问速度相对较慢. 其在多核心处理器中扮演着重要的角色, 多个核心共享L3 Cache中的数据. 可以将L3 Cache比作一个大型的公共数据存储区, 为多个核心之间的协同工作提供支持. 如果L1和L2 Cache都未命中, CPU会尝试从L3 Cache中获取数据, 以减少直接访问主内存的高延迟.

在Linux系统中, 可以通过`/sys/devices/system/cpu`​查看相关信息.

‍
