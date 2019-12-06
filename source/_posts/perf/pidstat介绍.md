---
title: pidstat介绍
date: 2019-12-05 12:12:28
tags:
- linux
---

# pidstat介绍

pidstat用来统计进程的信息

<!--more-->

# 安装

```bash
yum install sysstat -y
```

# 使用

## 看进程

```bash
pidstat -p 12349

Linux 3.10.0-957.27.2.el7.x86_64 (k8sdev-manager) 	12/05/2019 	_x86_64_	(4 CPU)

# %usr 用户态
# %system 内核态
# %CPU cpu使用量/cpu核心数量
# cpu 实际用的那个cpu
# %guest 用于虚拟机的cpu使用量
04:44:17 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
04:44:17 PM     0     12349   15.87    8.53    0.00   24.41     1  rancher
```

## 看io

```bash
pidstat -p 12349 -d
Linux 3.10.0-957.27.2.el7.x86_64 (k8sdev-manager) 	12/05/2019 	_x86_64_	(4 CPU)

# kB_ccwr/s pagecache的丢失
# todo 需要更多解释
04:47:55 PM   UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s  Command
04:47:55 PM     0     12349     46.97   2925.76     11.65  rancher
```

## 看缺页异常和内存使用效率

- major fault 如果内核找的数据从内存和cpu缓存中没有找的时候，需要访问磁盘，这个时候就会产生异常，因为内核不会把程序的全部数据都从磁盘放到内存中
- minor fault 物理内存中已经有了该page了，但是Memory management unit里面没有记录他，就会发生异常

```bash
# -l 写出执行命令
pidstat -p 12349 -r
Linux 3.10.0-957.27.2.el7.x86_64 (k8sdev-manager) 	12/05/2019 	_x86_64_	(4 CPU)

# minflt/s 每秒
# majflt/s 每秒
# VSZ 使用的虚拟内存 kb
# RSS 已经分配的物理内存
# %MEM 
07:54:13 PM   UID       PID  minflt/s  majflt/s     VSZ    RSS   %MEM  Command
07:54:13 PM     0     12349    137.24      0.26 16434936 1918880  23.96  rancher
```

## 看处理器统计

```bash
pidstat -p 12349 -u
Linux 3.10.0-957.27.2.el7.x86_64 (k8sdev-manager) 	12/05/2019 	_x86_64_	(4 CPU)

08:12:37 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
08:12:37 PM     0     12349    8.12    7.85    0.00   15.97     1  rancher
```

看详细的进程统计

```bash
# ms为毫秒
pidstat -p 12349 -T ALL -t
Linux 3.10.0-957.27.2.el7.x86_64 (k8sdev-manager) 	12/05/2019 	_x86_64_	(4 CPU)

08:15:29 PM   UID      TGID       TID    %usr %system  %guest    %CPU   CPU  Command
08:15:29 PM     0     12349         -    8.08    7.85    0.00   15.93     1  rancher
08:15:29 PM     0         -     12349    0.00    0.00    0.00    0.00     1  |__rancher
08:15:29 PM     0         -     12350    0.22    1.36    0.00    1.58     0  |__rancher
08:15:29 PM     0         -     12351    0.61    0.58    0.00    1.19     3  |__rancher
08:15:29 PM     0         -     12352    0.65    0.58    0.00    1.23     1  |__rancher
08:15:29 PM     0         -     12353    0.34    0.07    0.00    0.41     1  |__rancher
08:15:29 PM     0         -     12354    0.00    0.00    0.00    0.00     0  |__rancher
08:15:29 PM     0         -     12355    0.68    0.63    0.00    1.31     0  |__rancher
08:15:29 PM     0         -     12356    0.61    0.62    0.00    1.23     3  |__rancher
08:15:29 PM     0         -     12357    0.64    0.55    0.00    1.20     0  |__rancher
08:15:29 PM     0         -     12358    0.00    0.00    0.00    0.00     2  |__rancher
08:15:29 PM     0         -     12359    0.60    0.64    0.00    1.24     2  |__rancher
08:15:29 PM     0         -     12360    0.65    0.62    0.00    1.27     3  |__rancher
08:15:29 PM     0         -     12361    0.56    0.45    0.00    1.01     1  |__rancher
08:15:29 PM     0         -     12362    0.00    0.00    0.00    0.00     1  |__rancher
08:15:29 PM     0         -     12421    0.41    0.52    0.00    0.93     1  |__rancher
08:15:29 PM     0         -     12527    0.57    0.59    0.00    1.16     2  |__rancher
08:15:29 PM     0         -     12534    0.63    0.51    0.00    1.14     2  |__rancher
08:15:29 PM     0         -     12548    0.51    0.53    0.00    1.03     3  |__rancher

08:15:29 PM   UID      TGID       TID    usr-ms system-ms  guest-ms  Command
08:15:29 PM     0     12349         -   2705280   1959910         0  rancher
08:15:29 PM     0         -     12349   1009740    313520         0  |__rancher
08:15:29 PM     0         -     12350   1054960    598900         0  |__rancher
08:15:29 PM     0         -     12351   1138590    434540         0  |__rancher
08:15:29 PM     0         -     12352   1146550    434620         0  |__rancher
08:15:29 PM     0         -     12353   1081430    328070         0  |__rancher
08:15:29 PM     0         -     12354   1009600    313490         0  |__rancher
08:15:29 PM     0         -     12355   1151560    445960         0  |__rancher
08:15:29 PM     0         -     12356   1136910    443620         0  |__rancher
08:15:29 PM     0         -     12357   1144400    429430         0  |__rancher
08:15:29 PM     0         -     12358   1009600    313490         0  |__rancher
08:15:29 PM     0         -     12359   1136380    447620         0  |__rancher
08:15:29 PM     0         -     12360   1146030    443880         0  |__rancher
08:15:29 PM     0         -     12361   1127160    407620         0  |__rancher
08:15:29 PM     0         -     12362   1010130    313790         0  |__rancher
08:15:29 PM     0         -     12421   1095590    421890         0  |__rancher
08:15:29 PM     0         -     12527   1129210    437460         0  |__rancher
08:15:29 PM     0         -     12534   1141500    420300         0  |__rancher
08:15:29 PM     0         -     12548   1116220    423800         0  |__rancher
```


