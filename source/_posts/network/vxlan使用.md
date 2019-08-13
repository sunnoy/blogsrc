---
title: vxlan使用
date: 2019-08-12 12:12:28
tags:
- net
- docker
---

# vxlan概述

vxlan是一种隧道协议，用来形成二层网络

<!--more-->

# 点对点vxlan创建

## 创建一个vxlan

### 创建vxlan接口

```bash
# 帮助
ip link add type vxlan help
Usage: ... vxlan id VNI [ { group | remote } IP_ADDRESS ] [ local ADDR ]
                 [ ttl TTL ] [ tos TOS ] [ dev PHYS_DEV ]
                 [ dstport PORT ] [ srcport MIN MAX ]
                 [ [no]learning ] [ [no]proxy ] [ [no]rsc ]
                 [ [no]l2miss ] [ [no]l3miss ]
                 [ ageing SECONDS ] [ maxaddress NUMBER ]
                 [ [no]udpcsum ] [ [no]udp6zerocsumtx ] [ [no]udp6zerocsumrx ]
                 [ gbp ]

Where: VNI := 0-16777215
       ADDR := { IP_ADDRESS | any }
       TOS  := { NUMBER | inherit }
       TTL  := { 1..255 | inherit }

#添加
ip link add vxlan0 type vxlan \
    id 42 \
    dstport 4789 \
    remote 172.18.247.202 \
    local 172.18.247.201

#查看
ip -d link show vxlan0

17: vxlan0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT qlen 1000
    link/ether 5a:6f:fa:87:9f:ef brd ff:ff:ff:ff:ff:ff promiscuity 0
    vxlan id 42 remote 172.18.247.202 local 172.18.247.201 srcport 0 0 dstport 4789 ageing 300 addrgenmode eui64

# 添加IP
ip addr add 10.0.0.2/24 dev vxlan0
ip link set vxlan0 up
```

通过创建接口-配置IP-启用接口后系统的路由表和转发表将会新增

```bash
#路由表
ip route

10.0.0.0/24 dev vxlan0 proto kernel scope link src 10.0.0.2

# 转发表
bridge fdb
#默认的des都是 202
00:00:00:00:00:00 dev vxlan0 dst 172.18.247.202 self permanent

#同时监听udp 4789
netstat -nau | grep 4789
udp        0      0 0.0.0.0:4789            0.0.0.0:*
```

## 创建另一个vxlan

### 创建

```bash
#添加
ip link add vxlan0 type vxlan \
    id 42 \
    dstport 4789 \
    remote 172.18.247.201 \
    local 172.18.247.202

# 添加IP
ip addr add 10.0.0.3/24 dev vxlan0
ip link set vxlan0 up
```

### 测试

```bash
ping 10.0.0.2
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
64 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=0.476 ms
64 bytes from 10.0.0.2: icmp_seq=2 ttl=64 time=0.269 ms
64 bytes from 10.0.0.2: icmp_seq=3 ttl=64 time=0.255 ms
64 bytes from 10.0.0.2: icmp_seq=4 ttl=64 time=0.263 ms
```