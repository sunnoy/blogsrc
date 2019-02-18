---
title: ovn使用
date: 2018-12-14 12:12:28
tags:
- openflow
---

# 安装与启动

## rpm安装

```bash
wget -o /etc/yum.repos.d/ovs-master.repo https://copr.fedorainfracloud.org/coprs/leifmadsen/ovs-master/repo/epel-7/leifmadsen-ovs-master-epel-7.repo
yum install openvswitch openvswitch-ovn-*
```

<!--more-->

## 启动

ovn-northd进程会控制northbound db和southbound db连接信息

```bash
systemctl enable ovn-northd openvswitch ovn-controller
systemctl start ovn-northd openvswitch ovn-controller
```

# Central Controller节点

hostname为node1，IP：11.11.11.12

## 指定连接信息

连接方式为

- ptcp:port:ip 指定监听IP，客户端只能连接这个IP
- tcp:ip:port 所在主机IP，客户端都可以连接

```bash

#指定north进程
ovn-nbctl set-connection ptcp:6641:11.11.11.12
#指定north进程
ovn-sbctl set-connection ptcp:6642:11.11.11.12

#查看监听端口
[root@node1 ~]# netstat -lntp | grep  664
tcp        0      0 11.11.11.12:6641        0.0.0.0:*               LISTEN      9007/ovsdb-server
tcp        0      0 11.11.11.12:6642        0.0.0.0:*               LISTEN      9015/ovsdb-server

```
# ovn-controller节点

ovn-controller节点hostname：ansible，IP：11.11.11.11

## 创建ovn专用br-int

```bash
ovs-vsctl add-br br-int -- set Bridge br-int fail-mode=secure
```

## ovn-controller连接southbound db

ovs-vsctl命令set为设定数据库中某个表下面某条记录中的某一个字段值

open为openvswitch表的缩写，`.`为该表中的记录简写，这个表中只有一条记录简写为`.`

```bash
ovs-vsctl set open . external-ids:ovn-remote=tcp:11.11.11.12:6642
ovs-vsctl set open . external-ids:ovn-encap-type=geneve
#隧道封装IP为本地和remote相连同子网IP
ovs-vsctl set open . external-ids:ovn-encap-ip=11.11.11.11
```
# l2转发

![ovnl2](https://qiniu.li-rui.top/ovnl2.png)

## 创建交换机和端口

网络拓扑创建使用命令`ovn-nbctl`，下面创建一个交换机ls1，在该交换机上创建两个端口分别为ls1-vm1和ls1-vm2，分别对两个端口开启端口安全


```bash
# 创建交换机
ovn-nbctl ls-add ls1

# 创建交换机端口
ovn-nbctl lsp-add ls1 ls1-vm1
ovn-nbctl lsp-set-addresses ls1-vm1 02:ac:10:ff:00:11
#只接受本机地址的包，防止泛洪
ovn-nbctl lsp-set-port-security ls1-vm1 02:ac:10:ff:00:11

# create logical port
ovn-nbctl lsp-add ls1 ls1-vm2
ovn-nbctl lsp-set-addresses ls1-vm2 02:ac:10:ff:00:22
ovn-nbctl lsp-set-port-security ls1-vm2 02:ac:10:ff:00:22

#查看拓扑
[root@node1 ~]# ovn-nbctl show
switch 694fbbd3-de45-4646-9394-f6eaed9d4880 (ls1)
    port ls1-vm2
        addresses: ["02:ac:10:ff:00:22"]
    port ls1-vm1
        addresses: ["02:ac:10:ff:00:11"]

```

## 创建vm

这里以网络namespace来代替虚拟机，分别在两个节一个虚拟机

ansible

```bash
ip netns add vm1
ovs-vsctl add-port br-int vm1 -- set interface vm1 type=internal
ip link set vm1 netns vm1
ip netns exec vm1 ip link set vm1 address 02:ac:10:ff:00:11
ip netns exec vm1 ip addr add 172.16.255.11/24 dev vm1
ip netns exec vm1 ip link set vm1 up
ovs-vsctl set Interface vm1 external_ids:iface-id=ls1-vm1

ip netns exec vm1 ip addr show
```

node1

```bash
ip netns add vm2
ovs-vsctl add-port br-int vm2 -- set interface vm2 type=internal
ip link set vm2 netns vm2
ip netns exec vm2 ip link set vm2 address 02:ac:10:ff:00:22
ip netns exec vm2 ip addr add 172.16.255.22/24 dev vm2
ip netns exec vm2 ip link set vm2 up
ovs-vsctl set Interface vm2 external_ids:iface-id=ls1-vm2

ip netns exec vm2 ip addr show
```

# 抓包分析

## arp

可以看到使用了geneve隧道封装，geneve使用udp端口6081。隧道内包含以太网帧

![arp](https://qiniu.li-rui.top/arp.png)

## icpm

![icmp](https://qiniu.li-rui.top/icmp.png)

# 参考资料

[Dustin Spinhirne 的科技博客](http://blog.spinhirne.com/2016/09/a-primer-on-ovn.html)