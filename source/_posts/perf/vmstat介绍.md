---
title: vmstat介绍
date: 2019-12-05 12:12:28
tags:
- linux
---

# 简介

vmstat用来统计虚拟内存使用，进行信息

<!--more-->

# 使用方式

## 概览

```bash
# delay为统计间隔
# 统计次数
vmstat [options] [delay [count]]

```

## 选项

```bash

```

## 显示信息

```bash
vmstat -w

procs -----------------------memory---------------------- ---swap-- -----io---- -system-- --------cpu--------
 r  b         swpd         free         buff        cache   si   so    bi    bo   in   cs  us  sy  id  wa  st
 4  0            0      4559432       153668      1523132    0    0   171  1237   35 1035  13   5  80   2   0
```

r 运行队列中的进程数,在一个稳定的工作量下,应该少于处理器个数
b 等待队列中的进程数(等待I/O),通常情况下是接近0的


bi: 每秒读取的块数
bo: 每秒写入的块数


in: 每秒中断数，包括时钟中断。
cs: 每秒上下文切换数。


us: 用户进程执行时间(user time)
sy: 系统进程执行时间(system time)
id: 空闲时间(包括IO等待时间),中央处理器的空闲时间 。以百分比表示。
wa: 等待IO时间