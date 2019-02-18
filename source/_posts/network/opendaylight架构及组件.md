---
title: opendaylight架构及组件
date: 2018-12-14 12:12:28
tags:
- openflow
---

# 介绍

opendaylight是款sdn控制器，支持多种协议，包括bgp，ovsdb，openflow等，也是一个平台，[平台组件](https://docs.opendaylight.org/en/stable-oxygen/user-guide/index.html)

opendaylight基于osgi，运行在karfa容器中，平台组件均通过feature来插拔式安装，是一种装配式架构。

![OpenDaylight_logo](https://qiniu.li-rui.top/OpenDaylight_logo.png)

<!--more-->

## 版本历史

opendaylight版本命名依据元素周期表顺序，

![odlre](https://qiniu.li-rui.top/odlre.png)

从Fluorine开始，不再内置OpenDaylight User Interface (DLUX)组件，本文使用Oxygen版本


# Service Abstraction Layer(sal)

opendaylight中服务层有两方面作用

- SAL北向连接功能模块，以插件的形式为之提供底层设备服务
- 南向连接多种协议，屏蔽不同协议的差异性，为上层功能模块提供一致性服务

## ad-sal

ad-sal定义的抽象服务，屏蔽了南向协议差异，提供统一的抽象服务和API，并提供数据提供者(provider)和数据消费者(consumer)之间的Request Routing；北向Plugin可以通过调用我的北向API来实现对南向Plugin的调用，操作其所管理的设备和服务。在我这里，抽象服务由南向和北向API实现，南北向API是一对一映射关系。

api增多容易导致复杂化，api越来越庞大

![odlsal](https://qiniu.li-rui.top/odlsal.png)

## md-sal

### YANG模型 

YANG模型是一种数据建模语言，用来建模由NETCONF协议、NETCONF远端过程调用（RPCs）、和NETCONF通知（notification）操作的配置数据和状态数据。
YANG建模NETCONF协议的操作和内容层（RFC4741，Section 1.1）

为什么采用yang模型：

在开发过程中，一般先根据需求先设计YANG模型来确定准备提供哪些数据和服务，然后使用Maven进行编译时，通过YANG Tools插件将YANG文件的转译为相应的java类、接口。开发者通过实现自动生成的Java代码来实现具体的API和服务内容。

由于YANG文件通俗易懂，我们完全可以仅仅通过YANG文件就可以大概了解到系统提供的数据和服务内容，十分的方便。同时，OpenDaylight访问Data Store的API完全基于Yang模型，这部分代码Yang Tools插件自动生成，这样开发人员只需关注如何进行提供服务，大大提高了开发效率和降低了OpenDaylight控制器的开发难度。

### 具体过程

- 抽象服务和相应API是由各个Plugin通过YANG Model和Service来定义
- Yang Tools Plugin通过各个Plugin的model定义来自动生成API、Service Interface和相应Java代码。
- 开发者通过实现自动生成的Service Interface来实现具体的API和服务内容
- Plugin通过MD-SAL和生成的API（RPC, Notification）、DataStore去利用其他各个Plugin的服务和数据

![yangodl](https://qiniu.li-rui.top/yangodl.png)

### 如何调用

三种角色

- Consumer：使用由Provider提供的模型或API的组件，例如应用程序。
- Provider：通过特定的模型或API为应用程序提供功能服务的组件。
- Broker： 核心部分，实现在Provider/Consumer间路由RPC、Notification、Data Change等消息

主要完成的主要功能就是Provider和Consumer之间的连通

三种服务

- RPC：提供服务的远程调用接口。
- Notification：提供通知，可以发出通知和接收通知。
- DataStore：提供数据存储，读取，Transaction等功能。

![md-sal](https://qiniu.li-rui.top/md-sal.jpg)

### 两个数据库

主要是使用yang接口进行操作，yang接口主要是对两个Data Store树形数据库进行增删查改

- config库 存放持久化信息，一般是配置信息，比如对ovs的配置信息，几个端口以及几个vlan
- operational库 存储状态信息，存在内存中。比如查看ovs的端口和valn统计信息

示例

```bash
#建立拓扑 注意config
PUT http://localhost:8080/restconf/config/network-topology:network-topology/topology/<id>

#查询拓扑 注意operational
GET http://localhost:8080/restconf/operational/network-topology:network-topology/topology/<id>
```

# 安装

## 环境准备

[官网下载](https://docs.opendaylight.org/en/stable-fluorine/downloads.html)

准备好java环境，主要是添加环境变量

```bash
tail /etc/profile
export JAVA_HOME=/data/odl/jdk1.8.0_112
export CLASSPATH=$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export PATH=$PATH:$JAVA_HOME/bin:$CATALINA_HOME/lib:$CATALINA_HOME/bin

```

## 运行

解压后直接运行，

```bash
[root@node1 bin]# pwd
/data/odl/karaf-0.8.3/bin
[root@node1 bin]# ./karaf
Apache Karaf starting up. Press Enter to open the shell now...
100% [========================================================================]
Karaf started in 65s. Bundle stats: 400 active, 401 total

    ________                       ________                .__  .__       .__     __
    \_____  \ ______   ____   ____ \______ \ _____  ___.__.|  | |__| ____ |  |___/  |_
     /   |   \\____ \_/ __ \ /    \ |    |  \\__  \<   |  ||  | |  |/ ___\|  |  \   __\
    /    |    \  |_> >  ___/|   |  \|    `   \/ __ \\___  ||  |_|  / /_/  >   Y  \  |
    \_______  /   __/ \___  >___|  /_______  (____  / ____||____/__\___  /|___|  /__|
            \/|__|        \/     \/        \/     \/\/            /_____/      \/


Hit '<tab>' for a list of available commands
and '[cmd] --help' for help on a specific command.
Hit '<ctrl-d>' or type 'system:shutdown' or 'logout' to shutdown OpenDaylight.

opendaylight-user@root>

```

或使用start脚本启动，退出进程还在

```bash
./start
```

关闭

```bash
opendaylight-user@root>shutdown
Confirm: halt instance root (yes/no): yes
```

# 平台内组件

opendaylight是一个平台，提供了一个框架。开发者们提供了非常多的组件，每个组件都是一个项目，[项目列表](https://docs.opendaylight.org/en/stable-oxygen/user-guide/index.html).

这里对每个项目做简要介绍，看一下sdn里面都在玩些啥？

要使用某个项目需在karaf要安装相应的一个或多个feature

相关命令

```bash
#查看已经安装的
feature:list -i
#组件状态检查 
list
#查看日志
log:tail 

```

清理项目

```bash
./karaf  clean
#或者
rm /opt/karaf-0.7.0/data/
```

## opendaylight restful

提供核心的opendaylight api接口，以及拓扑抽象

```bash
feature:install odl-restconf-all
```

## dlux

平台web ui组件

### 安装

需要安装dlux.[文档地址](https://docs.opendaylight.org/en/stable-oxygen/user-guide/using-the-opendaylight-user-interface-(dlux).html)

```bash
feature:install odl-dlux-core*
feature:install odl-dluxapps*
feature:install odl-dluxapps-nodes
feature:install odl-dluxapps-yangui
feature:install odl-dluxapps-yangman
feature:install odl-dluxapps-topology
feature:install odl-dluxapps-yangutils
feature:install odl-dluxapps-applications
```

访问地址：http://HOSTIP:8181/index.html username和password均为 admin

## Netvirt

opendaylight提供的网络虚拟化解决方案，整体实现和ovn类似。[项目地址](https://docs.opendaylight.org/projects/netvirt/en/stable-oxygen/index.html)

## ALTO

ALTO就是Application Layer Traffic Optimization，用来配置应用流量.[项目地址](https://docs.opendaylight.org/en/stable-oxygen/user-guide/alto-user-guide.html)


## AAA

AAA就是Authentication, Authorization and Accounting，odl的用户和权限组件[项目地址](https://docs.opendaylight.org/en/stable-oxygen/user-guide/authentication-and-authorization-services.html)

## BGP

### BGP组件

BGP组件，可用于实现sdn-wan.[项目地址](https://docs.opendaylight.org/en/stable-oxygen/user-guide/bgpcep-guide/bgp/index.html)

### BGP Monitoring Protocol (BMP) 

BGP Monitoring Protocol (BMP)用来监控bgp设备运行状态，详见[rfc](https://tools.ietf.org/html/rfc7854#section-1).[项目地址](https://docs.opendaylight.org/en/stable-oxygen/user-guide/bgpcep-guide/bmp/bgp-monitoring-protocol-user-guide-overview.html)



## Bit Index Explicit Replication (BIER) 

Bit Index Explicit Replication (BIER) 一种改进的多播方式，[基本介绍](https://www.ietfjournal.org/an-overview-of-bit-index-explicit-replication-bier/)。[rfc8279](https://tools.ietf.org/html/rfc8279)

## CAPWAP

 the Control And Provisioning of Wireless Access Points无线接入点控制协议，用来无线终端接入点（AP）和无线网络控制器（AC）之间的通信交互，实现AC对其所关联的AP集中管理和控制。[基本介绍](https://cshihong.github.io/2018/04/25/CAPWAP%E5%9F%BA%E7%A1%80%E5%8E%9F%E7%90%86/)。[项目地址](https://docs.opendaylight.org/en/stable-oxygen/user-guide/capwap-user-guide.html)

## Cardinal

odl自身的监控服务。[项目地址](https://docs.opendaylight.org/en/stable-oxygen/user-guide/cardinal_-opendaylight-monitoring-as-a-service.html)

## Centinel

Centinel提供一个框架，该框架用来搜集、聚合一些流信息(比如Elastic search)，并采取相应的动作。[项目地址](https://docs.opendaylight.org/en/stable-oxygen/user-guide/centinel-user-guide.html)

## Data Export/Import

提供一个api供用户从odl和本地系统之间进行数据导入导出。[项目地址](https://docs.opendaylight.org/en/stable-oxygen/user-guide/daexim-user-guide.html)

## DIDM

DIDM The Device Identification and Driver Management 一个设备抽象组件，来不屏蔽不同设备的差异性。[项目地址](https://docs.opendaylight.org/en/stable-oxygen/user-guide/didm-user-guide.html)

## Distribution Version reporting

Distribution Version reporting 用来对不同odl版本进行识别，因为odl不同版本的组件名称以及使用方式是不一样的。[项目地址](https://docs.opendaylight.org/en/stable-oxygen/user-guide/distribution-version-user-guide.html)

## eman

eman为Energy Management，用于对不同类型的设备进行控制。[项目地址](https://docs.opendaylight.org/en/stable-oxygen/user-guide/eman-user-guide.html)

## Fabric

该组件主要针对南向接口，南向接口的通讯主要使用命令行和openflow等协议，但是他们都是基于拓扑的而不是基于应用的。fabric做了一层底层封装，来让应用和网络进行通信。[项目地址](https://docs.opendaylight.org/en/stable-oxygen/user-guide/fabric-as-a-service.html)

## Genius

Genius用于提供一个通用的接口树。[项目地址](https://docs.opendaylight.org/en/stable-oxygen/user-guide/genius-user-guide.html)

## Group Based Policy 

Group Based Policy  GBP用来让用户使用声明式的方式配置网络。[项目地址](https://docs.opendaylight.org/en/stable-oxygen/user-guide/group-based-policy-user-guide.html)

## JSON-RPC

JSON-RPC 远程调用组件 [项目地址](https://docs.opendaylight.org/en/stable-oxygen/user-guide/jsonrpc-user-guide.html)

## L2 Switch

L2 Switch 让虚拟交换机具有传统交换机的能力，比如mac学习等。[项目地址](https://docs.opendaylight.org/en/stable-oxygen/user-guide/l2switch-user-guide.html)

## LACP

LACP Link Aggregation Control Protocol 一个链路聚合控制协议组件，用于二层多条链路的聚合。[项目地址](https://docs.opendaylight.org/en/stable-oxygen/user-guide/link-aggregation-control-protocol-user-guide.html)

## LISP Flow Mapping

LISP Locator/ID Separation Protocol 一个overlay网络框架。[详见rfc6830](https://tools.ietf.org/html/rfc6830)。[项目地址](https://docs.opendaylight.org/en/stable-oxygen/user-guide/lisp-flow-mapping-user-guide.html)

## NEMO

NEtwork MOdeling 一个简化northbound interface的组件。[项目地址](https://docs.opendaylight.org/en/stable-oxygen/user-guide/nemo-user-guide.html)

## NETCONF

NETCONF  Network Configuration Protocol 一个基于xml的配置和监控网络中设备的协议，[详见rfc6241](https://tools.ietf.org/html/rfc6241).[项目地址](https://docs.opendaylight.org/en/stable-oxygen/user-guide/netconf-user-guide.html)

## NetIDE

NetIDE是一个接口转换组件，可以让为其他sdn控制器写的应用也可以对odl使用。[项目地址](https://docs.opendaylight.org/en/stable-oxygen/user-guide/netide-user-guide.html)

## Neutron

和openstack集成的组件。[项目地址](https://docs.opendaylight.org/en/stable-oxygen/user-guide/neutron-service-user-guide.html)

## NIC

Network Intent Composition 可以让用户描述一种状态，然后odl去操作系统达到这个状态。有点像k8s的机制。[项目地址](https://docs.opendaylight.org/en/stable-oxygen/user-guide/network-intent-composition-(nic)-user-guide.html)

## OCP

OCP一个南向插件，用于控制远程的Remote Radio Head 。[项目地址](https://docs.opendaylight.org/en/stable-oxygen/user-guide/ocp-plugin-user-guide.html)

## ODL-SDNi 

ODL-SDNi 是Software Defined Networking interface，[项目地址](https://docs.opendaylight.org/en/stable-oxygen/user-guide/odl-sdni-user-guide.html)

## OF-CONFIG

OF-CONFIG组件可以将一个OpenFlow switch 定义为一个逻辑交换机。[项目地址](https://docs.opendaylight.org/en/stable-oxygen/user-guide/of-config-user-guide.html)

## openflow

需要安装相关组件,[文档地址](https://docs.opendaylight.org/en/stable-oxygen/user-guide/openflow-plugin-project-user-guide.html)

[组件说明](https://docs.opendaylight.org/en/stable-oxygen/release-notes/projects/openflowplugin.html)

```bash
feature:install odl-openflowplugin-app-lldp-speaker
feature:install odl-openflowplugin-flow-services-rest
feature:install odl-openflowplugin-drop-test
```

## ovsdb

和ovs交互，需要安装相关组件,[文档地址](https://docs.opendaylight.org/en/stable-oxygen/user-guide/ovsdb-user-guide.html)

[组件说明](https://docs.opendaylight.org/en/stable-oxygen/release-notes/projects/ovsdb.html#odl-ovsdb-southbound-impl)

```bash
feature:install odl-ovsdb-southbound-api
feature:install odl-ovsdb-southbound-impl
feature:install odl-ovsdb-southbound-impl-ui
```

## OpFlex

OpFlex agent-ovs 是一个ovs agent。[项目地址](https://docs.opendaylight.org/en/stable-oxygen/user-guide/opflex-agent-ovs-user-guide.html)

## P4 

P4一种数据平面编程语言。[项目地址](https://docs.opendaylight.org/en/stable-oxygen/user-guide/p4plugin-user-guide.html)

## PacketCable

PacketCable用来做qos之类的。[项目地址](https://docs.opendaylight.org/en/stable-oxygen/user-guide/packetcable-user-guide.html)

## PCEP

OpenDaylight Path Computation Element Configuration Protocol.用于mpls网络中。[项目地址](https://docs.opendaylight.org/en/stable-oxygen/user-guide/bgpcep-guide/pcep/pcep-user-guide-overview.html)

## SFC

Service Function Chaining 组件。[项目地址](https://docs.opendaylight.org/en/stable-oxygen/user-guide/service-function-chaining.html)

## SNMP

SNMP网络管理协议插件。[项目地址](https://docs.opendaylight.org/en/stable-oxygen/user-guide/snmp-plugin-user-guide.html)

## SNMP4SDN

一个southbound插件，用来接管off-the-shelf commodity以太网交换机。[项目地址](https://docs.opendaylight.org/en/stable-oxygen/user-guide/snmp4sdn-user-guide.html)

## SXP

Scalable-Group Tag eXchange Protocol安全组交换协议.[项目地址](https://www.cisco.com/c/en/us/td/docs/switches/lan/trustsec/configuration/guide/trustsec/sxp_config.html)

## TSDR

Time Series Data Repository 框架在odl中收集，存储，查询时间序列数据。[项目地址](https://docs.opendaylight.org/en/stable-oxygen/user-guide/tsdr-user-guide.html)

## TTP

Table Type Patterns用来开启openflow中的一些选项。[项目地址](https://docs.opendaylight.org/en/stable-oxygen/user-guide/ttp-cli-tools-user-guide.html)

## Unimgr

User Network Interface Manager 用来进行服务管理。[项目地址](https://docs.opendaylight.org/en/stable-oxygen/user-guide/uni-manager-plug-in-project.html)

## USC

Unified Secure Channel用于远程通信加密的组件。[项目地址](https://docs.opendaylight.org/en/stable-oxygen/user-guide/unified-secure-channel.html)

## VTN

Virtual Tenant Network vpc组件。[项目地址](https://docs.opendaylight.org/en/stable-oxygen/user-guide/virtual-tenant-network-(vtn).html)

vtn是对使用openflow交换机的物理网络的接管，通过对交换机、端口、mac、vlan等映射来组件逻辑拓扑，降低网络资源管理复杂度。对于北向的诸多功能还需要其他组件，比如dhcp功能，vtn只提供了dhcp中继功能，acl控制也实现不是很方便。

# postman

## 需要添加认证信息

![passs](https://qiniu.li-rui.top/passs.png)

## get请求

### get请求头信息

```bash
Accept: application/xml
```

### 查询流表url

```bash
http://10.220.129.158:8181/restconf/operational/opendaylight-inventory:nodes/node/openflow:1
```

## post请求信息

###　url

```bash
http://localhost:8181/restconf/config/opendaylight-inventory:nodes/node/openflow:1/table/0/flow/1
```

### 请求头

```bash
Accept: application/xml
Content-type: application/xml
```
### 请求体

```xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<flow xmlns="urn:opendaylight:flow:inventory">
    <strict>false</strict>
    <instructions>
        <instruction>
            <order>0</order>
            <apply-actions>
                <action>
                    <order>0</order>
                    <dec-mpls-ttl/>
                </action>
            </apply-actions>
        </instruction>
    </instructions>
    <table_id>0</table_id>
    <id>1</id>
    <cookie_mask>255</cookie_mask>
    <installHw>false</installHw>
    <match>
        <ethernet-match>
            <ethernet-type>
                <type>0x8847</type>
            </ethernet-type>
            <ethernet-destination>
                <address>f4:ce:46:bf:32:f3</address>
            </ethernet-destination>
            <ethernet-source>
                <address>d8:d3:85:5b:ad:89</address>
            </ethernet-source>
        </ethernet-match>
    </match>
    <hard-timeout>12</hard-timeout>
    <cookie>4</cookie>
    <idle-timeout>34</idle-timeout>
    <flow-name>MYMACFLOW</flow-name>
    <priority>2</priority>
    <barrier>false</barrier>
</flow> 
```

## api文档

主要使用了swagger。需要登陆

```bash
http://10.220.129.158:8181/apidoc/explorer/index.html
```








