---
title: perf介绍
date: 2019-12-05 12:12:28
tags:
- linux
---

# perf

perf使用事件计数器来分析Linux性能

<!--more-->

# 事件来源

![perf_events_map](https://qiniu.li-rui.top/perf_events_map.png)

大概分为下面几类：

- Hardware Events: CPU中的寄存器含有performance counters(性能计数器)，用来统计 Hardware event，例如 cpu-cycles、instructions executed 、cache-misses、branch mispredicted等。这些event构成了应用程序profiling的基础。
- Software Events: 基于内核计数器的低优先级events， 例如, CPU migrations(处理器迁移次数), minor faults(soft page faults), major faults(hard page faults).
- Tracepoints: 是散落在内核源代码中的一些 hook，用来调用probe函数，开启后，它们便可以在特定的代码被运行到时被触发，这一特性可以被各种 trace/debug 工具所使用。Perf 就是该特性的用户之一。假如您想知道在应用程序运行期间，内核内存管理模块的行为，便可以利用潜伏在 slab 分配器中的 tracepoint。当内核运行到这些 tracepoint 时，便会通知 perf。
- Dynamic Tracing： probe函数(探针or探测函数)，kprobe(kernel probe)内核态探针，用来创建和管理内核代码中的探测点。Uprobes，user-probe，用户态探针，用来对用户态应用程序进行探测点的创建和管理，关于kprobe和uprobe可参考对应的内核文档

# 安装

```bash
yum install perf -y
```

# 使用

## top

```bash
perf top
Samples: 833  of event 'cpu-clock', Event count (approx.): 97742399
Overhead  Shared Object       Symbol
```

分别是采样数（Samples）

事件类型（event）

事件总数量（Event count）

第一列 Overhead ，是该符号的性能事件在所有采样中的比例，用百分比来表示。

第二列 Shared ，是该函数或指令所在的动态共享对象（Dynamic Shared Object），如内核、进程名、动态链接库名、内核模块名等。

第三列 Object ，是动态共享对象的类型。比如 [.] 表示用户空间的可执行程序、或者动态链接库，而 [k] 则表示内核空间。

最后一列 Symbol 是符号名，也就是函数名。当函数名未知时，用十六进制的地址来表示。

## record与report

```bash
# 全局检测
perf record -g 

# 检测命令
perf record -g docker ps

perf report
```

## 烈焰图

```bash
git clone https://github.com/brendangregg/FlameGraph
cd FlameGraph
perf record -F 99 -ag -- sleep 60
perf script | ./stackcollapse-perf.pl > out.perf-folded
cat out.perf-folded | ./flamegraph.pl > perf-kernel.svg
```

然后使用浏览器打开svg文件

