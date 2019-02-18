---
title: ovs-vsctl使用
date: 2018-12-03 12:14:28
tags:
- openflow
---

# ovs-vsctl使用

ovs使用分为两类

- 通过命令来直接操作ovs
- 使用控制器通过openflow协议或其他协议去远程操作ovs

<!--more-->

# 命令操作

[ovs命令文档](http://www.openvswitch.org/support/dist-docs/)

# 常用命令

![ovs](https://qiniu.li-rui.top/ovszjjg.png)

该命令通过ovsdb-server来配置ovs-vswitchd，下面根据[官方文档](http://www.openvswitch.org/support/dist-docs/ovs-vsctl.8.txt)整理

### 整体配置

```bash
ovs-vsctl 
#指定数据库连接。tcp:ip:port unix:file 等可选
--db=tcp:ip:port
#ovs-vswitchd没有运行的时候启用
--no-wait
```

### 输出格式

```bash
#默认 list，可选html json
ovs-vsctl  -f format
```

### Open vSwitch命令

```bash
#初始化数据库
ovs-vsctl init
#显示数据库信息概览
ovs-vsctl show
#重置配置
ovs-vsctl emer-reset
```

### 桥命令

网桥分为
 - real 网桥
 - faker 网桥，faker网桥依附在real网桥上

```bash
#创建网桥bridge
#--may-exist 如果该网桥存在就啥都不做，不加该选项就报错
ovs-vsctl [--may-exist] add-br bridge

#创建faker bridge，libvirt不可以直接使用ovs的网桥，可以创建一个faker来和libvirt相连
#vlan为 0  到  4095
ovs-vsctl add-br <fake bridge> <parent bridge> <VLAN>  

#删除网桥bridge
#--if-exists存在时网桥不存在不报错
#如果是real网桥会删除它和它的所以faker网桥
ovs-vsctl [--if-exists] del-br bridge

#显示网桥
#--real --fake来选择不同类型的网桥
ovs-vsctl [--real|--fake] list-br

#查看网桥是否存在
#存在命令$?为0 否则为2
ovs-vsctl br-exists dong

#查看网桥是否是faker网桥
#是就返回vlan id 否在返回0
ovs-vsctl br-to-vlan bridge

#查看是否是real网桥
#是就返回real网桥名称 否则就返回faker的real网桥
ovs-vsctl br-to-parent bridge
```

### port命令

```bash
#添加端口
#网桥为dong 端口名称为eth0，要想使用就要和网络设备名称相同
#后面tag=9 需要时可以加
ovs-vsctl add-port dong eth0 [tag=9]

#显示网桥端口
ovs-vsctl list-ports dong

#添加链路聚合端口
ovs-vsctl add-bond bridge eth0 eth1 ... 

#删除端口
#bridge不加就会删除所有该名称的端口，否则只删除该网桥上的端口
ovs-vsctl [--if-exists] del-port [bridge] port

#查看端口所在的网桥
ovs-vsctl port-to-br vvs

#创建一个网桥同时增加端口
ovs-vsctl add-br br0 -- add-port br0 eth0
```

### Interface命令

```bash
#查看该桥上的接口
ovs-vsctl list-ifaces dong

#查看该接口的所在的网桥
ovs-vsctl iface-to-br vvs
```

### OpenFlow控制器命令

```bash
#对一个网桥设置控制器
#target可以是 tcp:ip[:port] unix:file 等
ovs-vsctl set-controller bridge target

#查看某个网桥的控制器
ovs-vsctl get-controller bridge

#删除某个网桥的控制器
ovs-vsctl del-controller bridge
```

### 一些复杂操作

```bash
#创建网桥br0 和faker网桥 并设置附加的端口为internal类型 然后配置IP
ovs-vsctl add-port br0 vlan10 tag=10  --  set  Interface  vlan10 type=internal
ip addr add 192.168.0.123/24 dev vlan10

#创建一个gre隧道，并且指定远程IP
ovs-vsctl  add-port  br0  gre0  --  set  Interface gre0 type=gre options:remote_ip=1.2.3.4

#将eth0和eth1的流量都镜像到eth2
#eth2接受的包会被忽略
ovs-vsctl -- set Bridge br0 mirrors=@m \

-- --id=@eth0 get Port eth0 \

-- --id=@eth1 get Port eth1 \

-- --id=@eth2 get Port eth2 \

--   --id=@m create Mirror name=mymirror select-dst-port=@eth0,@eth1 select-src-port=@eth0,@eth1 output-port=@eth2
#删除镜像
ovs-vsctl clear Bridge br0 mirrors

#网桥开启stp
ovs-vsctl set Bridge br0 stp_enable=true
#关闭stp
ovs-vsctl set Bridge br0 stp_enable=false

#开启fstp
ovs-vsctl set Bridge br0 rstp_enable=true
#关闭fstp
ovs-vsctl set Bridge br0 rstp_enable=false

#对br0中的table 0流表中经过的flow限制到100
ovs-vsctl  --  --id=@ft  create  Flow_Table flow_limit=100 over‐flow_policy=refuse -- set Bridge br0 flow_tables=0=@ft
```

### 指定openflow版本

```bash
ovs-vsctl set bridge  br0  protocols=OpenFlow10,OpenFlow12,Open‐Flow13
```







