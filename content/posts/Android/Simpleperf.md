---
title: Simpleperf
date: 2025-03-11T09:46:48Z
lastmod: 2025-03-11T09:46:48Z
categories: ["Android"]
tags: ["Performance", "Android"]
---

# Simpleperf简明教程

```
./run_simpleperf_on_device.py record -a -g -f 1000 [-e event1[:...],event2[:...],...] --duration 20
adb pull /data/local/tmp/perf.data
./report_html.py
```

# Simpleperf简介

现代CPU都有性能检测单元(PMU). PMU拥有硬件计数器, 可以对一些event(如CPU cycle, 执行的指令, cache miss)计数. Linux内核将这些硬件计数器包装到硬件perf event中并提供独立于硬件的软件perf event和tracepoint event.
Linux内核通过`perf_event_open()`系统调用将所有这些暴露给用户空间.

Simpleperf利用`perf_event_open()`收集并分析程序的数据与性能. 每台Android设备上都有对应架构的Simpleperf, 随NDK发布.

Simpleperf有三个主要的用法: stat, record, report.

## stat

stat命令给出在一段时间内相应进程中所有发生的event的数量. 其工作流程如下:

- 根据用户给出的选项, simpleperf通过系统调用获取数据.
- Linux内核对用户指定的进程进行event计数.
- 监测后, simpleperf从linux内核读取event计数并汇报总结.

## record

record命令记录一段时间内相应进程的样本. 其工作流程如下:

- 根据用户给出的选项, simpleperf通过系统调用获取数据.
- simpleperf在其与linux内核间创建映射缓冲区
- linux内核对用户指定的进程进行event计数.
- 每次发生给定数量的event后, linux内核将采集的数据dump到映射缓冲区.
- simpleperf从映射缓冲区读取数据并生成perf.data文件

## report

report命令读取一个perf.data文件和所检测进程使用的的动态链接库, 输出一份对perf.data文件的分析结果.

## 脚本工具

有一系列python脚本辅助分析数据:

- annotate.py: 用于标注分析数据对应的源文件
- app_profiler.py: 用于分析Android app和native程序.
- binary_cache_builder.py: 用于从Android设备上获取库文件.
- pprof_proto_generator.py: 用于将数据转化为pprof使用的格式.
- report.py: 用于提供图形化界面的分析结果.
- report_sample.py: 用于生成火焰图.
- run_simpleperf_on_device.py: 一个对于在设备上运行simpleperf的简单的包装.
- simpleperf_report_lib.py: 提供一个python接口, 用于分析数据.

# Simpleperf命令

让我们来看几个重要的simpleperf命令.
你可以使用

```bash
$ simpleperf --help
$ simpleperf subcommand --help
```

来查看simpleperf的各项命令和其选项.

## simpleperf list

simpleperf list用于列出此台设备上所有可以被监测的event. 由于不同的硬件与内核版本, 不同的设备或许有不同的event. 其输出结果如下:

```plain
$ simpleperf list

List of hw-cache events:
  # More cache events are available in `simpleperf list raw`.
  branch-load-misses
  branch-loads
  dTLB-load-misses
  ...

List of coresight etm events:
  ...

List of hardware events:
  cpu-cycles
  instructions
  ...

List of pmu events:
  arm_dsu_0/bus_access/
  arm_dsu_0/bus_cycles/
  arm_dsu_0/cycles/
  ...

List of software events:
  cpu-clock
  task-clock
  context-switches
  ...

List of tracepoint events:
  mali:mali_jit_alloc
  mali:mali_jit_free
  mali:mali_jit_report_gpu_mem
  ...
```

关于event的具体说明如下:

## simpleperf stat

上文中提到, simpleperf stat用于获取event计数器的值, 以下为例子:

```plain
$ simpleperf stat -p 10269 --interval 1000

Performance counter statistics:

#          count  event_name                # count / runtime
     152,648,756  cpu-cycles                # 1.160633 GHz      
      33,988,640  stalled-cycles-frontend   # 479.877 M/sec     
      12,316,638  stalled-cycles-backend    # 310.866 M/sec     
       4,667,866  instructions              # 518.541 M/sec     
         219,455  branch-instructions       # 52.156 M/sec      
           2,957  branch-misses             # 5.619 M/sec       
  597.966390(ms)  task-clock                # 0.119515 cpus used
           2,502  context-switches          # 4.184 K/sec       
           2,194  page-faults               # 3.669 K/sec       

Total test time: 5.003289 seconds.
```

通过-e选项可以指定监测的event(event之间用`,`分隔):

```plain
$ simpleperf stat -p 10269 -e kmem:kfree,kmem:kmalloc --interval 1000 

Performance counter statistics:

#  count  event_name     # count / runtime
  10,212  kmem:kfree     # 10.674 K/sec
  13,660  kmem:kmalloc   # 14.278 K/sec

Total test time: 20.004157 seconds.
```

有时你会看到这样的结果:

```plain
Performance counter statistics:

4,331,018  cache-references     # 4.861 M/sec    (87%)
3,064,089  cache-references:u   # 3.439 M/sec    (87%)
1,364,959  cache-references:k   # 1.532 M/sec    (87%)
   91,721  cache-misses         # 102.918 K/sec  (87%)
   45,735  cache-misses:u       # 51.327 K/sec   (87%)
   38,447  cache-misses:k       # 43.131 K/sec   (87%)
9,688,515  instructions         # 10.561 M/sec   (89%)

Total test time: 1.026802 seconds.
```

每行最后的百分比代表着在总的时间内此event被监测的时间, 这是由于监测的event数量过多, CPU的PMU数量不够, 内核让多个event共享PMU. 这会引入一个问题, 有些event是需要同时记录的, 我们可以通过--group来给event分组, 规避这个问题, 示例如下:

```bash
$ simpleperf stat -p ${pid} --group kmem:kfree,kmem:kmalloc --group cache-references,cache-references:u,cache-references:k --group cache-misses,cache-misses:u,cache-misses:k -e instructions --interval 1000
```

可以分别通过-p和-t选项指定监测的进程(此进程下的所有线程包括新创建的线程)和线程, 用-a监测整个系统. 上述例子使用-p指定了一个进程, 也可以使用','分隔进程以监测多个进程或线程, 如:

```bash
$ simpleperf stat -p 10269,12343 --interval 1000
```

可以分别使用--duration和--interval指定监测的总时间和输出检测结果的间隔.
也可以直接以命令作为simpleperf的参数来指定监测的进程与时间.

```bash
$ simpleperf stat ls -l
```

使用--pre-core输出每个核心的事件计数, 它可以用来查看事件如何分布在不同的核心上. 当使用--per-core参数声明非系统范围时, simpleperf会为每个核心上的每个受监视线程创建一个监视器. 当线程处于运行状态时, 所有核心上的监视器都会启用, 但只有运行该线程的核心上的监视器处于运行状态. 因此百分比注释显示 runtime_on_a_core/runtime_on_all_cores.

```plain
$ simpleperf stat --per-core ls -l

Performance counter statistics:

# cpu          count  event_name                # count / runtime
  1       12,514,418  cpu-cycles                # 0.854307 GHz      
  6        5,698,468  cpu-cycles                # 0.470897 GHz      
  1        3,976,746  stalled-cycles-frontend   # 268.844 M/sec     
  6        1,042,574  stalled-cycles-frontend   # 86.154 M/sec      
  1        3,646,916  stalled-cycles-backend    # 248.816 M/sec     
  6        2,171,044  stalled-cycles-backend    # 179.406 M/sec     
  1        5,924,169  instructions              # 406.161 M/sec     
  6        7,393,954  instructions              # 611.005 M/sec     
  1          905,337  branch-instructions       # 56.523 M/sec      
  6          637,775  branch-instructions       # 52.703 M/sec      
  1          154,883  branch-misses             # 9.713 M/sec       
  6           41,567  branch-misses             # 3.435 M/sec       
  1    22.585000(ms)  task-clock                # 0.567647 cpus used
  6    12.170846(ms)  task-clock                # 0.305899 cpus used
  1                1  context-switches          # 44.277 /sec       
  6                1  context-switches          # 82.164 /sec       
  1              346  page-faults               # 15.320 K/sec      
  6              202  page-faults               # 16.597 K/sec      

Total test time: 0.039787 seconds.
```

## simpleperf record

simpleperf record用于dump simpleperf的监测记录.

simpleperf record支持在上文中描述的选项, 并可以用-f或-c指定dump的频率. 其中, -f指定每秒dump的次数, -c指定event发生的次数后dump.
可以使用--cpu-percent来指定simpleperf使用的CPU时间的最大百分比(默认为25). 用--cpu指定监测的cpu.

```bash
$ simpleperf record --cpu 0-3,5 -f 1000 -p 10269 --duration 10

$ simpleperf record -c 10000 -p ${pid} --duration 10 --cpu-percent 50
```

在默认情况下, simpleperf在当前目录创建perf.data记录数据, 也可以使用-o指定数据的存储位置.

## simpleperf report

simpleperf report用于根据simpleperf record生成的perf.data生成报告. simpleperf report将不同的样本以函数名分组, 根据每个样本集包含的事件数对其进行排序并输出.

利用-i指定输入文件.

```plain
$ simpleperf record --cpu 0-3 -e instructions --duration 1 -o /sdcard/Download/t.data -g ls

simpleperf I cmd_record.cpp:824] Recorded for 0.0251095 seconds. Start post processing.
simpleperf I cmd_record.cpp:918] Samples recorded: 44. Samples lost: 0.

$ simpleperf report -i /sdcard/Download/t.data

Cmdline: /system/bin/simpleperf record --cpu 0-3 -e instructions --duration 1 -o /sdcard/Download/t.data -g ls
Arch: arm64
Event: instructions (type 0, config 1)
Samples: 44
Event count: 6592131

Overhead  Command     Pid   Tid   Shared Object                           Symbol
16.37%    ls          9239  9239  /apex/com.android.runtime/bin/linker64  [linker]__memcpy_aarch64_simd
12.65%    ls          9239  9239  /apex/com.android.runtime/bin/linker64  [linker]__memchr_aarch64
7.95%     ls          9239  9239  /apex/com.android.runtime/bin/linker64  [linker]BionicAllocator::get_small_object_allocator(page_info*, void*)
...
```

为了获取函数符号, simpleperf需要读取所监视进程的二进制文件以获取符号表与调试信息. 默认情况下, simpleperf可以根据系统提供的信息找到文件, 但是不排除有文件不存在或文件不包含符号表和调试信息的情况. 可以通过——symfs来指定查找的二进制文件路径.
可以使用-g和--full-callgraph来获取函数调用关系. 例子如下:

```bash
$ simpleperf report -i /sdcard/Download/t.data -g --full-callgraph
```

注: 如果需要生成函数调用关系, 需要在record的时候使用-g或--call-graph.

使用simpleperf时, 可以使用--comms, --pids, --tids, --dsos过滤数据.

- --comms: 仅显示指定线程名称(comm)的数据
- --pids: 仅显示指定进程ID的数据
- --tids: 仅显示指定线程ID的数据
- --dsos: 仅显示指定的DSO(Dynamic Shared Object, 一般是指动态链接库)相关数据

```bash
$ simpleperf report -i /sdcard/Download/t.data --dsos /apex/com.android.runtime/lib64/bionic/libc.so -g
```

# 脚本

Android NDK中有对simpleperf命令进行封装的python脚本, 可以帮助使用simpleperf.
各脚本的功能在简介中已经提及, 下面简单演示各脚本.
注: 使用脚本是在host机上使用, 需要使用adb连接设备. 脚本默认的生成和读取文件路径都是/data/local/tmp/perf.data

## app_profiler.py

app_profiler.py封装了simpleperf record. 用于记录进程的数据.

```bash
# 记录一个本机进程.
$ ./app_profiler.py -np surfaceflinger

# 记录给定 pid 的本机进程.
$ ./app_profiler.py --pid ${pid}

# 记录一个命令.
$ ./app_profiler.py -cmd "ls"

# 使用-r指定simpleperf record选项
$ ./app_profiler.py -cmd "ls" -r "-e cpu-cycles:u --duration 10"
```

## report.py

快捷分析数据, 支持simpleperf report的选项.

```bash
$ ./report.py -g

# 可以使用--gui来启动图形界面
$ ./report.py -g --gui
```

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/gui-20241008184627-zzyzju1.png)

## binary_cache_builder.py

binary_cache_builder.py文件可以从Android设备中提取二进制文件, 也可以在本地的目录中查找二进制文件. 默认情况下, 执行完app_profiler.py脚本后, 会在本地生成一个binary_cache的目录, 该目录用于保存在分析数据时所需要的调试信息和符号表.

```bash
# 通过从设备中拉取二进制文件,为 perf.data 生成 binary_cache
$ ./binary_cache_builder.py
```

## annotate.py

annotate.py读取perf.data和binary_cache中的文件, 根据数据和源文件做注解. 可以通过annotate.config设置.

## pprof_proto_generator.py

pprof时一个将数据可视化的分析工具, 可以从[github](https://github.com/google/pprof)获取.
pprof_proto_generator.py生成pprof使用的文件格式.

```bash
$ ./pprof_proto_generator.py
$ pprof -pdf pprof.profile
```

## report_html.py

将采集的数据转换成一个html文件, 通过浏览器以饼状图, 火焰图, 样本表的形式查看其内容.
![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/chart-20241008184627-5a57jy9.png)
注: 有时打开html文件后显示空白, 实际上是js文件无法获取到.

## inferno.py

用于生成火焰图.

```bash
# -t表示duration
$ ./inferno.sh --pid ${pid} -t 15
```

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/flame1-20241008184627-cyxuev1.png)

## 脚本的实际使用

上文描述了众多脚本, 稍显混乱, 下图描述了其中的关系.
![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/relationship-20241008184627-02zniij.png)
可以看出perf.data是核心文件, 只要有了perf.data(也就是simpleperf record成功输出数据), 就可以转化为各种格式的文件供分析.
通常用以下方法生成火焰图:

```bash
# 监测进程, 生成perf.data
$ ./simpleperf record -p ... -e ... -f ... -o /data/local/tmp/perf.data

# 生成火焰图
$ ./report_html.py

# 或者单独使用inferno.py抓取数据并生成火焰图
# 也可以使用--one-flamegraph整合生成一个火焰图
$ ./inferno.py -f ... --pid ... -t ...--symfs ...
```

## 火焰图

### 火焰图简介

火焰图是一种用于可视化软件程序的性能剖析数据的工具, 通常用于分析程序的CPU使用情况. 它通过将时间轴与堆栈跟踪信息结合起来, 以可视化的方式展示了程序在执行过程中的活动情况.
火焰图以层叠的矩形条形图展示数据, 其中每个矩形代表了程序中的一个函数或方法. 矩形的宽度表示该函数在整个执行过程中占用CPU的时间比例, 而矩形的高度则表示该函数在堆栈中的调用深度. 通过观察火焰图, 可以快速识别出执行时间较长的函数以及函数之间的调用关系, 从而定位性能瓶颈.

### 火焰图原理

火焰图的实现原理是对CPU进行周期性的采样. 通过设置一定的采样频率, 让脚本频繁地访问CPU上执行的指令的地址, 并记录当前调用栈. 每次访问CPU运行情况的瞬间, 可以理解为采样点. 在一定周期内采集到的点, 可以汇集成线,这个线的精细程度由采样频率决定. 采样的频率越高, 收集到的数据就越详细, 但也会对程序性能产生更大的影响.
关于CPU的采样精细程度, 假设CPU在T0时间时, 采集到的采样点P0正在执行方法M0. 下一次采集时, CPU在T1时间时, 采集到的采样点P1正在执行方法为M1. 对比P0和P1函数和栈帧情况:

- P0和P1完全相等,则M0和M1完全相等. 则可以认为从T0时间到T1时间段, CPU都在执行M0方法, 或者CPU都在执行M1方法.
- P0和P1不相等, 则M0和M1不相等. 则可以认为从T0时间到T1时间段, CPU至少执行了2个及其以上的方法, 其中包含M0和M1. 在T0到T1过程中, 有某些方法执行时间短, 采样频率设置长, 导致没有采集到.

将不同的采样点连成线, 然后进行调用关系的整理, 就生成了火焰图.

注: 火焰图横向顺序不代表真正程序执行顺序.

[Simpleperf 理论与实践](https://zhuanlan.zhihu.com/p/666315598)

### 差分火焰图

差分火焰图是一种通过对比两个时间点的火焰图来分析性能变化的可视化图(可以将对比机和测试机进行对比).它可以帮助开发者了解程序在不同时间段内的性能特征,并识别出在两个时间点之间发生的性能变化.差分火焰图的重点在于分析函数的样本被采集到的数量以及两个时间点之间样本的变化情况.

- 蓝色：表示在后一时间点相比于前一时间点,函数调用次数减少了.这可能意味着在优化或者修复了某些问题后,相关函数的调用次数减少了；
- 红色：表示在后一时间点相比于前一时间点,函数调用次数增加了.这可能意味着在代码中引入了新的性能问题或者程序流程发生了变化,导致某些函数被更频繁地调用.

**Step 1**: simpleperf抓两份对比的perf.data

```Shell
simpleperf record -a -e cpu-cycles -f 1000 -g --duration 10 -o perf_first.data
simpleperf record -a -e cpu-cycles -f 1000 -g --duration 10 -o perf_second.data
```

**Step 2**: 下载FlameGraph源码

```Shell
git clone https://github.com/brendangregg/FlameGraph
```

**Step 3**: 将perf.data数据转为folded文件

```Shell
# stackcollapse.py 来自NDK
python stackcollapse.py -i perf_first.data > perf_first_cold_start.folded
python stackcollapse.py -i perf_second.data > perf_second_cold_start.folded
```

**Step 4**: 生成差分火焰图

```Shell
# difffolded.pl 来自FlameGraph项目
perl difffolded.pl ./folded/perf1.folded ./folded/perf2.folded | perl flamegraph.pl --negate > diff.svg
```

![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/diff-20250311094648-m5wzzew.svg)

# Simpleperf与Perfetto

* Perfetto: 通过精确的事件数据了解设备上正在发生的事情
* Simpleperf: 获取特定时间短内具体的占用资源的函数
