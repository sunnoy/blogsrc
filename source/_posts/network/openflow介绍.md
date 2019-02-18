---
title: openflow介绍
date: 2018-12-01 12:18:28
tags:
- openflow
---

# openflow介绍

openflow是一个开放标准，也是一个网络架构中控制层和数据层之间的通信协议。

在传统的交换机或者路由器上，要想实现转发就要存在mac表和路由表，openflow和传统的转发协议有着类似的地方，但是openflow协议内存在的是流表，流表囊括了mac表和路由表。ovs是openflow的开源具体实现，也有实现openflow协议的硬件产品，本文根据openflow协议1.5.1介绍。


![openflow-sdn-architecture](https://qiniu.li-rui.top/openflow-sdn-architecture.png)

<!--more-->

openflow由ONF(Open Networking Foundation )来维护，他提供每个版本的标准规格，详细的规格获取[地址](https://www.opennetworking.org/software-defined-standards/specifications/)

SDN和openflow的关系
- SDN不等同于openflow
- SDN不仅是将控制层和数据层分离

# 网络抽象

SND中的控制层是对网络抽象封装。通过控制层，应用可以通过控制层的api去操作网络设备，而不是直接操作网络设备。openflow就是完成这个抽象控制层的一种方法。

openflow操作的是网络设备的流表而不是网络设备自身，因此一些网络设备自身的配置就需要其他工具来完成。

![openflow-sdn-architecture-2](https://qiniu.li-rui.top/openflow-sdn-architecture-2.png)

## 基本架构

![openflow-components](https://qiniu.li-rui.top/openflow-components.png)

- Remote Controller：我们知道，SDN网络将传统的网络结构划分成了Control Plane和Data Plane两部分，这里的Remote Controller正是Control Plane部分，通过约定的通信协议来远程控制管理OpenFlow Switch，增加、删除或者修改OpenFlow Switch的Flow Entries。
- OpenFlow Channel：它可以被认为是OpenFlow Switch对外开放的接口，接受来自于Remote Controller的通信协议，进而来操纵OpenFlow Switch。
- OpenFlow Protocol：一种通信协议规范，用于Remote Controller和OpenFlow Switch之间的消息交换。
- Flow Table：包含多个Flow Entry记录，控制数据包的流向
- Group Table：相对于Flow Table，控制着数据包更高级的转发特性，比如Flooding、Multipath、Fast Reroute、Link Aggregation等。
- Pipeline：由多个Flow Table链接而成，控制数据包的一系列行为。

总的看来openflow有三大核心组件

- openflow控制器
- openflow协议
- 交换机和其中的一系列流表

## OpenFlow 端口

openflow端口就是网络接口，用于在openflow和外界进行流量传递。openflow交换机必须支持下面三种端口

- Physical Ports 和硬件设备直接关联的端口
- Logical Ports 不是硬件接口，比如隧道，回环接口等
- Reserved Ports 保留接口用于进行一些动作，比如广播，常见的有ALL, CONTROLLER, TABLE, IN_PORT, ANY, UNSET, LOCAL

## openflow交换机

openflow交换机分为两类

- OpenFlow-only 所有的数据包只能通过openflow流表的pipeline
- OpenFlow-hybrid 既可以使用openflow操作也可以使用常见的以太网交换机操作

OpenFlow-hybrid交换机会用到reserver port中的NORMAL或者FLOOD端口

![openflow-hybrid](https://qiniu.li-rui.top/openflow-hybrid.png)

## 流

流由管理员定义，根据不同的流执行不同的策略。同一时间内，经过同一网络并且具有相同属性的数据包走向就是一个流。例如我们可以规定访问同一个IP为一个流，也可以规定同一源目IP的数据为一个流或者同一个协议为一个流。所有数据都是以流为单位进行处理的。

## 流表

### 介绍

![openflow](https://qiniu.li-rui.top/openflow.png)

流表相比传统的mac表和路由表有很多的优势

- 可以编程化操作
- 可以出现花样的转发规则
- 可以整出繁多的表
- 除了传统mac表和路由表中动作的转发和丢弃外还会有第三个动作 去下一张表

一个ovs交换机里面可以存在200多张流表，每个流表都有流表相组成。流表由controller来下发。

### 流表项

流表项分为两类

- 被动流表项 无需控制器操作就含有的流表项，比如mac学习
- 主动流表项 应用通过控制器来操作的流表项

流表项定义出流的规则。流表项包含有

- 匹配字段（Match Filed）
- 优先级（Priority）
- 计数器（Counters）
- 指令（Instruction）
- 超时（Timeouts）
- Cookies
- flags

### pipeline

每个openflow交换机内至少存在一个流表，流表的匹配必须从table0开始，table0中优先级大的先匹配。`goto`语句可以将包指向其他比本次匹配流表号大的流表。

![openflow-flow](https://qiniu.li-rui.top/openflow-flow.png)

### Table-miss流表项

如果进入流表内包无法匹配就会进入Table-miss流表项处理。Table-miss在流表中的优先级是0，可以匹配任何包，Table-miss的流表项动作可以去到其他流表也可以drop包

![openflow-flow-det](https://qiniu.li-rui.top/openflow-flow-det.png)

## 组表

每个交换机内都存在一个组表(Group Table)，组表可以存在更高级的数据包转发特性。组表有组表项组成：
- Group Identifier：Group Entry的唯一标识。
- Group Type：表明了对数据包的处理行为
- Counters：被该Group Entry处理过的数据包的统计量。
- Action Buckets：一个Action Bucket的有序列表，每个Action Bucket又包含了一组Action集合及其参数。

### Group Type

Group Type有多种类型可用

- all 该类型中的数据包都会执行Action Buckets中的动作，常用于多播和广播
- select 仅仅执行Group Table中的某一个Action Bucket
- indirect：执行Group Table中已经定义好的Action Bucket
- fast failover：执行第一个live的Action Bucket (组表可以不包含此项)

## Meter Table

Meter Table用来对流进行qos限制，包含多个meter table项

- Meter Identifier：Meter Entry的唯一标识。
- Meter Bands：一个无序的Meter Band集合，每个Meter Band指明了带宽速率以及处理数据包的行为。
- Counters：被该Meter Entry处理过的数据包的统计量。

### Meter Bands

Meter Bands 用来限定包的各项参数。有下面部分组成

- Band Type：定义了数据包的处理行为。包含drop和dscp remark(增加数据包IP头DSCP域的丢弃优先级)
- Rate：Meter Band的唯一标识，定义了Band可以应用的最低速率。
- Counters：被该Meter Band处理过的数据包的统计量。
- Type specific arguments：某些Meter Band有一些额外的参数。

## openflow消息类型

openflow消息，也就是发包类型。存在三种

### Controller-to-switch

控制器和交换机之间，包含

- Features – switch needs to request identity
- Configuration – set and query configuration parameters
- Modify-State – also called ‘flow mod’, used to add, delete and modify flow/group entries
- Read-States – get statistics
- Packet Outs – controller send message to the switch, either full packet or buffer ID.
- Barrier – Request or reply messages are used by controller to ensure message dependencies have been met, and receive notification.
- Role-Request – set the role of its OpenFlow channel
- Asynchronous-Configuration – set an additional filter on asynchronous message that it wants to receive on OpenFlow Channel

### Asynchronous

- Packet-in – transfer the control of a packet to the controller
- Flow-Removed – inform controller that flow has been removed
- Port Status – inform controller that switch has gone down
- Error – notify controller of problems

### Symmetric

- Hello – introduction or keep-alive messages exchanged between switch and controller
- Echo – sent from either switch or controller, these verify liveness of connection and used to measure latency or bandwidth
- Experimenter – a standard way for OpenFlow switches to offer additional functionality within the OpenFlow message type space.

# openflow测试

openflow测试工具有

- [mininet](https://github.com/mininet/mininet/wiki/Introduction-to-Mininet) OpenFlow Switch
- [OpenDaylight](https://www.opendaylight.org/) OpenFlow Controller

openflow交换机和控制器连接走的TCP协议，openflow1.3.2之前为端口为TCP 6633之后为TCP 6653 post

## 抓包分析

- OpenFlow Switch是192.168.0.11
- OpenFlow Controller是192.168.0.12

### 初始状态

创建测试

```bash
mn --controller=remote,192.168.0.12 --mac --topo=single,2 --switch=ovsk,protocols=OpenFlow13
```

起始状态交换机初始化OpenFlow Channel，交换机向控制器发送HELLO信息

![openflow-wireshark-1](https://qiniu.li-rui.top/openflow-wireshark-1.png)

### 信息获取

接着控制器向交换机发送`OFPT_FEATURES_REQUEST`

![openflow-wireshark-2](https://qiniu.li-rui.top/openflow-wireshark-2.png)

交换机向控制器回复`OFPT_FEATURES_REPLY`

![openflow-wireshark-3](https://qiniu.li-rui.top/openflow-wireshark-3.png)

n_tables 表示交换机支持的流表数量

### 多次请求压缩

OFPT_MULTIPART_REPLY, OFPMP_DESC包用来表示多次请求的压缩

![openflow-wireshark-5](https://qiniu.li-rui.top/openflow-wireshark-5.png)

### PACKET-IN和PACKET-OUT

PACKET-IN表示从交换机到控制器

交换机接口之间数据传递将会被openflow协议包进行封装

![openflow-wireshark-8](https://qiniu.li-rui.top/openflow-wireshark-8.png)

也可以通过命令`ovs-ofctl -O OpenFlow13 dump-flows s1` 来查看流信息



















