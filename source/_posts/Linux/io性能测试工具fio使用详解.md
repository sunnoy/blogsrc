---
title: io性能测试工具fio使用详解
date: 2019-02-21 12:12:28
tags:
- linux
---

# linux io模型

## 基础概念

### 同步与异步

线程的执行过程中，产生一个外部调用，如果需要等待该调用返回才能继续线程流则叫做同步，不需要等待结果返回线程流可以继续往下执行的情况叫做异步。
区分：线程流向下执行需不需要等待系统调用的结果。

<!--more-->

### 阻塞与非阻塞

线程执行过程中，产生一个外部调用后，会不会把该线程流“堵”住，会“堵”对应的是阻塞，反之为非阻塞。

## io模型

两两配对就有四种io模型

- 同步阻塞I/O
- 同步非阻塞I/O
- 异步阻塞I/O
- 异步非阻塞I/O（AIO）

![iomm](https://qiniu.li-rui.top/iomm.png)

详细的介绍请参考[该篇博文](https://juejin.im/post/5c0f1739f265da616c65724e#heading-3)

# fio

- [仓库地址](https://github.com/axboe/fio)
- [文档地址](https://www.li-rui.top/fio_doc_gitpages/index.html)

## 基本介绍

fio是一个io模拟软件，可以模拟大多数的程序使用io情形

## 安装

这里选择3.13版本，较新版本请到[该页面](https://github.com/axboe/fio/releases)下载。
fio二进制文件已经安装到/usr/local/bin/fio

```bash
wget https://github.com/axboe/fio/archive/fio-3.13.tar.gz
tar -xvf fio-3.13.tar.gz && cd fio-fio-3.13 && ./configure && make -j2 && make install
```

## 生成文档

fio使用sphinx来生成文档静态页面

```bash
pip install sphinx
```

创建文档页面文件，创建后的文档页面在fio-fio-3.13/doc/output/html。博主已经把该页面托管到[GitHub pages](https://www.li-rui.top/fio_doc_gitpages/index.html)

```bash
make -C doc html
```

# fio中概念

从下面的概念中我们可以了解fio是如何模拟和测试的

## I/O type

主要是描述io的方式，是否需要缓存，随机读，顺序读，随机写，顺序写等

## Block size

块写入的大小

## I/O size

io测试需要写的文件的大小

## I/O engine

使用哪一种io引擎来测试，Linux常用libaio

## I/O depth

当io引擎为async时，同时向系统发送多少个io请求

## Target file/device

fio测试的目标设备，可以是某个盘下面的一个文件，也可以是某个块设备

## Threads, processes and job synchronization

fio测试中所创建的线程等

# fio使用

## 示例

```bash
fio -directory=/tmp/ -direct=1 -iodepth 1 -thread -rw=randread -ioengine=posixaio  -bs=4k -size=2G -numjobs=2 -runtime=180 -group_reporting -name=rand_100read_4k
```

## 参数解释

- directory 为在某个位置下面写入文件，也可以使用 filename=/dev/sda:/dev/sdb，要注意**filename会清空磁盘**
- direct=1 为直接写 不经缓冲和缓存
- iodepth 为io队列为1，表示使用AIO时，同时发出I/O数的上限
- thread 以方式来进行测试，默认调用fork函数来创建进程
- rw 为测试模式 randread randwrite write read 
- ioengine 选取的io引擎 posix(Portable Operating System Interface for Computing System) aio引擎
- bs block size 写入块的大小 默认为4k 测试IOPS时，建议将bs设置为一个比较小的值，如4k。测试吞吐量时，建议将bs设置为一个较大的值，如1024k
- size 测试文件的大小
- numjobs 克隆几个任务来同时测试
- runtime  测试超时 单位为 秒
- group_reporting numjobs使用的时候 将结果进行组合起来
- rand_100read_4k 测试的名称

## 输出查看

### 执行状态输出

```bash
...
fio-3.13
Starting 2 threads
rand_100read_4k: Laying out IO file (1 file / 2048MiB)
rand_100read_4k: Laying out IO file (1 file / 2048MiB)
#线程数量 
Jobs: 2 (f=2): [r(2)][100.0%][r=1465KiB/s][r=366 IOPS][eta 00m:00s]
```

- 2 (f=2) 为线程数量
- [r(2)][100.0%] 当前的线程状态和完成百分比  线程状态为 r代表随机读

![fior](https://qiniu.li-rui.top/fior.png)

- [r=1465KiB/s][r=366 IOPS] 表示任务的读写速率和iops
- [eta 00m:00s] 测试剩余时间


### 结果输出

可从下面结果看出：

带宽：1673kB/s 平均IOPS：408 运行时长：180004msec 耗时：4890.90

```bash
#测试名称：（执行测试所用的groupid，测试一共用了多个job）：错误数量：pid：时间
rand_100read_4k: (groupid=0, jobs=2): err= 0: pid=21415: Mon Feb 25 11:26:05 2019
  #带宽：1673kB/s 平均IOPS：408 运行时长：180004msec
  read: IOPS=408, BW=1634KiB/s (1673kB/s)(287MiB/180004msec)
    slat (nsec): min=738, max=3000.2k, avg=7442.12, stdev=14630.71
    clat (usec): min=107, max=194521, avg=4883.46, stdev=5205.41
    #完成时间4890.90
     lat (usec): min=111, max=194526, avg=4890.90, stdev=5205.49
    clat percentiles (usec):
     |  1.00th=[  196],  5.00th=[  269], 10.00th=[  343], 20.00th=[ 1418],
     | 30.00th=[ 2835], 40.00th=[ 3785], 50.00th=[ 4555], 60.00th=[ 5211],
     | 70.00th=[ 5800], 80.00th=[ 6521], 90.00th=[ 7832], 95.00th=[ 9634],
     | 99.00th=[24511], 99.50th=[36439], 99.90th=[56361], 99.95th=[63177],
     | 99.99th=[94897]
   bw (  KiB/s): min=  784, max= 2104, per=100.00%, avg=1633.25, stdev=105.94, samples=720
   iops        : min=  196, max=  526, avg=408.24, stdev=26.49, samples=720
  lat (usec)   : 250=3.82%, 500=9.27%, 750=0.26%, 1000=0.15%
  lat (msec)   : 2=12.54%, 4=16.67%, 10=52.58%, 20=2.85%, 50=1.63%
  lat (msec)   : 100=0.21%, 250=0.01%
  cpu          : usr=0.26%, sys=0.29%, ctx=73983, majf=0, minf=15
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=73515,0,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
   READ: bw=1634KiB/s (1673kB/s), 1634KiB/s-1634KiB/s (1673kB/s-1673kB/s), io=287MiB (301MB), run=180004-180004msec

Disk stats (read/write):
    dm-0: ios=73459/75, merge=0/0, ticks=353202/395, in_queue=354050, util=99.94%, aggrios=73515/86, aggrmerge=0/5, aggrticks=353449/616, aggrin_queue=354216, aggrutil=99.93%
  vda: ios=73515/86, merge=0/5, ticks=353449/616, in_queue=354216, util=99.93%

```

#### read: IOPS=408, BW=1634KiB/s (1673kB/s)(287MiB/180004msec)

- read 表示测试的io方向 有 read/write/trim 可选
- IOPS 一秒内磁盘进行多少次 I/O 读写
- BW 磁盘的吞吐量 BW=千分二进制方式KiB/s (千分十进制方式kB/s)(千分二进制方式MiB/线程执行时间msec)  （1MB = 1,000 KB 1MiB = 1,024KiB）

#### slat/clat/lat/cpu

- slat Submission latency 提交延迟 其中stdev为standard deviation 标准偏差 时间单位nanoseconds(nsec), microseconds(usec),milliseconds(msec)
- clat Completion latency 完成延迟 从提交到完成  对于同步io为0
- lat Total latency 总延迟 从fio提交到完成的总时间
- clat percentiles 表示不同时间区间下的分布
- bw per 为每个线程所占该测试组bw的百分比 samples 为所用样本数量
- iops 基于样本的iops统计
- cpu usr=用户时间, sys=系统时间, ctx=上下文切换次数, majf=最大page faults, minf=最小page faults
- lat 此处的延迟细分 250=3.82%(百分之3.82的io请求在250usec内完成), 500=9.27%(百分之9.27的io在250-500usec时间完成)
- IO depths io depths在测试期间的分布
- IO submit io提交分布
- IO complete io完成分布
- IO issued rwts read/write/trim/s请求触发数量 以及相应的
- IO latency 和latency_target设定相关

#### Run status

- bw=1634KiB/s (1673kB/s) 总的bw, 1634KiB/s-1634KiB/s (1673kB/s-1673kB/s) 最小最大bw, 
- io=287MiB (301MB), 总的io
- run=180004-180004msec 改测试组中最短和最长线程执行时间

#### Disk stats (read/write)

- ios=63586 读/58 写,  执行的所用io次数
- merge=0/0,  io merge 次数
- ticks=353863/5759,  io ticks次数
- in_queue=359959, 花费在磁盘队列的时间
- util=99.96%,  测试周期内磁盘的占用状态
- aggrios=63635/68,  agg为aggregate 合计
- aggrmerge=0/6, 
- aggrticks=353889/7891, 
- aggrin_queue=361879, 
- aggrutil=99.95%

# fio常见测试示例

## 顺序读

block size是4M, 30个线程并发，持续时间200s

```bash
fio -direct=1 -iodepth=128 -rw=read -ioengine=libaio -bs=4M -size=1G -numjobs=1 -runtime=1000 -group_reporting -filename=/var/lib/ceph/osd/ceph-3/test0012 -name=Read_PPS_Testing
```

## 随机读

block size是4M, 30个线程并发，持续时间200s

```bash
fio -direct=1 -iodepth=128 -rw=randread -ioengine=libaio -bs=4M -size=1G -numjobs=1 -runtime=1000 -group_reporting -filename=/var/lib/ceph/osd/ceph-3/test0012 -name=Rand_Read_Testing

```

## 顺序写

block size是4M, 30个线程并发，持续时间200s

```bash
fio -direct=1 -iodepth=128 -rw=write -ioengine=libaio -bs=4M -size=1G -numjobs=1 -runtime=1000 -group_reporting -filename=/var/lib/ceph/osd/ceph-3/test00123 -name=Write_PPS_Testing

```

## 随机写

block size是4M, 30个线程并发，持续时间200s

```bash
$ fio -direct=1 -iodepth=128 -rw=randwrite -ioengine=libaio -bs=4M -size=1G -numjobs=1 -runtime=1000 -group_reporting -filename=/var/lib/ceph/osd/ceph-3/test001234 -name=Rand_Write_Testing

```










