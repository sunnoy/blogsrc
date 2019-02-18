---
title: pacemeker+iSCSI+drbd部署
date: 2018-11-04 12:12:28
tags:
- mkdoc
---
## pacemeker+iSCSI+drbd部署

### 前期准备

#### hostname

- 设定hostname

```bash
#centos6
vi /etc/sysconfig/network
#centos7
hostnamectl set-hostname node1
hostnamectl set-hostname node2
```
<!--more-->
- hosts解析

```bash
vi /etc/hosts
192.168.8.163 node1
192.168.8.238 node2
```

#### ssh互信


```bash
ssh-keygen
ssh-copy-id
```

#### 时间同步

```bash
yum install -y ntp
ntpdate -u time1.aliyun.com
```

#### 安全相关

- selinux

```bash
setenforce 0   
sed -i.bak "s/SELINUX=enforcing/SELINUX=disable/g" /etc/selinux/config
```

- firewall

```bash
systemctl stop  firewalld
systemctl disable  firewalld
```

### drbd

#### 软件安装

安装需要源elrepo

>elrepo是CentOS十分有用的稳定的软件源，与其他软件源不一样的是，这个第三方源主要是提供硬件驱动、内核更新方面的支持，如显卡、无线网卡、内核等等

```bash
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-2.el7.elrepo.noarch.rpm
yum install -y kmod-drbd84 drbd84-utils

#模块加载
modprobe drbd
#查看模块
lsmod |grep drbd
```


#### 主机磁盘准备

```bash
#node1
[root@node1 ~]# lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sr0     11:0    1 1024M  0 rom
vda    253:0    0   40G  0 disk
├─vda1 253:1    0  500M  0 part /boot
└─vda2 253:2    0 39.5G  0 part /
#使用下面三块盘
vdb    253:16   0  400G  0 disk
├─vdb1 253:17   0   50G  0 part
├─vdb2 253:18   0   50G  0 part
└─vdb3 253:19   0  300G  0 part

#node2
[root@node2 ~]# lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sr0     11:0    1 1024M  0 rom
vda    253:0    0   40G  0 disk
├─vda1 253:1    0  500M  0 part /boot
└─vda2 253:2    0 39.5G  0 part /
#使用下面三块盘
vdb    253:16   0  400G  0 disk
├─vdb1 253:17   0   50G  0 part
├─vdb2 253:18   0   50G  0 part
└─vdb3 253:19   0  300G  0 part

```

#### drbd块准备

```bash
#暂时不用
# mknod /dev/drbd1 b 159 0
# mknod /dev/drbd2 b 159 0
# mknod /dev/drbd3 b 160 0
```

#### drbd配置

- 全局配置

```bash
#vim /etc/drbd.d/global_common.conf

global {
        usage-count no;

}

common {
        protocol C;
        syncer {
                rate 100M;
        }
        net {
                #protocol C;
                cram-hmac-alg "sha1";
                shared-secret "zzidc-cluster12";
        }
        disk {
                on-io-error detach;
        }
}
```

- 资源配置

```bash
#cat /etc/drbd.d/r0.res
resource r0 {
  volume 0 {
    device    /dev/drbd1;
    disk      /dev/vdb1;
    meta-disk internal;
  }
  volume 1 {
    device    /dev/drbd2;
    disk      /dev/vdb2;
    meta-disk internal;
  }
  volume 2 {
    device    /dev/drbd3;
    disk      /dev/vdb3;
    meta-disk internal;
  }
  on node1 {
    address 192.168.8.163:7788;
  }
  on node2 {
    address 192.168.8.238:7788;
  }
}


```

#### drbd资源启动

```bash
#各节点启动资源
drbdadm create-md r0
#该命令会隐式启动drbd服务
drbdadm up r0
#各节点启动drbd服务
systemctl start drbd
systemctl enable drbd
#设定主资源[node1]
drbdadm primary --force r0
#等待同步存储资源
cat /proc/drbd
```

**一定确定各个节点/dev/drbd下面有下面两个文件夹来确保udev设备进行处理，否则reboot节点**

**drbd使用前要格式化**

```bash
drbd
├── by-disk
└── by-res
```

## iSCSI 配置

- target软件安装

```bash
yum install epel-release -y
yum --enablerepo=epel -y install scsi-target-utils

#服务配置开机启动
systemctl start tgtd
systemctl enable tgtd
#tgt的target.conf保持默认就行
```

## pacemake配置

#### pacemaker集群软件安装

- pacemaker和pcs和corosync

pacemaker和corosync将会作为pcs的依赖被安装

```bash
#安装必要包
yum install -y pcs corosync pacemaker

#pacemaker
pacemaker会创建hacluster用户
#启动pcsd服务来管理集群
systemctl start pcsd
systemctl enable pcsd

```

- crmsh可选

```bash
yum install -y python-dateutil PyYAML
#http://download.opensuse.org/repositories/network:/ha-clustering:/Stable/CentOS_CentOS-7/noarch/

#下载
crmsh crmsh-scripts python-parallax

wget -P /etc/yum.repos.d/ http://download.opensuse.org/repositories/network:/ha-clustering:/Stable/CentOS_CentOS-7/network:ha-clustering:Stable.repo

yum makecache

yum install crmsh 
```

#### 创建集群

- 集群用户密码

```bash
echo 123456 | passwd --stdin hacluster
```

- 防火墙

```bash
#开启多播
iptables -A INPUT -p igmp -j ACCEPT
iptables -A INPUT -m addrtype --dst-type MULTICAST -j ACCEPT
#允许端口，tcp，udp
iptables -A INPUT -p udp -m state --state NEW -m multiport --dports 5404,5405 -j ACCEPT
iptables -I INPUT -p tcp -m state --state NEW -m multiport --dports 2224,3121,5403,21064 -j ACCEPT
```

- 节点认证

```bash
pcs cluster auth node1 node2 -u hacluster -p 123456
```

- 创建集群

```bash
#--force会摧毁现有集群进行重建
pcs cluster setup --name iscsi-cluster node1 node2 --force
```

- 启动集群

```bash
#启动所有节点
pcs cluster start --all
#启动单个节点
pcs cluster start node1
#配置集群开机启动
pcs cluster enable all
```

- 查看集群状态

```bash
pcs status
#查看集群报错
crm_verify -L -V
#处理节点
pcs cluster standby node1

```

#### 集群默认参数

```bash
#没有多主去掉
pcs property set stonith-enabled=false
#双节点不需要选举
pcs property set no-quorum-policy=ignore
#增加资源粘性，不会来回切换资源
pcs resource defaults resource-stickiness=100
```

### 创建集群资源

#### 资源创建

- ip

```bash
pcs resource create iscsi-ip ocf:heartbeat:IPaddr2 \
        ip="192.168.8.165" cidr_netmask="24" nic="eth2" \
        op monitor interval="10s"
```

- drbd

```bash
pcs resource create iscsi-drbd ocf:linbit:drbd \
         drbd_resource=r0 op monitor interval=12s

#克隆为主备资源类型
pcs resource master iscsi-data iscsi-drbd \
         master-max=1 master-node-max=1 clone-max=2 clone-node-max=1 \
         notify=true
```

- target

```bash
pcs resource create iscsi-target ocf:heartbeat:iSCSITarget \
          iqn="iqn.storagevol.com:iscsi" tid="1" \
        op monitor interval="10s"

pcs resource create iscsi-lun1 ocf:heartbeat:iSCSILogicalUnit \
          target_iqn="iqn.storagevol.com:iscsi" lun="1" path="/dev/drbd1"

pcs resource create iscsi-lun2 ocf:heartbeat:iSCSILogicalUnit \
          target_iqn="iqn.storagevol.com:iscsi" lun="2" path="/dev/drbd2"

pcs resource create iscsi-lun3 ocf:heartbeat:iSCSILogicalUnit \
          target_iqn="iqn.storagevol.com:iscsi" lun="3" path="/dev/drbd3"

```
- 建组

```bash
pcs resource group group-luns iscsi-target iscsi-lun1 iscsi-lun2 iscsi-lun3
```

#### 更改资源

```bash
pcs resource update nfs-ip cidr_netmask="16"
```

#### 资源限制

```bash

 pcs constraint order luns-after-data inf: iscsi-data:promote group-luns:start

 pcs constraint colocation luns-with-date inf: group-luns iscsi-data:Master

 pcs constraint order ip-after-data inf: iscsi-data:promote iscsi-ip:start

 pcs constraint colocation ip-after-data inf: iscsi-ip iscsi-data:Master
```

### nfs组合

```bash
pcs resource create nfs-ip ocf:heartbeat:IPaddr2 \
        ip="172.16.3.200" cidr_netmask="24" nic="eth1" \
        op monitor interval="10s"
```

- drbd

```bash
pcs resource create nfs-drbd ocf:linbit:drbd \
         drbd_resource=r0 op monitor interval=12s

#克隆为主备资源类型
pcs resource master nfs-data nfs-drbd \
         master-max=1 master-node-max=1 clone-max=2 clone-node-max=1 \
         notify=true
```


- nfs

```bash
pcs resource create nfs-server service:nfs \
  op monitor interval="10s"
  
pcs resource create nfs-exportfs ocf:heartbeat:exportfs \
  clientspec="172.16.0.0/16" \
  directory="/data" \
  fsid="0" \
  options="insecure,rw,async,no_root_squash" \
  op monitor interval="10s" 
```

- file

```bash
pcs resource create nfs-file ocf:heartbeat:Filesystem \
    device=/dev/drbd1 \
    directory=/data \
    fstype=ext4 \
  op monitor interval="10"  

```

```bash  
pcs resource delete 
pcs resource group add nfss nfs-file nfs-server nfs-exportfs 
 
pcs constraint colocation add nfs-ip with nfs-data INFINITY with-rsc-role=Master
pcs constraint colocation add nfss with nfs-data INFINITY with-rsc-role=Master

pcs constraint order promote nfs-data then start nfs-ip  
pcs constraint order promote nfs-data then start nfss  
  
``` 

### 去掉一些限制

```bash
pcs property set stonith-enabled=false
pcs property set no-quorum-policy=ignore
pcs resource defaults resource-stickiness=100
```
### 认为设定主

```bash
#INFINITY可以为数字
pcs constraint location nfs-exportfs prefers Storage-1=50
```

## 参考链接

```bash
https://access.redhat.com/documentation/zh-tw/red_hat_enterprise_linux/6/html-single/configuring_the_red_hat_high_availability_add-on_with_pacemaker/
```
