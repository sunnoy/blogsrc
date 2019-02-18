---
title: ceph-volume命令
date: 2018-11-13 12:12:28
tags:
- ceph
---

# 命令简介

`ceph-volume`命令用于以lvm方式创建osd，在ceph13中用于替代之前的`ceph-disk`。

该命令包含两个子命令

- `ceph-volume lvm`用于在裸盘上创建新的osd
- `ceph-volume simple`用于接管用`ceph-disk`创建的osd

本文主要介绍lvm子命令

<!--more-->

## ceph-volume lvm

### ceph-volume lvm prepare

创建osd

```bash
#使用已经存在的lvm
ceph-volume lvm prepare --bluestore --data vg/lv
#让ceph创建lvm
ceph-volume lvm prepare --bluestore --data /path/to/device

#还可以指定
--block_device
--block_uuid
--db_device
--db_uuid
--wal_device
--wal_uuid
```

### ceph-volume lvm activate

一旦执行完成后`ceph-volume lvm prepare`需要执行`ceph-volume lvm activate`，命令`ceph-volume lvm activate`用来启动systemd中osd进程

该命令执行需要提供osd id 和 uuid

```bash
#查看osd uuid
ceph osd dump 
#激活osd
ceph-volume lvm activate --bluestore 0 0263644D-0BF1-4D6D-BC34-28BD98AE3BC8
```

也可以执行，对已经启动的osd没有影响

```bash
ceph-volume lvm activate --all

ceph-volume lvm activate --all
--> OSD ID 9 FSID 9078fbcc-1faf-49a8-ba5a-a51a7f81f91a process is active. Skipping activation
```

### ceph-volume lvm create

该命令包含`prepare`和`activate`两个命令，执行完成后会立即让osd进入集群，分别使用前两个命令可以避免数据立即均衡

### ceph-volume lvm list

列出当前的lvm osd

```bash
#以json格式输出--format=json
ceph-volume lvm list 
```

### ceph-volume lvm zap

用来清除lvm格式的osd

```bash
#逻辑卷
ceph-volume lvm zap {vg name/lv name}

#块设备
ceph-volume lvm zap /dev/sdc1
```

如果要重建osd

```bash
ceph-volume lvm zap /dev/sdc --destroy
```

如果遇到逻辑卷无法删除

```bash
dmsetup remove {lv name}
```

## 共存

命令`ceph-volume`和命令`ceph-disk`都是创建osd，只不过是对磁盘的利用方式不一样，前者使用逻辑卷，后者直接使用块设备，因此他们也可以同时存在

```bash
sdc                                                                 8:32   0 558.4G  0 disk
├─sdc2                                                              8:34   0 558.3G  0 part
└─sdc1                                                              8:33   0   100M  0 part /var/lib/ceph/osd/ceph-6
sdj                                                                 8:144  0 558.4G  0 disk
└─ceph--fcb5fb68--4b6b--42eb--8b76--24ce37ad6dd7-osd--block--9078fbcc--1faf--49a8--ba5a--a51a7f81f91a
                                                                  253:3    0 558.4G  0 lvm
```


