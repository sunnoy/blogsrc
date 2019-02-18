---
title: gpt分区
date: 2018-11-04 12:12:28
tags:
- gpt
---
## gpt分区

### parted

```bash
#进入交互式
parted /dev/sdb
#更改分区表，会清除数据
mklabel gpt
#打印分区信息
print
#分区
mkpart primary 0GB 100%

```
<!--more-->

### 非交互式

```bash
mkpart [part-type fs-type name] start end
#type
primary  extended  logical
#fs

#更改分区表
parted -s /dev/$i mklabel gpt
#mkpart创建一个分区，共日志区域可用范围
parted -s /dev/$i mkpart journal ext4 2048s 20G
#创建一个分区除了日之外均为存放数据
parted -s /dev/$i mkpart data ext4 20G 100%
#删除分区


```

### 加入挂载

```bash
#/etc/fstab
/dev/xvdd1 /data1 ext4 defaults 0 0
#执行挂载
mount -a
```