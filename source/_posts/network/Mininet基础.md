---
title: Mininet基础
tags:
- openflow
---

Mininet是一个sdn模拟平台，可以模拟交换机和主机，使用Linux中namespace来实现相关功能。

# Mininet安装

拉取git

```bash
git clone git://github.com/mininet/mininet
```

<!--more-->

## 安装脚本修改

默认Mininet安装脚本并不支持centos发行版，需要对安装脚本做些改动

```bash
vim util/install.sh

#在if [ "$ARCH" = "i686" ]; then ARCH="i386"; fi下面添加

test -e /etc/centos-release && DIST="CentOS"
if [ "$DIST" = "CentOS" ]; then
    install='sudo yum -y install'
    remove='sudo yum -y erase'
    pkginst='sudo rpm -ivh'
    # Prereqs for this script
    if ! which lsb_release &> /dev/null; then
        $install redhat-lsb-core
    fi
fi

#修改行DISTS='Ubuntu|Debian|Fedora|RedHatEnterpriseServer|SUSE LINUX'
DISTS='Ubuntu|Debian|Fedora|RedHatEnterpriseServer|SUSE|CentOS LINUX'

#如果还不行就全局替换Fedora为CentOS
```

## 安装

```bash
#查看脚本使用帮助
bash -x /util/install.sh -h
#安装mininet和openflow实现
bash -x /util/install.sh -nf
```

# openvswitch安装

mininet安装脚本可以安装ovs，但是我们自己安装最新版ovs，编译好的rpm包x64[下载](https://github.com/sunnoy/books_note_res)

```bash
#编译工具和依赖安装
yum -y install gcc make python-devel openssl-devel kernel-devel graphviz \
kernel-debug-devel autoconf automake rpm-build redhat-rpm-config \
libtool wget checkpolicy selinux-policy-devel python-sphinx unbound-devel

#到此下载
http://www.openvswitch.org/download/

#编译rpm包安装
mkdir -p ~/rpmbuild/SOURCES/
cd ~/rpmbuild/SOURCES/
wget https://www.openvswitch.org/releases/openvswitch-2.10.1.tar.gz
tar -xvf openvswitch-2.10.1.tar.gz && cd openvswitch-2.10.1
rpmbuild -bb --without check rhel/openvswitch.spec
rpm -ivh --nodeps ~/rpmbuild/RPMS/x86_64/openvswitch*.rpm

#启动
systemctl start openvswitch
```

或使用包安装

```bash
[leifmadsen-ovs-master]
name=Copr
baseurl=https://copr-be.cloud.fedoraproject.org/results/leifmadsen/ovs-master/epel-7-$basearch/
type=rpm-md
skip_if_unavailable=True
gpgcheck=1
gpgkey=https://copr-be.cloud.fedoraproject.org/results/leifmadsen/ovs-master/pubkey.gpg
repo_gpgcheck=0
enabled=1
enabled_metadata=1

yum install openvswitch openvswitch-ovn-* -y
```

# 基本使用

## 默认配置

默认执行将会创建h1 h2两个host和一个交换机s1

```bash
#直接使用，就会进入交互界面
[root@node1 ~]# mn
*** Creating network
*** Adding controller
*** Adding hosts:
h1 h2
*** Adding switches:
s1
*** Adding links:
(h1, s1) (h2, s1)
*** Configuring hosts
h1 h2
*** Starting controller
c0
*** Starting 1 switches
s1 ...
*** Starting CLI:
mininet>
```

如果遇到启动卡住，试着运行`mnexec -n ifconfig -a`，可能是内存少了，执行`echo 1 > /proc/sys/vm/drop_caches`释放一些内存

## 交互模式常用命令

```bash
#显示帮助
help
#显示多有节点
nodes
#执行ovs命令，替代ovs-ofctl
dpctl
#显示拓扑信息
dump
#链路查看
links
#链路操作
link
#接口关系
net
#节点操作 node command args
h1 ifconfig -a
h1 ping h2
```

## 连接远程控制器

```bash
mn --controller=remote,ip=[controller IP],port=[controller listening port]
```






