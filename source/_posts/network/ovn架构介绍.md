---
title: ovn架构介绍
date: 2018-12-15 12:12:28
tags:
- openflow
---

# ovn基本介绍

ovn是一个sdn控制器，是OVS提供的原生虚拟化网络方案，旨在解决传统SDN架构的性能问题

<!--more-->

# 架构

架构图

```bash

                                         CMS
                                          |
                                          |
                              +-----------|-----------+
                              |           |           |
                              |     OVN/CMS Plugin    |
                              |           |           |
                              |           |           |
                              |   OVN Northbound DB   |
                              |           |           |
                              |           |           |
                              |       ovn-northd      |
                              |           |           |
                              +-----------|-----------+
                                          |
                                          |
                                +-------------------+
                                | OVN Southbound DB |
                                +-------------------+
                                          |
                                          |
                       +------------------+------------------+
                       |                  |                  |
         HV 1          |                  |    HV n          |
       +---------------|---------------+  .  +---------------|---------------+
       |               |               |  .  |               |               |
       |        ovn-controller         |  .  |        ovn-controller         |
       |         |          |          |  .  |         |          |          |
       |         |          |          |     |         |          |          |
       |  ovs-vswitchd   ovsdb-server  |     |  ovs-vswitchd   ovsdb-server  |
       |                               |     |                               |
       +-------------------------------+     +-------------------------------+
```

## 组件以及作用

### 主要组件

- Cloud Management System (CMS) Cloud Management System (CMS)可认为是ovn的一个强大的客户端，openstack就是一个cms。cms和ovn交互需要相应的cms插件
- OVN Database 存放ovs配置信息等
- ransport node或者chassis ovs接口节点，一般为hypervisors节点或者gateways

### 各组件作用

从上到下依次：

#### cms以及插件

cms以及插件用来将cms中的相关配置信息输入到ovn体系中

#### OVN Northbound  Database

OVN Northbound  Database接受cms插件传递的虚拟网络配置信息，可以使用命令`ovn-nb`配置。nb还有一个客户端就是ovn-northd(命令)

#### ovn-northd

ovn-northd连接两个数据库，OVN Northbound DB和OVN Southbound DB，主要作用是将OVN Northbound DB中的虚拟网络配置信息转化为逻辑datapath flows信息存入OVN Southbound DB

#### OVN Southbound Database

OVN Southbound Database是整个ovn系统的核心组件，其客户端为ovn-northd和ovn-controller。该数据中存在三个表

- 用于连接计算节点的物理网络表(PN) 计算节点维护
- 存放逻辑datapath flows信息的逻辑网络表LN ovn-northd维护
- bind 表用于存放PN和LN之间的绑定信息 计算节点维护

随着集群规模扩大OVN Southbound Database会出现瓶颈，可能需要集群部署

#### ovn-controller

ovn-controller在计算节点或软件网关中为ovn的agent，主要连接Southbound Database并填充相关信息

在北向：它采集ovs配置填充PN表，采集计算节点信息填充bind表
在南向：作为ovs控制器和ovs-vswitchd连接，并配置ovsdb-server数据库

### 信息流

ovn配置信息从北到南，由cms插件将逻辑网络配置输入ovn系统，通过ovn-northd将逻辑网络配置转化为可以被ovn系统识别的信息

ovn flow信息从南到北，比如：cms如何知道一个vm是否在线？首先，ovn-northd需要在northbound数据库Logical_Switch_Port表中填充up列，如果在northbound数据库Port_Binding表中，一个逻辑端口的chassis列非空，ovn-northd就在up列中填充true，否则为false。
其次，ovn还会对cms对vm配置成功与否给出反馈。

## 计算节点ovs

计算节点上需要有一个ovn专用的ovs，成为integration bridge，而且该ovs需要先于ovn-controller启动。该网桥名称习惯命名为br-int，并且需要加入两项配置

- fail-mode=secure ovs会等控制器启动才会初始化flow
- other-config:disable-in-band=true 主要是因为ovs使用的逻辑控制器而不是远程控制器，防止对系统中其他远程控制器造成影响，因此禁用带内管理。
带内(inband)管理和和带外管理(outband)主要区别在管理网络是否占用业务网络。

直接启动ovn-controller，ovn-controller会默认创建ovs，该ovs带有一下默认端口：隧道端口和对于计算节点为虚拟端口，对于网关为patch端口

## 逻辑网络(logical network)

ovn中的逻辑网络和物理网络通过隧道隔离，需要使用和物理网络不一样的地址区间。

ovn中有关的网络概念：

- 逻辑交换机(Logical  switches) 以太网交换机
- 逻辑路由器(Logical routers)
- 逻辑datapath(Logical  datapaths) openflow交换机，包含逻辑交换机和逻辑路由器
- 逻辑端口 是逻辑交换机和逻辑路由器的连接出口和入口
  - 逻辑端口就是VIF(virtual network interface)，
  - Localnet  ports 连接着逻辑交换机和物理网络，利用patch端口实现，一边为integration bridge，一边为连接物理接口的ovs交换机
  - Logical patch ports 逻辑交换机和逻辑路由器之间，或者逻辑路由器之间的连接
  - Localport ports 该端口流量不经过隧道，用于在计算节点上和vm通讯

# vif和cif接口生命周期

## VIF生命周期

VIF在hypervisor中是vm或容器上的接口。


1 管理员通过cms中api创建vif并将它加到逻辑交换机上，并分配vif-id和mac

2 CMS plugin将创建的vif更新到OVN  Northbound database，并初始化相关字段

3 ovn-northd收到Northbound database跟新后就在Southbound database中做关联更新。主要包括在Logical_Flow表中初始化flow，和填充相关的字段

4 在计算节点上，ovn-controller接收Logical_Flow表的更新，然后等待绑定vif的vm开机

5 vm开机后，vif就会加到OVN integration bridge上，并且会将vif-id放到external_ids:iface-id中，表示这是一个新的vif实例

6 ovn-controller注意到external_ids:iface-id存在新的vif实例后，ovn-controller将会更新Southbound DB中binding表中的chassis列。接着ovn-controller更新本地的openflow表，使经过此vif的包可以被转发

7 但是在openstack中网络准备好才开机vm，那怎么办呢？ovn-northd来更新Northbound  database中Logical_Switch_Port表中up列，显示网络已经完成

8 除vif所在的计算节点外，其他所有的计算节点上的ovn-controller都会去检查Binding表中新vif所在逻辑端口的物理位置，因此这些计算节点上的ovn-controller都会跟新本地ovs，以至于从新的vif经过的包可以进入隧道转发

9 vm关机后，integration bridge会删除相关vif

10 ovn-controller检测到vif删除后，会相应的移除binding表中Chassis列的逻辑端口

11 所有的计算节点上的ovn-controller注意到binding表中Chassis列中为空后，就会对本地openflow表做reflect操作

12 最终由cms管理员删除该vif

13 CMS  plugin从Northbound database中删除

14 ovn-northd删除在Southbound database相关联的字段

15 在所有的计算节点上ovn-controller将会在本地openflow表进行relect该更新

## vm中cif生命周期

容器在计算节点上直接运行容器上接口(container interface,CIF)和vm是一样的，容器在计算节点上的vm中运行稍微复杂一些。

ovn使用一个vif来对应所有的cif，并为每一个cif打上vlan标签来区分来自不同cif的包

1 cif创建后需要知道所在vm的vif-id以及没有使用过的vlan id

2 容器创建后，创建容器的cms或者其他的工具需要更新northbound db中Logical_Switch_Port，分别填充name、parent_name(vif-id)、tag等信息

3 ovn-northd收到northbound db更新后就会对更新信息做转换到southbound db中的Logical_Flow表和Binding表中除chassis外的字段

4 所有计算节点上的ovn-controller，都会收到bingding表的更新。当新建容器的计算节点上的ovn-controller发现bingding表的external_ids:iface-id字段值vif-id在本地OVN  integration bridge上面，ovn-contrller就会更新本地的openflow表，使进出cif带vlan的包可以被转发

5 如果需要等网络完成后才启动容器，类似openstack启动vm，ovn-northd将会关注binging表中的chassis字段，然后把northbound db中Logical_Switch_Port表的up字段跟新，容器管理工具就会看到该更新然后启动容器

6 容器关闭后就通过容器管理工具删除Logical_Switch_Port相关字段

7 ovn-northd接收更新后，也会删除southbound db中的cif相关信息

8 每个计算节点，ovn-controller收到ovn-north对Logical_Flow做的更新后也会删除本地openflow表中的字段

# ovn中包转发过程

vm之间数据在ovn中如和走？

## 元数据字段

- tunnel key 隧道key，不同的隧道不同名称
- logical datapath 逻辑datapath，可以理解为ovs标识
- logical input port ovs上入口标识，ovn设置ovs寄存器(register)为14，Geneve and STT隧道会携带该字段，register为包在不同flow表中的标识，初始化为0
- logical output port ovs出口标识，ovs设置ovs寄存器(register)为15，Geneve and STT隧道会携带该字段，vxlan会发到table 8寻找outport，到table 32 就会到table 33进行本地转发
- conntrack zone field(logical ports)，仅对本地有用，ovn设置ovs寄存器(register)为13
- conntrack zone fields(routes)，仅对本地有用，ovn对DNAT设置为ovs寄存器(register)11，对SNAT设置为ovs寄存器(register)12
- logical flow flags 用来在flow表之间的规则匹配，仅对本地有用，ovn设置ovs寄存器(register)为10
- VLAN ID vm中容器使用

## 包在交换机中的转发过程

vm或者容器向OVN integration bridge上的入口发了一个包：

1 根据openflow协议，包首先进入openflow table 0，然后设置logical datapath和logical input port，接着就会把包resubmits到table 8开始进入ingress pipeline

而对于vm中容器的包会匹配到vlan id然后strip掉vlan，然后进入ingress pipeline

从其他计算节点通过不同隧道到本地的包也会进入talbe 0，除了设置logical datapath和logical input port还会设置logical output port，这些都可以从隧道获取。最后将包resubmit到table 33 进入egress pipeline

2 openflow table 8 到 31 主要执行ingress pipeline，这些table在southbound db中表Logical_Flow定义。ovn-controller会将Logical_Flow 表 0 到 23 转化为openflow table 8 到 31

ovn-controller将logical flow的uuid的前32为作为其openflow 中flow的cookie 而对于conjunctive match字段匹配，cookie设置为0，对于没有vif可以匹配的logical flow则丢弃

ovn中大多数actions和openflow类似，next类似resubmit；field  =  constant;类似set_field，更多字段详见[ovn-sb(5)](http://www.openvswitch.org/support/dist-docs/ovn-sb.5.html)

3 openflow table 32 到 47 实现output actions，表32处理有关远程计算节点的包，表33处理有关本地计算节点的包，表34查看包logical出入口是否一致，一致则丢弃

logical patch port包也由表33处理，为单播模式。多播模式包由表32处理

在表32中由单播和多播包，对于单播包设置当前计算节点的隧道key，远程节点会识别该包然后交给表33处理。对于多播包，将包复制一下打上当前隧道id发往远程节点。如果多播包中含有本地端口就发往表33

表32对于vxlan包处理，包上面有MLF_RCV_FROM_VXLAN flags发往表33到本地，其他的都到表8选择outport，具有高优先级

表32收到localport类型包，发往表33。没有其他的匹配就发往33

表33中，对于output到本地的单播包发往表34，对多播包，因为存在多个logical port，对每一个port都改变output然后发往表34

对于存在localnet并且和远程端口相连，包发往表33

表34drop掉入口和出口一样并且MLF_ALLOW_LOOPBACK flag没有设置的包，其他的包发往表40

4 表40到63处理Southbound db 中Logical_Flow表中egress pipeline，ovn-controller执行output action将包发往表64，没有outport的包直接drop掉

5 在表64中如果MLF_ALLOW_LOOPBACK设置，先保存ingress端口，然后设置0，发往表65，如果MLF_ALLOW_LOOPBACK没有设置直接发往表65

6 表65处理逻辑到物理转换，和表0相反。将包发往OVN integration bridge，如果是vm内的容器就加上valn id

## 包在“交换机-路由器-交换机”中的转发流程

### Gateway Routers

上一部分介绍了包在单个逻辑交换机中的旅程，这部分介绍包在“交换机-路由器-交换机”之间的旅程。

对于不同子网的vif或者cif包转发需要用到路由和交换机之间的patch port。现在假设一个包从vif或cif发到不同子网的vif或cif。

当包在一个交换机的pipeline被转发到表65时，逻辑egress port 就是逻辑patch port，对于不同的ovs版本，表65有不同的实现

- ovs <= 2.6中 表65中的包会被转发到patch port的另一端的路由，从另一端的路由的表0开始
- ovs >= 2.7中 表65中的包会复制一份被转发到patch port的另一端的路由，从另一端路由中的ingress port开始，并不一定是表0开始

包通过交换机-路由之间的patch port到达路由后仍需要从走表8到65的流程.

包在路由中被转发到表65，接着就通过路由-交换机之间的patch port进入交换机(另一子网的vif或cif相连的交换机)，同样需要走表8到65的流程(第三次8到65了喔)。如果目的vif或cif在远程就通过隧道发出去，如果在本地就直接到交换机上的vif或者cif

上面介绍的都是单逻辑路由和单逻辑交换之间的包走向，下面看“逻辑路由+逻辑交换机”组成的整体逻辑设备(Gateway Routers)之间的包走向

Gateway Routers就是在特定计算节点上的逻辑路由+逻辑交换机，以及他们之间的相关的patch port和peer port。这种类型逻辑设备在southbound db表Port_Binding中设定的port类型为l3gateway，因为这些patch port是和计算节点绑定的。

在Gateway Routers中包的egress port是3gateway，数据包会转发到表32然后去发往远程计算节点中的gateway router，分布式路由和物理网络均通过逻辑交换机和Gateway Routers相连，Gateway Routers在本地节点做snat让大家上网

Gateway Routers

### Distributed Gateway Ports

本地一个物理接口比如eth0绑定在逻辑交换机上(具有localnet)，该逻辑交换机和分布式逻辑路由相连，他们之间的patch port就是Distributed Gateway Ports。该端口主要用来当前计算节点的vif或cif需要去外网的情况，这些流量可以直接通过该端口去外网或者从外网到vif或cif，通过localnet。可见，每一台计算节点上都需要有Distributed Gateway Ports

# 隧道封装

计算节点之间的隧道包有三种元数据

- 24位的logical datapath标识符，来自哪个交换机
- 15位的logical ingress port标识符，0保留，1-32767为port
- 16位的logical egress  port标识符，0-32767和ingress port一致，32768-65535使用多播

计算节点之间的隧道类型仅支持geneve 和 stt，主要是因为他们都支持每个包超过32位元数据


















































