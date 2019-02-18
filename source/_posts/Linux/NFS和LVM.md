---
title: NFS和LVM
date: 2018-07-30 12:12:28
tags:
- lvm
---

### 1. 服务端

```bash
yum install -y nfs-utils rpcbind    

#配置
vi /etc/exports
/kuaiyun/static 172.16.2.0/24(rw,no_root_squash,sync) 172.16.3.0/24(rw,no_root_squash,sync)

#启动服务
chkconfig nfs on
chkconfig rpcbind on
service rpcbind start
service nfs start
#查看客户端挂载状态
showmount -a

#添加网段路由

#windows挂载
#1. 首先进行nfs客户端安装
#2. 执行挂载命令
#3. 在cmd才可以
#4. 在计算机使用映射网络驱动器才可以重启存在\\nfsserver\files
 mount [NFS servers Hostname or IP address]:/[share name] [Local Drive to mount]:\

 #脚本

echo mount nfs at %date% %time% >>c:\nfs.log  
net start pcnfsd >>c:\nfs.log  
net use /pers:no >>c:\nfs.log  
mount -o mtype=hard -o pcnfs=localhost -u:root 192.168.6.55:/vmsnfs N: >>c:\nfs.log  
dir N: >>c:\nfs.log   
mount 172.16.3.202:/kuaiyun/static N:

#windows通过映射网络驱动器挂载

#路径填写 NFS servers Hostname or IP address\dir


```
<!-- more -->
### windows服务器端

**通过服务管理器来开启nfs**

```bash
#把需要共享的文件夹 直接右键nfs共享即可
```

### 2. 客户端

```bash
#查看服务端状态，服务端IP
showmount -e 172.16.3.202
#客户端需要安装
yum install nfs-utils -y
#挂载，默认为hard挂载
mount -t nfs -o rw 10.140.133.9:/export/home/sunky /mnt/nfs
#软挂载
mount -t nfs -o soft 10.220.129.47:/data data/
```

逻辑卷

```bash
#准备物理卷
pvcreate /dev/vdb /dev/vdc

#创建逻辑卷组
vgcreate vg_data /dev/vdb /dev/vdc

#创建逻辑卷，332798*4 mb
lvcreate -n lv_data -l 332798 vg_data

#格式化
mkfs.ext4 /dev/vg_data/lv_data

#创建挂载目录
mkdir -p /kuaiyun/static

#永久挂载
echo "/dev/vg_data/lv_data /kuaiyun/static/ ext4 defaults 0 0" >> /etc/fstab
echo "/dev/vdb1 /file xfs defaults 0 0" >> /etc/fstab

echo "192.168.9.41:/share/xlx /share/xlx nfs rw 0 0 " >> /etc/fstab
 

```