---
title: kylin系统arm64虚拟化安装
date: 2019-08-20 12:12:28
tags:
- linux
---

# 概述

本文介绍如何在arm宿主机上安装arm虚拟机，主要使用两家国内企业的产品

- [银河麒麟](http://www.kylinos.cn/)
- [飞腾](http://www.phytium.com.cn/)

<!--more-->

# 环境准备

## kylin iso

```bash
# 2000就是飞腾2000的意思
Kylin-4.0.2-server-sp2-2000-19080415.Z1-arm64.iso
```

## 宿主机

### 宿主机cpu

```bash
lscpu

Architecture:          aarch64
Byte Order:            Little Endian
CPU(s):                64
On-line CPU(s) list:   0-63
Thread(s) per core:    1
Core(s) per socket:    4
Socket(s):             16
NUMA node(s):          8
#飞腾2000+
Model name:            Phytium,FT2000PLUS
CPU max MHz:           2200.0000
CPU min MHz:           1000.0000
BogoMIPS:              3600.00
NUMA node0 CPU(s):     0-7
NUMA node1 CPU(s):     8-15
NUMA node2 CPU(s):     16-23
NUMA node3 CPU(s):     24-31
NUMA node4 CPU(s):     32-39
NUMA node5 CPU(s):     40-47
NUMA node6 CPU(s):     48-55
NUMA node7 CPU(s):     56-63
Flags:                 fp asimd evtstrm crc32
```

### 宿主机系统和内核

```bash
#内核
uname -a
Linux Kylin 4.4.131-20190726.kylin.server-generic #kylin SMP Tue Jul 30 16:44:09 CST 2019 aarch64 aarch64 aarch64 GNU/Linux

#系统
cat /etc/.kyinfo

[dist]
name=Kylin
milestone=4.0.2-server-sp2-2000-19080415.Z1
arch=arm64
beta=False
time=2019-08-04 15:37:14
dist_id=Kylin-4.0.2-server-sp2-2000-19080415.Z1-arm64-2019-08-04 15:37:14

[servicekey]
key=0004030

[os]
to=
term=2020-11-09
```

## iso源

qemu以及libvirt包银河麒麟工程师已经做了针对性修改，因此要使用他们发布的包，并且iso内的文件已经包含相关的包

```bash
mount Kylin-4.0.2-server-sp2-2000-19080415.Z1-arm64.iso /mnt
#备份源
cp /etc/apt/sources.list /root/sources.list
#导入本地配置源
echo "deb file:///mnt juniper main" > /etc/apt/sources.list
#更新源
apt-get clean
apt-get update

```

## 基础包安装

## 安装包

```bash
apt-get install qemu-* -y --allow-unauthenticated
apt-get install libvirt* -y --allow-unauthenticated 
apt install virt-manager -y --allow-unauthenticated 
```

### libvirt配置

```bash
# vim /etc/libvirt/qemu.conf


# Some examples of valid values are:
#
#       user = "qemu"   # A user named "qemu"
#       user = "+0"     # Super user (uid=0)
#       user = "100"    # A user named "100" or a user with uid=100
#
user = "root"

# The group for QEMU processes run by the system instance. It can be
# specified in a similar way to user.
group = "root"

# Whether libvirt should dynamically change file ownership
# to match the configured user/group above. Defaults to 1.
# Set to 0 to disable file ownership changes.
dynamic_ownership = 0

systemctl restart libvirtd

```

# 安装虚拟机

## 命令安装

```bash
virt-install \
--connect qemu:///system \
--virt-type kvm \
--name kvm0 \
--disk path=/media/xylink/kylin-4-0-2-arm64.img \
--ram 1024 \
--network bridge=br-wan \
--memballoon model=virtio \
--cdrom /media/Kylin-4.0.2-server-sp2-2000-19080415.Z1-arm64.iso \
--graphics vnc,port=5916,listen='0.0.0.0' \
--autostart --noautoconsole --os-type=linux 
```

## 图形化安装

`virt-manager` 图形化安装即可

### vnc

```bash
# 安装vnc
apt-get install vnc4server

# 开启vnc服务
# 接着输入一下密码就行了
# 端口为590 开头
vnc4server 
```

### 远程连接

```bash
apt-get install ssh-askpass-gnome --no-install-recommends
ln -s /data/vm /var/lib/libvirt/images
```

# 虚拟机配置

## 宿主机网桥

```bash
apt-get install bridge-utils
vi /etc/network/interfaces.d/br-wan

auto br-wan
iface br-wan inet dhcp
        bridge_ports enp11s0f0
        bridge_stp off
        bridge_fd 0
        bridge_maxwait 0

systemctl restart networking
```

[网桥静态配置](https://www.linuxprobe.com/build-bridge-ubuntu.html)

## 虚拟机内开机自启

### dhcp脚本

```bash
ip link set enp1s0 up
dhclient
cp /opt/interface/enp1s0bak /opt/interface/enp1s0
ip=`ifconfig enp1s0 | grep "inet addr:" | awk '{print $2}' | cut -c 6-`
sed -i "s/WANIP/$ip/g"  /opt/interface/enp1s0
cp /opt/interface/enp1s0 /etc/network/interfaces.d/enp1s0
rm -rf enp1s0
systemctl restart networking
```
### 静态IP

```bash
vi /etc/network/interfaces.d/enp1s0

##########动态
auto enp1s0
iface enp1s0 inet dhcp

##########静态
auto enp1s0
iface enp1s0 inet static
address x.x.x.52  
netmask 255.255.x.0
gateway 172.17.1x.254
dns-nameserver 1x.17.x.x

systemctl restart networking
```

## 虚拟机国内源

```bash
vi /etc/apt/sources.list


deb http://mirrors.ustc.edu.cn/ubuntu-ports/ xenial main multiverse restricted universe
deb http://mirrors.ustc.edu.cn/ubuntu-ports/ xenial-backports main multiverse restricted universe
deb http://mirrors.ustc.edu.cn/ubuntu-ports/ xenial-proposed main multiverse restricted universe
deb http://mirrors.ustc.edu.cn/ubuntu-ports/ xenial-security main multiverse restricted universe
deb http://mirrors.ustc.edu.cn/ubuntu-ports/ xenial-updates main multiverse restricted universe
deb-src http://mirrors.ustc.edu.cn/ubuntu-ports/ xenial main multiverse restricted universe
deb-src http://mirrors.ustc.edu.cn/ubuntu-ports/ xenial-backports main multiverse restricted universe
deb-src http://mirrors.ustc.edu.cn/ubuntu-ports/ xenial-proposed main multiverse restricted universe
deb-src http://mirrors.ustc.edu.cn/ubuntu-ports/ xenial-security main multiverse restricted universe
deb-src http://mirrors.ustc.edu.cn/ubuntu-ports/ xenial-updates main multiverse restricted universe
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ xenial main multiverse restricted universe
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ xenial-security main multiverse restricted universe
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ xenial-updates main multiverse restricted universe
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ xenial-backports main multiverse restricted universe
deb-src http://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ xenial main multiverse restricted universe
deb-src http://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ xenial-security main multiverse restricted universe
deb-src http://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ xenial-updates main multiverse restricted universe
deb-src http://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ xenial-backports main multiverse restricted universe
deb http://mirrors.aliyun.com/ubuntu-ports/ xenial main
deb-src http://mirrors.aliyun.com/ubuntu-ports/ xenial main

deb http://mirrors.aliyun.com/ubuntu-ports/ xenial-updates main
deb-src http://mirrors.aliyun.com/ubuntu-ports/ xenial-updates main

deb http://mirrors.aliyun.com/ubuntu-ports/ xenial universe
deb-src http://mirrors.aliyun.com/ubuntu-ports/ xenial universe
deb http://mirrors.aliyun.com/ubuntu-ports/ xenial-updates universe
deb-src http://mirrors.aliyun.com/ubuntu-ports/ xenial-updates universe

deb http://mirrors.aliyun.com/ubuntu-ports/ xenial-security main
deb-src http://mirrors.aliyun.com/ubuntu-ports/ xenial-security main
deb http://mirrors.aliyun.com/ubuntu-ports/ xenial-security universe
deb-src http://mirrors.aliyun.com/ubuntu-ports/ xenial-security universe
deb http://mirrors.ustc.edu.cn/ubuntu-ports/ trusty main multiverse restricted universe
deb http://mirrors.ustc.edu.cn/ubuntu-ports/ trusty-backports main multiverse restricted universe
deb http://mirrors.ustc.edu.cn/ubuntu-ports/ trusty-proposed main multiverse restricted universe
deb http://mirrors.ustc.edu.cn/ubuntu-ports/ trusty-security main multiverse restricted universe
deb http://mirrors.ustc.edu.cn/ubuntu-ports/ trusty-updates main multiverse restricted universe
deb-src http://mirrors.ustc.edu.cn/ubuntu-ports/ trusty main multiverse restricted universe
deb-src http://mirrors.ustc.edu.cn/ubuntu-ports/ trusty-backports main multiverse restricted universe
deb-src http://mirrors.ustc.edu.cn/ubuntu-ports/ trusty-proposed main multiverse restricted universe
deb-src http://mirrors.ustc.edu.cn/ubuntu-ports/ trusty-security main multiverse restricted universe
deb-src http://mirrors.ustc.edu.cn/ubuntu-ports/ trusty-updates main multiverse restricted universe
deb [arch=armhf] https://download.docker.com/linux/ubuntu trusty stable
deb http://mirrors.ustc.edu.cn/ubuntu-ports/ xenial main multiverse restricted universe
deb http://mirrors.ustc.edu.cn/ubuntu-ports/ xenial-backports main multiverse restricted universe
deb http://mirrors.ustc.edu.cn/ubuntu-ports/ xenial-proposed main multiverse restricted universe
deb http://mirrors.ustc.edu.cn/ubuntu-ports/ xenial-security main multiverse restricted universe
deb http://mirrors.ustc.edu.cn/ubuntu-ports/ xenial-updates main multiverse restricted universe
deb-src http://mirrors.ustc.edu.cn/ubuntu-ports/ xenial main multiverse restricted universe
deb-src http://mirrors.ustc.edu.cn/ubuntu-ports/ xenial-backports main multiverse restricted universe
deb-src http://mirrors.ustc.edu.cn/ubuntu-ports/ xenial-proposed main multiverse restricted universe
deb-src http://mirrors.ustc.edu.cn/ubuntu-ports/ xenial-security main multiverse restricted universe
deb-src http://mirrors.ustc.edu.cn/ubuntu-ports/ xenial-updates main multiverse restricted universe
```





