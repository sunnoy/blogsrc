---
title: kickstart文件语法
date: 2018-11-04 12:12:28
tags:
- cobbler
---

## 语法

### 章节顺序

主要的章节有顺序，章节内的指令可以没有顺序，不需要的指令可以忽略

主要章节顺序：
- Command和%addon 
- %packages
- %pre 和 %post

**带有%的部分必须以%end结尾**

<!--more-->

### 格式检查

需要安装pykickstart包

```bash
yum install pykickstart -y
ksvalidator /path/to/kickstart.ks
```

### 相关指令

#### bootloader必须

```bash
bootloader --location=mbr --append="hdd=ide-scsi ide=nodma"

#append用来追加内核参数
```

#### clearpart

删除系统现有分区

```bash
#指定从那几个删除分区
clearpart --drives=hda,hdb --all
#指定要清理的分区
clearpart --list=sda2,sda3,sdb1
#默认不删除分区
--none
#删除所有分区
clearpart --all --initlabel
```

#### firewall

关闭防火墙

```bash
firewall --disabled
```

#### url

```bash
url --url="http://173.16.0.11/cobbler/ks_mirror/CentOS-7.2-x86_64/"
```

#### keyboard必须

```bash
keyboard --vckeymap=us --xlayouts='us'
```

#### lang必须

```bash
#添加语言支持
lang en_US --addsupport=zh_CN
```

#### network

为目标系统指定网络配置

```bash
network --bootproto=static --ip=22.0.2.15 --netmask=255.255.255.0 --gateway=22.0.2.254 --nameserver=192.168.2.1 --nameserver=192.168.3.1 --device=em1
```

#### part或者partition必须

```bash
part /boot --asprimary --fstype="xfs" --size=1000
part swap --fstype="swap" --size=8000
part / --fstype="xfs" --grow --size=1
```

#### reboot

安装完成后重启系统

#### yum

配置yum源

```bash
repo --name=repoid [--baseurl=<url>|--mirrorlist=url] [options]
```

#### rootpw必须

```bash
#使用python加密
python -c 'import crypt; print(crypt.crypt("My Password"))'
rootpw --iscrypted $1$hgfvQffN$tXNj5mQldgQt4ziW1QhNF0
```

#### selinux

```bash
selinux --disabled
```

#### services

```bash
services --disabled=firewalld,sshd
```

#### 安装模式

```bash
graphical #图形
text #文本

```

#### timezone必须

```bash
timezone Asia/Shanghai --ntpservers=3.centos.pool.ntp.org,0.centos.pool.ntp.org
```

#### %include调用其他的ks文件

```bash
%include /usr/share/anaconda/interactive-defaults.ks
```

#### 软件包的选择

```bash
%packages
#组
@X Window System
@Desktop
@Sound and Video
#包
aspell
docbook*
#排除组包
-@Graphical Internet
-autofs
-ipa*fonts

%end
```

#### %pre

```bash
%pre --interpreter=/usr/bin/python  --log=/mnt/sysimage/root/ks-pre.log#/usr/bin/bash
--- Python script omitted --
%end
```

#### %post

类比于%pre

#### 


### 示例ks

```bash
#platform=x86, AMD64, or Intel EM64T
# System authorization information
auth  --useshadow  --enablemd5
# System bootloader configuration
bootloader --location=mbr
# Partition clearing information
clearpart --all --drives=sda --initlabel
# Use text mode install
text
# Firewall configuration
firewall --disabled
# Run the Setup Agent on first boot
firstboot --disable
# System keyboard
keyboard us
# System language
lang en_US --addsupport=zh_CN
# Use network installation
url --url=$tree
# If any cobbler repo definitions were referenced in the kickstart profile, include them here.
$yum_repo_stanza
# Network information
$SNIPPET('network_config')
# Reboot after installation
reboot

#Root password
rootpw --plaintext 123456
# SELinux configuration
selinux --disabled
# Do not configure the X Window System
skipx
# System timezone
timezone Asia/Shanghai
# Install OS instead of upgrade
install
# Clear the Master Boot Record
zerombr
# Allow anaconda to partition the system as needed
ignoredisk --only-use=sda

part /boot --fstype="xfs" --size=500 --ondisk=sda
part swap --fstype="swap" --size=16384  --ondisk=sda
part pv.0082 --size=1 --grow    --ondisk=sda
volgroup vg_system pv.0082
logvol / --fstype=ext4 --size=102400 --name=lv_root --vgname=vg_system
logvol /home --fstype=ext4 --size=51200 --name=lv_home --vgname=vg_system
logvol /var --fstype=ext4 --size=10 --grow --name=lv_var --vgname=vg_system


%pre
$SNIPPET('log_ks_pre')
$SNIPPET('kickstart_start')
$SNIPPET('pre_install_network_config')
# Enable installation monitoring
$SNIPPET('pre_anamon')
%end

%packages
$SNIPPET('func_install_if_enabled')
@^minimal
@core
%end

%post --nochroot
$SNIPPET('log_ks_post_nochroot')
%end

%post

#yum repo

rm -rf /etc/yum.repos.d/CentOS-Base.repo
curl -L  http://173.16.1.2/repo/base.repo -o /etc/yum.repos.d/base.repo
yum -y install epel-release
rm -rf /etc/yum.repos.d/epel*
curl -L  http://173.16.1.2/repo/epel.repo -o /etc/yum.repos.d/epel.repo


# kernel install
curl -L  http://173.16.1.2/kernel/kernel-3.18.16-1.x86_64.rpm -o /tmp/kernel-3.18.16-1.x86_64.rpm
curl -L  http://173.16.1.2/kernel/kernel-devel-3.18.16-1.x86_64.rpm -o /tmp/kernel-devel-3.18.16-1.x86_64.rpm
rpm -i /tmp/kernel-3.18.16-1.x86_64.rpm
rpm -i /tmp/kernel-devel-3.18.16-1.x86_64.rpm
grub2-set-default 'CentOS Linux (3.18.16) 7 (Core)'

# some post things
$SNIPPET('add_host_ssh_keys')
$SNIPPET('adjustment_kernel')

$SNIPPET('log_ks_post')
# Start yum configuration
$yum_config_stanza
# End yum configuration
$SNIPPET('post_install_kernel_options')
$SNIPPET('post_install_network_config')
$SNIPPET('func_register_if_enabled')
$SNIPPET('download_config_files')
$SNIPPET('koan_environment')
$SNIPPET('redhat_register')
$SNIPPET('cobbler_register')
# Enable post-install boot notification
$SNIPPET('post_anamon')
# Start final steps
$SNIPPET('kickstart_done')
# End final steps
%end

```