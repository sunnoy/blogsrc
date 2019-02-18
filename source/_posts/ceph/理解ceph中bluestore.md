---
title: 理解ceph中bluestore
date: 2018-11-04 12:12:28
tags:
- ceph
---
# 理解ceph中bluestore
## 架构图

![Bluestore架构图](https://qiniu.li-rui.top/Bluestore架构图.png)
<!--more-->
### 组件

bluestore不使用本地文件系统，直接接管裸设备，并且只使用一个原始分区，HDD所在的物理块设备实现在用户态下使用linux aio直接对裸设备进行I/O操作。

bulestore直接管理的裸盘，所以要自己实现存储数据的组件
<!--more-->
#### RocksDB

数据的存储需要有元数据来支持，元数据存储需要数据库，RocksDB是一个键值数据库，用来存放元数据。

rocksdb本身是基于文件系统的。

#### BlueRocksEnv

rocksdb是基于文件系统的，BlueRocksEnv就是裸盘上实现的文件系统接口提供给rocksdb使用。

#### BlueFS

BlueFS是个小型文件系统实现，他实现了BlueRocksEnv接口，并解决元数据、文件空间及磁盘空间的分配和管理。


### 实现

从BlueStore 的设计和实现上看，可以将其理解为用户态下的一个文件系统，同时使用RocksDB来实现BlueStore所有元数据的管理，简化实现。

​对于整块数据的写入，数据直接以aio的方式写入磁盘，再更新RocksDB中数据对象的元数据，避免了filestore的先写日志，后apply到实际磁盘的两次写盘。同时避免了日志元数据的冗余存储占用，因为传统文件系统有他们自己内部的日志和元数据管理机制。

​对于随机IO，直接WAL的形式，写入RocksDB 高性能的KV存储中。

### db和wal

#### block.db

大小为不小于block的4%。

#### block.wal

wal优先使用ssd

#### block





