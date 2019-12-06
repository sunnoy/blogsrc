---
title: dstat介绍
date: 2019-12-05 12:12:28
tags:
- linux
---

# 介绍

一个替代 vmstat, iostat and ifstat 的工具

# 安装

```bash
yum install dstat -y
```

<!--more-->

# 使用

```bash
dstat
You did not select any stats, using -cdngy by default.
# hiq 硬中断
# siq 软中断
# int 中断
----total-cpu-usage---- -dsk/total- -net/total- ---paging-- ---system--
usr sys idl wai hiq siq| read  writ| recv  send|  in   out | int   csw
  2   2  95   1   0   0|  58k 2140k|   0     0 |   0     0 |5656    10k
  1   2  97   0   0   0|  12k  148k|  20k   13k|   0     0 |5078  9519
```

# 获取的统计项目

```bash
# 单项目，默认的项目
  -c, --cpu              enable cpu stats
     -C 0,3,total           include cpu0, cpu3 and total
  -d, --disk             enable disk stats
     -D total,hda           include hda and total
  -g, --page             enable page stats
  -y, --sys              enable system stats
  -n, --net              enable network stats
     -N eth1,total          include eth1 and total
# 下面不是默认的项目
  -i, --int              enable interrupt stats
     -I 5,eth2              include int5 and interrupt used by eth2
  -l, --load             enable load stats
  -m, --mem              enable memory stats

  -p, --proc             enable process stats
  -r, --io               enable io stats (I/O requests completed)
  -s, --swap             enable swap stats
     -S swap1,total         include swap1 and total
  -t, --time             enable time/date output
  -T, --epoch            enable time counter (seconds since epoch)


  --aio                  enable aio stats
  --fs, --filesystem     enable fs stats
  --ipc                  enable ipc stats
  --lock                 enable lock stats
  --raw                  enable raw stats
  --socket               enable socket stats
  --tcp                  enable tcp stats
  --udp                  enable udp stats
  --unix                 enable unix stats
  --vm                   enable vm stats

# 组合项目
  -a, --all              equals -cdngy (default)
  -f, --full             automatically expand -C, -D, -I, -N and -S lists
  -v, --vmstat           equals -pmgdsc -D total
```

## 进程统计

```bash
dstat -p
# run runing的进程
# blk blocked的进程
# new 新创建的进程
---procs---
run blk new
```