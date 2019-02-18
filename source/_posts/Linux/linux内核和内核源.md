---
title: linux内核和内核源
date: 2018-11-04 12:12:28
tags:
- linux
---
## linux内核和内核源

### ELRepo项目

专注于内核和内核模块的源，rpm包

<!--more-->
### 使用

#### 源配置

```bash
#导入key
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org

#centos7
rpm -Uvh https://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm
#cenos6
rpm -Uvh https://www.elrepo.org/elrepo-release-6-8.el6.elrepo.noarch.rpm
```

#### 安装

最新版内核安装

```bash

yum --enablerepo=elrepo-kernel install kernel-ml kernel-ml-devel
#也可以网站上直接下载特定版本
http://elrepo.org/linux/kernel/el7/x86_64/RPMS/
#仅仅安装这个源的包
yum --disablerepo=\* --enablerepo=elrepo install kmod-nvidia

```

内核模块安装

```bash
yum install kmod-r8168
```

## 内核默认启动

### 查看启动项目

```bash
#有序号
awk -F\' '$1=="menuentry " {print i++ " : " $2}' /etc/grub2.cfg
#或者没有序号
grep "^menuentry" /boot/grub2/grub.cfg | cut -d "'" -f2
#查看默认启动项目
grub2-editenv list
```

### 设定默认启动内核

```bash
#centos6
sed -i 's/^default=.*/default=0/g' /boot/grub/grub.conf
#centos7 /boot/grub2/grub.cfg
grub2-set-default 0
```

### 开启bbr

```bash
sed -i '/net.core.default_qdisc/d' /etc/sysctl.conf
sed -i '/net.ipv4.tcp_congestion_control/d' /etc/sysctl.conf
echo "net.core.default_qdisc = fq" >> /etc/sysctl.conf
echo "net.ipv4.tcp_congestion_control = bbr" >> /etc/sysctl.conf
sysctl -p >/dev/null 2>&1
```