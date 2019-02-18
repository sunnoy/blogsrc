---
title: Linux中vlan使用
date: 2018-12-24 12:12:28
tags:
- net
---

# vlan介绍

vlan就是虚拟机局域网virtual LAN。一种在二层上隔离广播域的协议

<!--more-->

## 报文解析

这里以以太网II来说明：

```bash
                      #vlan字段
                      +-----------+----------+--------+-----------+
                      |   TPID    |   PRI    |  CFI   |    VID    |
                      |  2 Bytes  |  3 Bits  | 1 Bits |  12 Bits  |
                      +-----------+----------+--------+-----------+
                      |                                           |
                           |                                   |
                               |                          |
                                    |              |
            +-----------+-----------+--------------+---------------+------+-----------+----------+
            |   DMAC    |   SMAC    |  802.1Q Tag  |  Length/Type  |       Data       |   FCS    |
            |  6 Bytes  |  6 Bytes  |   4 Bytes    |    2 Bytes    |  Variable length | 4 Bytes  |
            +-----------+-----------+--------------+---------------+------+-----------+----------+
            |                                                                                    |
                         |                                                                     |
                                          |                                                 |
                                                                |                      |
            +--------------------+-----------------+-------------+---------------------+
            |        帧间隙      |      前同步码   |帧开始定界符 |    Ethernet Frame   |
            |    至少12Bytes     |    7 Bytes      |   1 Byte    |  Variable length    |
            +--------------------+-----------------+-------------+---------------------+
```

以太网报文中802.1Q Tag为可选，当有vlan存在时就会填充相应的字段

在vlan字段中，我们进行抓包

```bash
   VLAN tag: VLAN=10, Priority=Best Effort (default)
         #TPID Tag Protocol Identifier 表示帧类型。0x8100时表示802.1Q Tag
         Identifier: 802.1Q Virtual LAN (0x8100)
         #PRI（Priority）表示帧的QoS优先级，取值范围为0～7，值越大优先级越高，用于阻塞控制，3位
         000. .... .... .... = Priority: Best Effort (default) (0)
         #CFI (Canonical Format Indicator，标准格式指示)，长度为1比特，表示MAC地址是否是标准格式，0为标准，1位
         ...0 .... .... .... = CFI: Canonical (0)
         #VID（VLAN ID），长度为12比特，表示该帧所属的VLAN
         .... 0000 0000 1010 = VLAN: 10
```

vlan字段VID中有三种VID类型：

- Untagged帧：VID不计
- Priority-tagged帧：VID为0x000
- VLAN-tagged帧：VID范围0～4095

三个特殊的VID：

- 0x000：设置优先级但无VID
- 0x001：缺省VID
- 0xFFF：预留VID

# 交换机端口类型

## access

access口中，包进来打上vlan标签，包从端口出去则剥离vlan标签

## trunk

trunk口用于交换机之间的连接，包从trunk口出的时候查看是否有vlan标签，没有就打上默认vlan。包进到trunk口查看该vlan是否允许，允许就通过，否则就丢弃

# 网卡模式

网卡是直接和二层相关的设备，数据报文的传递是通过mac地址实现的，根据发送或接收包的mac地址策略，网卡工作在不同的模式

- 广播模式（Broad Cast Model）:它的物理地址（MAC）地址是 0Xffffff 的帧为广播帧，工作在广播模式的网卡接收广播帧。
- 多播传送（MultiCast Model）：多播传送地址作为目的物理地址的帧可以被组内的其它主机同时接收，而组外主机却接收不到。但是，如果将网卡设置为多播传送模式，它可以接收所有的多播传送帧，而不论它是不是组内成员。
- 直接模式（Direct Model）:工作在直接模式下的网卡只接收目地址是自己 Mac地址的帧。
- 混杂模式（Promiscuous Model）:工作在混杂模式下的网卡接收所有的流过网卡的帧，信包捕获程序就是在这种模式下运行的。

网卡混杂模式配置

```bash
ip link set dev promisc { on | off }
```

# vlan

## 确保模块加载

```bash
modinfo 8021q
```

## ip命令

linux中配置vlan可使用ip命令

```bash
ip link add DEVICE type TYPE ARGS

#TYPE可选
TYPE := { vlan | veth | vcan | dummy | ifb | macvlan | macvtap |
          bridge | bond | team | ipoib | ip6tnl | ipip | sit | vxlan |
          gre | gretap | ip6gre | ip6gretap | vti | nlmon | team_slave |
          bond_slave | ipvlan | geneve | bridge_slave | vrf | macsec }


[root@matrix_01 ~]# ip link help vlan
Usage: ... vlan id VLANID
                [ protocol VLANPROTO ]
                [ reorder_hdr { on | off } ]
                [ gvrp { on | off } ]
                [ mvrp { on | off } ]
                [ loose_binding { on | off } ]
                [ ingress-qos-map QOS-MAP ]
                [ egress-qos-map QOS-MAP ]

VLANID := 0-4095
VLANPROTO: [ 802.1Q / 802.1ad ]
QOS-MAP := [ QOS-MAP ] QOS-MAPPING
QOS-MAPPING := FROM:TO

#设置vlan，eth0为设置vlan接口
ip link add link eth0 name eth0.8 type vlan id 8
```

## 配置文件

### 上级接口配置

/etc/sysconfig/network-scripts/ifcfg-ethX

```bash
DEVICE=ethX
TYPE=Ethernet
BOOTPROTO=none
ONBOOT=yes
```

### vlan接口配置

/etc/sysconfig/network-scripts/ 目录中配置 VLAN 接口。配置文件名应为上级接口加上 . 字符再加上 VLAN ID 号码

```bash
#此处vlan为192
DEVICE=ethX.192
BOOTPROTO=none
ONBOOT=yes
IPADDR=192.168.1.1
PREFIX=24
NETWORK=192.168.1.0
VLAN=yes
```

## 查看vlan

```bash
#-d为details
ip -d link show eth0.8
```

## 删除vlan

```bash
ip link delete eth0.8
```



