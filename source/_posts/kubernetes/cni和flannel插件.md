---
title: cni和flannel插件
date: 2019-08-13 12:12:28
tags:
- kubernetes
---

# cni概述

[Container Network Interface](https://github.com/containernetworking/cni) 是cncf的项目，是一个编写容器网络插件的依赖库以及相关的标准

cni本质上使用的docker bridge网络，一般是cni0

<!--more-->

# 插件

插件一般分为下面几大类

## 接口创建

- bridge 创建网桥
- ipvlan 在容器中创建ipvlan接口
- loopback 配置回环接口状态
- macvlan 配置macvlan接口
- ptp 创建 veth对接口
- vlan 创建vlan接口
- host-device 将存在的接口放入容器中

## windows

- win-bridge 创建接口，连接主机和容器
- win-overlay 创建overlay网络接口

## IPAM

- dhcp 在host上运行dhcp服务器为host上的容器分配IP
- host-local 在本地维护一个IP数据库去分配IP地址
- static 分配一个用于debug的静态IP

## 其他插件

- flannel 根据flannel config file来创建接口
- tuning 对一个接口调节 sysctl 参数
- portmap 基于iptables，做一个host port到容器por的映射
- bandwidth 进行带宽限制
- sbr 配置基于源地址的路由
- firewall 以iptables为基础为容器做防火墙配置

# cni配置

cni的配置默认在/etc/cni/net.d，该配置两个地方要使用

- 容器创建系统 如kubelet
- 网络插件 如flannel

```json
// cat /etc/cni/net.d/10-flannel.conflist
{
  "name": "cbr0",
  "plugins": [
    {
      "type": "flannel",
      "delegate": {
        "hairpinMode": true,
        "isDefaultGateway": true
      }
    },
    {
      "type": "portmap",
      "capabilities": {
        "portMappings": true
      }
    }
  ]
}
```

# flannel插件

## 核心概念

- network 负责网络的管理（以后的方向是多网络模型），根据每个网络的配置调用 subnet；
- subnet 负责和 etcd 交互，把 etcd 中的信息转换为 flannel 的子网数据结构，并对 etcd 进行子网和网络的监听；
- backend 接受 subnet 的监听事件，并做出对应的处理。

## backend

### vxlan

vxlan是一种基于udp的隧道封装协议

主要过程

- 创建一个vxlan接口，flannel.1
- 创建该host上子网的路由表以及下一跳
- 为vxlan创建静态fdb

同一个子网就不进行隧道封装就是直接路由

### host-gw

前提是node在二层网络，需要大量的路由表

## 配置文件

```json
{
    "Network": "{{ CLUSTER_CIDR }}",
    "Backend": {
    "DirectRouting": true,
    "Type": "vxlan"
    }
}
```



# 总流程



flannel CNI插件首先读取netconf配置和subnet.env信息，生成适用于bridge CNI插件的netconf文件和ipam（host-local）配置，并设置其delegate为bridge CNI插件。然后调用走bridge CNI插件挂载容器到bridge的流程。由于各个Pod的IP地址分配是基于host-local的Ipam，因此整个流程完全是分布式的，不会对API-Server造成太大的负担

# 参考

```bash
https://www.cnblogs.com/aguncn/p/10548009.html
https://www.cnblogs.com/linuxk/p/10517055.html#vxlan%E5%90%8E%E7%AB%AF%E5%92%8Cdirect-routing
https://segmentfault.com/a/1190000016304924
```

