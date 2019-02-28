---
title: 隧道技术和mtu
date: 2018-12-19 12:12:28
tags:
- net
---

# 隧道技术与mtu

## 隧道技术

在网络中隧道就是用一种协议作为另外一种协议的链路层，常见的隧道技术有：

- 各种vpn
- vxlan
- gre
- stt
- Geneve

<!--more-->

## mtu

### 以太网mtu

mtu就是Maximum Transmission Unit，最大传输单元。在以太网中mtu表示以太网帧的数据载荷，也就是说当mtu为1500字节时，以太网帧大小=数据载荷1500+以太网头部14(目标网卡地址6+源网卡地址6+类型2，其中crc 4一般没有)，也就是1514字节

![以太网](https://qiniu.li-rui.top/以太网.png)

## 分片

### 分片和报文

当一个包的大小大于链路层(以太网)的mtu时，就会产生分片。分片在IP层和tcp层都有实现，对于icmp包分片是依赖于IP层的。

ip报文如下：

```bash
#IP头部为20字节
    0                   1                   2                   3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |Version|  IHL  |Type of Service|          Total Length         |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |         Identification        |Flags|      Fragment Offset    |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |  Time to Live |    Protocol   |         Header Checksum       |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                       Source Address                          |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                    Destination Address                        |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                    Options                    |    Padding    |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

Flags

           0     1      2
        +-----+------+------+
        |  0  |  DF  |  MF  |
        +-----+------+------+

```

ip层分片主要依赖报文中Fragment Offset和Flags字段中的标志位MF

Bit 0: 保留位，必须为0。
Bit 1: DF（Don't Fragment），能否分片位，0表示可以分片，1表示不能分片。
Bit 2: MF（More Fragment），表示是否该报文为最后一片，0表示最后一片，1代表后面还有。

分片报文有三类

- 第一片报文 Fragment Offset=0且MF=1

- 中间片报文 Fragment Offset和MF都非0

- 最后一片报文 MF=0且Fragment Offset不为0

分片tcpdump抓包，对于分片包id均为61197，表明这来自同一个报文包并且进行了分片

```bash
#从192.168.6.231 ping 192.168.6.230

#icmp请求包没有分片直接传送
18:27:08.955743 IP (tos 0x0, ttl 64, id 50649, offset 0, flags [none], proto ICMP (1), length 5828)
    192.168.6.231 > 192.168.6.230: ICMP echo request, id 31456, seq 1, length 5808

#icmp回应包进行了分片，ip包标识符一致 20126
18:27:08.955813 IP (tos 0x0, ttl 64, id 20126, offset 0, flags [+], proto ICMP (1), length 1500)
    192.168.6.230 > 192.168.6.231: ICMP echo reply, id 31456, seq 1, length 1480

#icmp分片包没有icmp头部
#第一片
18:27:08.955821 IP (tos 0x0, ttl 64, id 20126, offset 1480, flags [+], proto ICMP (1), length 1500)
    192.168.6.230 > 192.168.6.231: ip-proto-1

#第二片
18:27:08.955824 IP (tos 0x0, ttl 64, id 20126, offset 2960, flags [+], proto ICMP (1), length 1500)
    192.168.6.230 > 192.168.6.231: ip-proto-1

#第三片
18:27:08.955827 IP (tos 0x0, ttl 64, id 20126, offset 4440, flags [none], proto ICMP (1), length 1388)
    192.168.6.230 > 192.168.6.231: ip-proto-1

```

### 分片类型

#### 设备分片

早期的分片操作是在操作系统层完成的，随着网卡offload特性的加入，一些在操作系统层的数据包操作就可以在网卡上完成，分片就是其中一种。因此分片分为两种

- 操作系统分片
- 网卡分片

### 路径中分片

一个数据包进行分片，可能发生在路径中的任何一个设备，除了目的端设备。目的端设备负责重组。我们知道数据包是在路径中的设备之间一跳一跳的发送数据包。如果路径中的一个路由器mtu很小，即使发送端没有分片，到了这个路由器也会分片。

### Path MTU Discovery

如果发送主机在一个包内设置了不允许IP分片，即DF为1，那么当这个包到达一个mtu比较小的设备时就会直接丢弃，并发送一个icmp信息，对ipv4是：Fragmentation Needed (Type 3, Code 4) ，对ipv6是：Packet Too Big (Type 2)，并附上该设备的mtu值到发送端，发送端看到后再去调整发送数据包的大小。如果设置了DF位但是没有收到任何回应，应该是icmp回应消息被防火墙拦截了。

# 各种隧道中的头部

## vxlan

VxLAN 的 overhead 是1514- 1464 = 50 byte。

![vxlna](https://qiniu.li-rui.top/vxlna.png)

## gre

GRE 的 overhead 是 1514 - 1490 = 24 byte

![gre](https://qiniu.li-rui.top/gre.png)

## geneve

geneve 的 overhead 是 1514 - 1490 = 50 byte

![geneve](https://qiniu.li-rui.top/geneve.png)



