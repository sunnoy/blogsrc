---
title: dmsetup命令
date: 2018-11-13 12:12:28
tags:
- linux
---

# dmsetup命令

dmsetup 命令是一个用来与 Device Mapper 沟通的命令行封装器（wrapper），用来管理Device Mapper框架中的卷设备

## Device Mapper

Device Mapper 是 Linux2.6 内核中支持逻辑卷管理的通用设备映射机制，用于物理设备到虚拟设备的映射，它为实现用于存储资源管理的块设备驱动提供了一个高度模块化的内核架构

![dm](https://qiniu.li-rui.top/dm.gif)

逻辑设备有很多，lvm就是其中的一种，lvm底层机制就是用的Device Mapper，docker在centos中的存储驱动也是Device Mapper

<!--more-->

## dmsetup信息查看

### dmsetup info

查看Device Mapper设备概述

```bash
dmsetup info
Name:              docker-253:0-67136107-256b1b0d3215346141fa60ae9e56cb2f2a52ed40eb7fd5be2162aacb5871d8fd
State:             ACTIVE
Read Ahead:        8192
Tables present:    LIVE
Open count:        1
Event number:      0
Major, minor:      253, 4
Number of targets: 1

```

### dmsetup ls 命令

用来列出所有的逻辑卷设备

```bash
#对于多路径 dmsetup ls --tree
dmsetup ls
#逻辑卷名称 设备号
docker-253:0-67136107-256b1b0d3215346141fa60ae9e56cb2f2a52ed40eb7fd5be2162aacb5871d8fd  (253:4)
docker-253:0-67136107-9bdc5d27c455ca3e708acc1dd6a3bb4bb4d829a06e5aca7cf1572409de440444  (253:5)
docker-253:0-67136107-b050a4cbcd891ec738f35f12b3ee4bdb499a97b6e3b6c8b58b8ae74d29e448aa  (253:3)
docker-253:0-67136107-pool      (253:2)
centos-swap     (253:1)
centos-root     (253:0)
```

### dmsetup deps

显示依赖关系

```bash
[root@localhost ~]# dmsetup deps
docker-253:0-67136107-256b1b0d3215346141fa60ae9e56cb2f2a52ed40eb7fd5be2162aacb5871d8fd: 1 dependencies  : (253, 2)
docker-253:0-67136107-9bdc5d27c455ca3e708acc1dd6a3bb4bb4d829a06e5aca7cf1572409de440444: 1 dependencies  : (253, 2)
docker-253:0-67136107-b050a4cbcd891ec738f35f12b3ee4bdb499a97b6e3b6c8b58b8ae74d29e448aa: 1 dependencies  : (253, 2)
docker-253:0-67136107-pool: 2 dependencies      : (7, 0) (7, 1)
centos-swap: 1 dependencies     : (252, 2)
centos-root: 2 dependencies     : (252, 3) (252, 2)
```

### dmsetup table

用来查看物理设备到虚拟设备的映射表

```bash
dmsetup table
docker-253:0-67136107-256b1b0d3215346141fa60ae9e56cb2f2a52ed40eb7fd5be2162aacb5871d8fd: 0 20971520 thin 253:2 59
docker-253:0-67136107-9bdc5d27c455ca3e708acc1dd6a3bb4bb4d829a06e5aca7cf1572409de440444: 0 20971520 thin 253:2 61
docker-253:0-67136107-b050a4cbcd891ec738f35f12b3ee4bdb499a97b6e3b6c8b58b8ae74d29e448aa: 0 20971520 thin 253:2 29
docker-253:0-67136107-pool: 0 209715200 thin-pool 7:1 7:0 128 32768 1 skip_block_zeroing
centos-swap: 0 2097152 linear 252:2 2048
centos-root: 0 17842176 linear 252:2 2099200
centos-root: 17842176 62898176 linear 252:3 2048

```

## 操作类

### dmsetup remove

删掉逻辑卷

```bash
dmsetup remove centos-swap
```

## 和udev结合

[参见](https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/7/html/logical_volume_manager_administration/udev_device_manager)

