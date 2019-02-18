---
title: ovs架构介绍
date: 2018-12-01 12:14:28
tags:
- openflow
---

# ovs架构介绍

ovs在虚拟化中起着越来越重要的角色，是因为在宿主和虚拟机之间提供了很好的自定义网络形式

ovs在虚拟化架构中的位置

![ovsall](https://qiniu.li-rui.top/ovsall.png)

<!--more-->

## ovs主要组件

ovs主要三部分组成，ovsdb-sever、ovs-vswitchd、ovs kernel module

- ovsdb-server ovs的数据库服务，用来存储ovs的配置信息
- ovs-vswitchd ovs核心部件，它和上层controller通信遵从OPENFLOW协议，它与ovsdb-server通信使用OVSDB协议，它和内核模块通过netlink通信
- ovs kernel module ovs内核模块，如果在内核的缓存中找到转发规则就转发，否则发向用户空间去处理。查看模块`lsmod | grep openvswitch`

![ovszjjg](https://qiniu.li-rui.top/ovszjjg.png)

作为ovs中的核心组件，ovs-vswitchd通过ovsdb-server将配置持久化，ovs-vsctl直接操作ovsdb-server来改变配置，是较常用的命令。ovs-appctl则与ovs-vswitchd来通信来更改配置

### 和openflow对比

- openflow是协议包含控制器，openflow协议和openflow交换机
- ovs是openflow交换机和openflow协议的实现
- ovs除了支持openflow协议进行远程控制支持OVSDB协议
- ovs可以不用远程的openflow控制器来操作，一般使用命令`ovs-vsctl`，该命令也是使用openflow协议来操作

## 包流向

![openvswitch-arch](https://qiniu.li-rui.top/openvswitch-arch.png)

### 发送

当数据包从某块虚拟网卡到达内核模块ovs kernel module后，内核模块会首先查看这个包所在的流在内核空间是否存在缓存，如果存在就缓存来走，么有就通过generic netlink交给用户空间的ovs-vswitchd处理

用户空间的ovs-vswitchd收到内核传过来的包以后就去查找ovsdb-server看看这个包的目的端口是谁，接着就把信息返回给内核模块ovs kernel module

ovs kernel module将会把包发送到用户设置的目的端口

### 接收

接收过程与发送过程大体一样，ovs会在端口相连的外部设备存放一个钩子，一旦这些设备接收到了数据包，ovs就会将他们发送到用户空间去判断对包需要执行的动作和发往的目的地

## 基本概念

### packet

网络数据包是网络转发的最小数据单元，数据包总是从一个端口被转发到一个多个目的端口

### Bridge

网桥就是交换机，主要作用就是将一个端口的数据包转发到其他的端口

### Port

端口是用来收发数据包的基本单元，一个端口总是在网桥上面。ovs中的端口依据作用分为多种类型

- Normal 普通的收发端口，可以是机器上的一块网卡
- Internal ovs会创建一块虚拟网卡，端口收到的所有数据包都会交给该网卡，发出的包会通过该端口交给ovs。当ovs创建一个新网桥时，默认会创建一个与网桥同名的Internal Port
- Patch 用于交换机之间相连的端口
- Tunnel 隧道端口，用于VXLAN和GRE隧道使用

### Interface

接口就是ovs用来和外部交换数据的组件，接口的数据将会发往端口。可能是Open vSwitch生成的虚拟网卡，也可能是物理网卡挂载在Open vSwitch上，也可能是操作系统的虚拟网卡（TUN/TAP）挂载在Open vSwitch上

### Flow 

流就是端口之间数据包的交换规则，规则分为两部分

- 匹配 这个流规则将会处理那些包
- 动作 匹配命中的包会进行什么样的动作

一个复杂的流

```bash
#匹配
当数据包来自端口A，并且其源MAC是aa:aa:aa:aa:aa:aa，并且其拥有vlan tag为a，并且其源IP是a.a.a.a，并且其协议是TCP，其TCP源端口号为a，
#动作
则修改其源IP为b.b.b.b，发往端口B
```

### Datapath

当流多了匹配起来就会非常耗时，降低效率。datapath就是内核中的流的缓存，把流的结果缓存起来下来包直接转发走。datapath完全内核中实现，并且缓存时间比较短，3秒左右。









