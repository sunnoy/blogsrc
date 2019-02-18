---
title: 理解ceph使用ceph-disk创建osd过程
date: 2018-08-10 12:12:28
tags:
- osd
- ceph
---


## 基本环境

- ceph 12.2.7
- os CentOS Linux release 7.5.1804 (Core)

**ceph架构网络上资料很多，本文不再赘述，本文聚焦osd和系统磁盘连接和OSD创建过程**

![osd](https://qiniu.li-rui.top/osd.png)

<!--more-->
## osd

### Object Storage Device(OSD)

osd在ceph集群中是对象存储设备，一个文件存入ceph集群，被划分为多个Object，Object形成只有ceph可以读懂的文件，这些文件的最终归宿就是osd。

osd在系统中是各种物理或者逻辑的存储单元，这些存储单元可以是块设备，可以是目录，也可以是lvm。

### osd磁盘结构

#### 数据类型

对bulestore来说，需要存放`ceph data`, `ceph block`, `ceph block.db`, `ceph block.wal` 四种类型的数据。

具体到磁盘上，就会分出两个区：

```bash
vdb             252:16   0   80G  0 disk
├─vdb1          252:17   0  100M  0 part /var/lib/ceph/osd/ceph-8
└─vdb2          252:18   0 79.9G  0 part
```
**可以看到只有一个分区被挂载到了ceph有关的目录，那另一个呢？**

#### 被挂载的分区

一个较小的分区挂载到`/var/lib/ceph/osd/ceph-8`，这个100M的分区里面有啥呢？下面可以看到里面存放了该osd的元数据。比如ceph集群的fsid，密钥等。

```bash
[root@node3 ceph]# ll /var/lib/ceph/osd/ceph-8/
total 52
-rw-r--r-- 1 root root 393 Aug 10 11:00 activate.monmap
-rw-r--r-- 1 ceph ceph   3 Aug 10 11:00 active
lrwxrwxrwx 1 ceph ceph  58 Aug 10 11:00 block -> /dev/disk/by-partuuid/d7809980-acc0-448b-ac34-73b5d0a59cd4
-rw-r--r-- 1 ceph ceph  37 Aug 10 11:00 block_uuid
-rw-r--r-- 1 ceph ceph   2 Aug 10 11:00 bluefs
-rw-r--r-- 1 ceph ceph  37 Aug 10 11:00 ceph_fsid
-rw-r--r-- 1 ceph ceph  37 Aug 10 11:00 fsid
-rw------- 1 ceph ceph  56 Aug 10 11:00 keyring
-rw-r--r-- 1 ceph ceph   8 Aug 10 11:00 kv_backend
-rw-r--r-- 1 ceph ceph  21 Aug 10 11:00 magic
-rw-r--r-- 1 ceph ceph   4 Aug 10 11:00 mkfs_done
-rw-r--r-- 1 ceph ceph   6 Aug 10 11:00 ready
-rw-r--r-- 1 ceph ceph   0 Aug 10 11:01 systemd
-rw-r--r-- 1 ceph ceph  10 Aug 10 11:00 type
-rw-r--r-- 1 ceph ceph   2 Aug 10 11:00 whoami

```

可以看到其中`block`是个链接文件，链接位置是`/dev/disk/by-partuuid/d7809980-acc0-448b-ac34-73b5d0a59cd4`

>目录/dev/disk内记录了很多Linux设备的持久化名称，是为了系统重启产生的盘符漂移。设备的持久化有很多类型,/dev/disk/下面有每种类型的文件夹:
>by-id  by-partlabel  by-parttypeuuid  by-partuuid  by-path  by-uuid
举个列子哈
>by-label Data -> ../../sda3
>by-uuid  b411dc99-f0a0-4c87-9e05-184977be8539 -> ../../sda3
>by-id  wwn-0x60015ee0000b237f -> ../../sda
>by-path  pci-0000:00:1f.2-ata-1 -> ../../sda
>by-partlabel  Home -> ../../sda3
>by-partuuid  039b6c1c-7553-4455-9537-1befbc9fbc5b -> ../../sda4

osd使用的**by-partuuid**，可见是个链接

那么d7809980-acc0-448b-ac34-73b5d0a59cd4又是啥呢？

```bash
[root@node3 ceph]# ll /dev/disk/by-partuuid/
total 0
lrwxrwxrwx 1 root root 10 Aug 10 11:00 64a13bae-7cbe-41a5-9864-10bffbfa0688 -> ../../vdb1
lrwxrwxrwx 1 root root 10 Aug 10 11:00 d7809980-acc0-448b-ac34-73b5d0a59cd4 -> ../../vdb2

```

原来是指向了`/dev/vdb2`，也就解释了只有`vdb1`被挂载。

**被存放到OSD的文件会进入到分区/var/lib/ceph/osd/ceph-8(vdb1)内，然后通过block链接存放到vdb2。使用by-partuuid保证了重启后即使发生盘符漂移OSD仍可以正常启动**

## 生成OSD过程

生成OSD有两个工具`ceph-disk`和`ceph-volume`

### ceph-disk

#### 如何使用

```bash
    prepare             Prepare a directory or disk for a Ceph OSD
    activate            Activate a Ceph OSD
    activate-lockbox    Activate a Ceph lockbox
    activate-block      Activate an OSD via its block device
    activate-journal    Activate an OSD via its journal device
    activate-all        Activate all tagged OSD partitions
    list                List disks, partitions, and Ceph OSDs
    suppress-activate   Suppress activate on a device (prefix)
    unsuppress-activate
                        Stop suppressing activate on a device (prefix)
    deactivate          Deactivate a Ceph OSD
    destroy             Destroy a Ceph OSD
    zap                 Zap/erase/destroy a device's partition table (and
                        contents)
    trigger             activate any device (called by udev)
    fix                 fix SELinux labels and/or file permissions

```

#### 示例

ceph-disk prepare 会隐式调用ceph-disk activate。

```bash
#准备磁盘，会自动启动OSD进程
ceph-disk prepare --bluestrore /dev/vdb
#激活磁盘，此步骤可以省略
ceph-disk activate /dev/vdb1
```

## ceph-disk执行过程

**我们使用ceph-disk -v 来追踪执行过程**

### 获取分区参数

文件`/etc/ceph.conf`会指定一些OSD的参数，如分区大小。ceph-disk执行的第一步就是获取配置参数。

```bash
#bulstore
bluestore_block_size
bluestore_block_db_size
bluestore_block_size
bluestore_block_wal_size
#osd
osd_mkfs_type
osd_fs_type
osd_mkfs_options_xfs
osd_fs_mkfs_options_xfs
osd_mount_options_xfs
osd_fs_mount_options_xfs
```

### 第一个分区

#### 分区

分区命令调用gpt分区工具`sgdisk`

```bash
/usr/sbin/sgdisk --new=1:0:+100M --change-name=1:ceph data --partition-guid=1:64a13bae-7cbe-41a5-9864-10bffbfa0688 --typecode=1:89c57f98-2fe5-4dc0-89c1-f3ad0ceff2be --mbrtogpt -- /dev/vdb
```
第一个分区默认分配100MB。
请注意`partition-guid`和`typecode`，`partition-guid`作为链接使用，`typecode`则是一个udev的钩子，是ceph做的标记。下面详细说明。

#### 刷新分区表

调用`partprobe`来刷新分区表

```bash
/usr/bin/udevadm settle --timeout=600
/usr/bin/flock -s /dev/vdb /usr/sbin/partprobe /dev/vdb
/usr/bin/udevadm settle --timeout=600
```
`udevadm settle --timeout=600`包裹在刷新分区表的命令是为了保证刷新分区表成功。
`flock -s`用来对/dev/vdb创建一个共享锁，就是其他进程请求独占锁失败，请求共享锁就会成功。

#### 生成链接

`partprobe`会发送一个**uevent**事件到udev的守护进程systemd-udevd，udev守护进程会根据ceph在`/lib/udev/rules.d/95-ceph-osd.rules`定义的各种规则来生成链接。

我们来看95-ceph-osd.rules中的规则

```bash
# OSD_UUID
ACTION=="add", SUBSYSTEM=="block", \
  ENV{DEVTYPE}=="partition", \
  ENV{ID_PART_ENTRY_TYPE}=="4fbd7e29-9d25-41b8-afd0-062c0ceff05d", \
  OWNER:="ceph", GROUP:="ceph", MODE:="660", \
  RUN+="/usr/sbin/ceph-disk --log-stdout -v trigger /dev/$name"
ACTION=="change", SUBSYSTEM=="block", \
  ENV{ID_PART_ENTRY_TYPE}=="4fbd7e29-9d25-41b8-afd0-062c0ceff05d", \
  OWNER="ceph", GROUP="ceph", MODE="660"
```

ceph中使用ID_PART_ENTRY_TYPE来定位OSD中的不同分区，匹配到以后去调用`/usr/sbin/ceph-disk --log-stdout -v trigger /dev/$name`来向udev发送**uevent**事件

但是我们注意到第一个分区的typecode在文件95-ceph-osd.rules并不存在，那么就暂时还不可以生成链接。

```bash
[root@node1 rules.d]# grep 89c57f98-2fe5-4dc0-89c1-f3ad0ceff2be 95-ceph-osd.rules
[root@node1 rules.d]#
```
### 第二个分区

分区过程和第一个分区是一样的，我们再走一遍

#### 分区

```bash
/usr/sbin/sgdisk --largest-new=2 --change-name=2:ceph block --partition-guid=2:d7809980-acc0-448b-ac34-73b5d0a59cd4 --typecode=2:cafecafe-9b03-4f30-b4c6-b4b80ceff106 --mbrtogpt -- /dev/vdb

```
这里使用了largest来用完盘vdb剩下的空间

我们标记出第二个分区的typecode是`cafecafe-9b03-4f30-b4c6-b4b80ceff106`

#### 刷新分区表

和第一个分区是一样的

```bash
/usr/bin/udevadm settle --timeout=600
/usr/bin/flock -s /dev/vdb /usr/sbin/partprobe /dev/vdb
/usr/bin/udevadm settle --timeout=600
```
这时我们发现第二个分区的typecode的`cafecafe-9b03-4f30-b4c6-b4b80ceff106`可以在95-ceph-osd.rules中找到

#### 生成链接

```bash
# BLOCK_UUID
ACTION=="add", SUBSYSTEM=="block", \
  ENV{DEVTYPE}=="partition", \
  ENV{ID_PART_ENTRY_TYPE}=="cafecafe-9b03-4f30-b4c6-b4b80ceff106", \
  OWNER:="ceph", GROUP:="ceph", MODE:="660", \
  RUN+="/usr/sbin/ceph-disk --log-stdout -v trigger /dev/$name"
ACTION=="change", SUBSYSTEM=="block", \
  ENV{ID_PART_ENTRY_TYPE}=="cafecafe-9b03-4f30-b4c6-b4b80ceff106", \
  OWNER="ceph", GROUP="ceph", MODE="660"

```
从规则中可以发现第二个分区会被作为bluestone的BLOCK使用，并且通过`/usr/sbin/ceph-disk --log-stdout -v trigger /dev/$name`生成链接

```bash
[root@node3 ceph]# ll /dev/disk/by-partuuid/d7809980-acc0-448b-ac34-73b5d0a59cd4
lrwxrwxrwx 1 root root 10 Aug 10 11:00 /dev/disk/by-partuuid/d7809980-acc0-448b-ac34-73b5d0a59cd4 -> ../../vdb2
```
经过了前面一阵折腾，ceph-disk已经分好了两个区，还把第二个区通过udev规则生成了链接。

### 第一个分区准备

#### 格式化

默认会格式化xfs格式

```bash
/usr/sbin/mkfs -t xfs -f -i size=2048 -- /dev/vdb1
```
#### 权限操作

然后ceph-disk会调用mount把刚刚格式化的分区进行挂载到/var/lib/ceph/tmp/里面，使用随机的路径。然后进行数据填充和更改权限

挂载

```bash
/usr/bin/mount -t xfs -o noatime,inode64 -- /dev/vdb1 /var/lib/ceph/tmp/mnt.EQKZPL
```

使用`restorecon`恢复属性

```bash
/usr/sbin/restorecon /var/lib/ceph/tmp/mnt.EQKZPL
```

在/var/lib/ceph/tmp/mnt.EQKZPL准备文件,主要下面的

```bash
[root@node3 ceph]# ll /var/lib/ceph/osd/ceph-8/
total 52
-rw-r--r-- 1 root root 393 Aug 10 11:00 activate.monmap
-rw-r--r-- 1 ceph ceph   3 Aug 10 11:00 active
lrwxrwxrwx 1 ceph ceph  58 Aug 10 11:00 block -> /dev/disk/by-partuuid/d7809980-acc0-448b-ac34-73b5d0a59cd4
-rw-r--r-- 1 ceph ceph  37 Aug 10 11:00 block_uuid
-rw-r--r-- 1 ceph ceph   2 Aug 10 11:00 bluefs
-rw-r--r-- 1 ceph ceph  37 Aug 10 11:00 ceph_fsid
-rw-r--r-- 1 ceph ceph  37 Aug 10 11:00 fsid
-rw------- 1 ceph ceph  56 Aug 10 11:00 keyring
-rw-r--r-- 1 ceph ceph   8 Aug 10 11:00 kv_backend
-rw-r--r-- 1 ceph ceph  21 Aug 10 11:00 magic
-rw-r--r-- 1 ceph ceph   4 Aug 10 11:00 mkfs_done
-rw-r--r-- 1 ceph ceph   6 Aug 10 11:00 ready
-rw-r--r-- 1 ceph ceph   0 Aug 10 11:01 systemd
-rw-r--r-- 1 ceph ceph  10 Aug 10 11:00 type
-rw-r--r-- 1 ceph ceph   2 Aug 10 11:00 whoami
```

因为使用的root来填充文件，所以要修改文件权限，又是一顿操作

```bash
command: Running command: /usr/sbin/restorecon -R /var/lib/ceph/tmp/mnt.EQKZPL/ceph_fsid.21979.tmp
command: Running command: /usr/bin/chown -R ceph:ceph /var/lib/ceph/tmp/mnt.EQKZPL/ceph_fsid.21979.tmp
command: Running command: /usr/sbin/restorecon -R /var/lib/ceph/tmp/mnt.EQKZPL/fsid.21979.tmp
command: Running command: /usr/bin/chown -R ceph:ceph /var/lib/ceph/tmp/mnt.EQKZPL/fsid.21979.tmp
command: Running command: /usr/sbin/restorecon -R /var/lib/ceph/tmp/mnt.EQKZPL/magic.21979.tmp
command: Running command: /usr/bin/chown -R ceph:ceph /var/lib/ceph/tmp/mnt.EQKZPL/magic.21979.tmp
command: Running command: /usr/sbin/restorecon -R /var/lib/ceph/tmp/mnt.EQKZPL/block_uuid.21979.tmp
command: Running command: /usr/bin/chown -R ceph:ceph /var/lib/ceph/tmp/mnt.EQKZPL/block_uuid.21979.tmp
```

### 第二个分区准备

ceph-disk充分利用资源，将第一个分区的挂载点，更换链接到第二个分区

```bash
adjust_symlink: Creating symlink /var/lib/ceph/tmp/mnt.EQKZPL/block -> /dev/disk/by-partuuid/d7809980-acc0-448b-ac34-73b5d0a59cd4
```

对第二个分区进行一顿操作后umount掉

```bash
command: Running command: /usr/sbin/restorecon -R /var/lib/ceph/tmp/mnt.EQKZPL/type.21979.tmp
command: Running command: /usr/bin/chown -R ceph:ceph /var/lib/ceph/tmp/mnt.EQKZPL/type.21979.tmp
command: Running command: /usr/sbin/restorecon -R /var/lib/ceph/tmp/mnt.EQKZPL
command: Running command: /usr/bin/chown -R ceph:ceph /var/lib/ceph/tmp/mnt.EQKZPL

unmount: Unmounting /var/lib/ceph/tmp/mnt.EQKZPL
command_check_call: Running command: /bin/umount -- /var/lib/ceph/tmp/mnt.EQKZPL
```

### 被遗忘的第一个分区

我们前面说到，第一个分区没有在udev规则里面，还没有形成链接，ceph-disk会调用sgdisk来更改第一个分区的typecode，然后刷分区表

```bash
/usr/sbin/sgdisk --typecode=1:4fbd7e29-9d25-41b8-afd0-062c0ceff05d -- /dev/vdb

command_check_call: Running command: /usr/bin/udevadm settle --timeout=600
command: Running command: /usr/bin/flock -s /dev/vdb /usr/sbin/partprobe /dev/vdb
command_check_call: Running command: /usr/bin/udevadm settle --timeout=600
```

我们看一下这个typecode`fbd7e29-9d25-41b8-afd0-062c0ceff05d`在udev规则里面，会被作为OSD对待

```bash
# OSD_UUID
ACTION=="add", SUBSYSTEM=="block", \
  ENV{DEVTYPE}=="partition", \
  ENV{ID_PART_ENTRY_TYPE}=="4fbd7e29-9d25-41b8-afd0-062c0ceff05d", \
  OWNER:="ceph", GROUP:="ceph", MODE:="660", \
  RUN+="/usr/sbin/ceph-disk --log-stdout -v trigger /dev/$name"
ACTION=="change", SUBSYSTEM=="block", \
  ENV{ID_PART_ENTRY_TYPE}=="4fbd7e29-9d25-41b8-afd0-062c0ceff05d", \
  OWNER="ceph", GROUP="ceph", MODE="660"
```

**好了**经过漫长的顿顿操作，ceph-disk终于准备好了两个分区，

下面ceph-disk将会执行一个重要的操作

```bash
/usr/bin/udevadm trigger --action=add --sysname-match vdb1
```
该操作会隐式调用`ceph-disk activate /dev/vdb`，完成osd的挂载和启动osd守护进程。

## 摧毁一个OSD

### 多次搭建会报错

```bash
command_check_call: Running command: /usr/bin/ceph-osd --cluster ceph --mkfs -i 5 --monmap /var/lib/ceph/tmp/mnt.JUr99n/activate.monmap --osd-data /var/lib/ceph/tmp/mnt.JUr99n --osd-uuid a123ca2a-e63c-423d-b547-4ac6002f76bd --setuser ceph --setgroup ceph
2018-08-10 10:07:25.183297 7f254d53bd80 -1 bluestore(/var/lib/ceph/tmp/mnt.JUr99n/block) _check_or_set_bdev_label bdev /var/lib/ceph/tmp/mnt.JUr99n/block fsid b831847a-4e49-4f4b-8eae-433088bb0ccc does not match our fsid a123ca2a-e63c-423d-b547-4ac6002f76bd
mount_activate: Failed to activate
unmount: Unmounting /var/lib/ceph/tmp/mnt.JUr99n
```

### 初始化OSD

在磁盘层面清洁OSD磁盘

```bash
ceph-disk zap /dev/vdb
```

对的，又是一顿操作

```bash
#分区1
/usr/sbin/wipefs --all /dev/vdb1
/usr/bin/dd if=/dev/zero of=/dev/vdb1 bs=1M count=100
#分区2
/usr/sbin/wipefs --all /dev/vdb2
/usr/bin/dd if=/dev/zero of=/dev/vdb2 bs=1M count=110
#整个磁盘
/usr/sbin/sgdisk --zap-all -- /dev/vdb
/usr/sbin/sgdisk --clear --mbrtogpt -- /dev/vdb
#刷新分区表
/usr/bin/udevadm settle --timeout=600
/usr/bin/flock -s /dev/vdb /usr/sbin/partprobe /dev/vdb
/usr/bin/udevadm settle --timeout=600
```




















