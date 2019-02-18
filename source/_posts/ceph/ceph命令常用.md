---
title: ceph常用命令
date: 2018-11-04 12:12:28
tags:
- ceph
---
## ceph常用命令

### 集群

```bash
#查看集群状态
ceph -w

#查看集群详细状态信息
ceph health detail
```

### osd

```bash
#生成
ceph-disk prepare --bluestrore /dev/vdb
ceph-disk prepare --bluestore /dev/sdd --block.db /dev/sdg --block.wal /dev/sdh
#删除
for osd in {0..12}
#for varible1 in 1 2 3 4 5
do  
    ceph osd out osd.$osd
    ceph osd down osd.$osd
    ceph osd crush rm osd.$osd
    ceph auth del osd.$osd
    ceph osd rm osd.$osd
done

#强制down
fuser -k /var/lib/ceph/osd/ceph-7

for osd in {b,c,d,e} ; do ceph-disk zap /dev/sd$osd ;done

for osd in {b,c,d,e} ; do ceph-disk prepare --bluestore /dev/sd$osd ;done

#获取osd最大pg数量

```
<!--more-->


### pool

```bash
# 创建pool  
ceph osd pool create kube 128 128 replicated
#需要对pool进行应用划分，'cephfs', 'rbd', 'rgw'
ceph osd pool application enable kube rbd

#删除pool
ceph.conf 
mon allow pool delete = 1
#查看pool
ceph osd lspools
#1 .rgw.root,2 default.rgw.control,3 default.rgw.meta,4 default.rgw.log,

#删除池，需要写两次池的名称
ceph osd pool rm .rgw.root .rgw.root --yes-i-really-really-mean-it

#获取pg数量
ceph osd pool get .rgw.root pg_num

# 查看pool
rados lspools
```

### rgw

```bash
#查看总的服务器状态
systemctl status ceph-radosgw.target
```
### rbd

```bash
#创建rbd，mb
rbd create <image-name> --size <megabytes> --pool <pool-name> --image-format 2
#查看pool内rbd
rbd list volumes 
#查看rbd信息
rbd info volumes/volume-test02-system
#扩容缩容
rbd volumes/volume-test02-system resize --size 2000
#缩容
rbd volumes/volume-test02-system resize --size 1000 --allow-shrink
#挂到本地
rbd map {pool-name}/{image-name}
rbd showmapped
mkfs.ext4 /dev/rbd0
mount /dev/rbd0 /mnt/
#取消映射
rbd unmap /dev/rbd/{poolname}/{imagename}

#qcow2转导入ceph格式
qemu-img convert -f qcow2 -O raw linux74-centos7.464-bit.qcow2 centos7.4
#导入ceph集群中
rbd import {local} volumes/volume-test02-system
#导出到本地
rbd export volumes/volume-test02-system {local}
#删除rbd
rbd rm {pool-name}/{image-name}
#复制rbd
rbd cp {pool-name1}/{image-name1} {pool-name2}/{image-name2}

```

### rbd快照

![rbd](https://qiniu.li-rui.top/rbd.png)

使用思路是：先创建快照，然后对其进行保护，接着在此基础上以快照的方式使用

```bash
#创建快照
rbd snap create {pool-name}/{image-name}@{snap-name}
#保护快照
rbd snap protect {pool-name}/{image-name}@{snapshot-name}
#取消保护快照
rbd snap unprotect {pool-name}/{image-name}@{snapshot-name}
#查看某一个镜像快照
rbd snap ls {pool-name}/{image-name}
#回滚快照
rbd snap rollback {pool-name}/{image-name}@{snap-name}
#删除快照
rbd snap rm {pool-name}/{image-name}@{snap-name}
#删除镜像所有快照
rbd snap purge {pool-name}/{image-name}
#克隆快照
rbd clone {pool-name}/{parent-image}@{snap-name} {pool-name}/{child-image-name}
#拍平快照
rbd flatten {pool-name}/{image-name}
#查看快照的子孙
rbd children {pool-name}/{image-name}@{snap-name}
```

## osd标记

获取pg在osd中的分布数据

```bash
ceph pg dump pgs|awk '{print $1,$15}'|grep -v pg   > pg1.txt
```

### 删除osd

```bash
#从集群删除，盘数据还在
ceph osd set norebalance
ceph osd set nobackfill
ceph osd set norecover
ceph osd out 3
systemctl stop ceph-osd@3
#删除crush规则，osd map key
ceph osd purge 3 --yes-i-really-mean-it
#删除
umount /var/lib/ceph/osd/ceph-3
ceph-disk zap /dev/sdd
parted -s /dev/sdf rm 1
parted -s /dev/sdf rm 1
ceph-disk prepare --bluestore /dev/sdd --block.db /dev/sdg --block.wal /dev/sdh

ceph osd unset norebalance
ceph osd unset nobackfill
ceph osd unset norecover
```

### 重建osd

```bash
ceph osd destroy {id} --yes-i-really-mean-it
```





