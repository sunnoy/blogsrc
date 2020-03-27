---
title: opendaylightONV
date: 2018-10-14 12:12:28
tags:
- openflow
---
# OpenDaylight Network Virtualization (ONV)

odl虚拟化网络，类似ovn，流表用的比ovn要多，onv使用的隧道类型为vxlan

<!--more-->

# 连接kvm

## 创建计算节点网桥

包含
- 选取节点上隧道网卡
- 创建网桥ovs-br0（类似ovn中br-int）并添加隧道网卡端口
- 在bridge表中添加隧道信息，区别于ovn是在openvswitcth表上添加隧道信息

```bash
/root/start-ovs.sh
#!/bin/sh
#指配数据平面网卡
iface_data=eth1

#获取网卡地址
test -f /sys/class/net/$iface_data/address
mac=$(cat /sys/class/net/$iface_data/address | tr -d : )

#以及mac地址来创建datapath id
dpid=0000$mac

#创建run路径
mkdir -p /var/run/openvswitch

#启动ovs

#指定数据库
ovsdb-server /etc/openvswitch/conf.db \ 
         #指定连接客户端连系方式punix sock方式
         --remote=punix:/var/run/openvswitch/db.sock \
		 #获取更多的连接方式
         --remote=db:Open_vSwitch,manager_options \
		 #tcp连接方式
         --remote=ptcp:6635 \
		 #后台运行
		 --detach
		 
		 
#初始化ovsdb --no-wait 当ovs-vswitchd没有启动时加上	 
ovs-vsctl --no-wait init

#操作ovsdb数据库Bridge表，以虚拟交换机名称为记录值，配置字段值
#other-config中的字典，供onv使用的标记。
#这是进行与opendaylight进行绑定
ovs-vsctl --no-wait set bridge ovs-br0 \
   other-config:datapath_type=system \
   other-config:datapath-id=$dpid
   
#启动vswitchd  
ovs-vswitchd --pidfile --detach 
#启动brcompatd进程，centos中不需要
#ovs-brcompatd --pidfile --detach

#配置隧道ip，就是配置网桥上的IP
#/root/configure-tunnel-ip.sh

#!/bin/sh
iface_tunnel="ovs-br0"
tunnelip=""
while true; do
    #获取隧道IP
    tunnelip=$(ip addr show $iface_tunnel | \
               awk '$1=="inet" { print $2 }' | \
               cut -d/ -f 1 ) || true
    if [ -n "$tunnelip" ]; then
        break;
    fi
    sleep 1
done
#设定隧道IP，仍然在bridge中的字段other-config中的字典中添加
ovs-vsctl --no-wait set bridge ovs-br0 other-config:tunnel-ip=$tunnelip

```

## 添加控制器

```bash
#ovs添加控制器
ip=173.16.248.153
port=6633
ovs-vsctl set-controller ovs-br0 tcp:$ip:$port

#查看控制器连接
ovs-vsctl list controller 
```

## 虚拟机连接ovs-br0网桥

和ovn中连接一样，直接连接到ovs-br0上

# onv设计架构

## 逻辑拓扑

onv中使用逻辑拓扑来完成网络需求，逻辑拓扑和虚拟拓扑之间可以进行相关类型的映射，包含端口，vlan id等

## 路由分类

![530px-Onv-routing-image2](https://qiniu.li-rui.top/530px-Onv-routing-image2.png)

- external  到外网
- tenant   租户子网之间路由
- system   租户之间路由

# 租户创建

onv主要是租户概念，通过不同租户类型之间的路由来完成整个逻辑拓扑的创建

虚拟拓扑

![Onv-configuration-image1](https://qiniu.li-rui.top/Onv-configuration-image1.png)

## 创建两个租户

### 初始租户id

虚拟拓扑中的端口连上控制器时都在初始`default`的租户id下面

```bash
show onv all mac-address-table

# Tenant  onv     Address Space MAC Address            VLAN IP Address Attachment Point Last Seen
-|-------|-------|-------------|----------------------|----|----------|----------------|---------
1 default default default       00:00:00:00:00:01 (h1)      22.0.0.1   s6/1 (s6-eth1)   0 minute
2 default default default       00:00:00:00:00:02 (h2)      22.0.0.2   s6/2 (s6-eth2)   0 minute
3 default default default       00:00:00:00:00:03 (h3)      22.0.0.3   s7/1 (s7-eth1)   0 minute
4 default default default       00:00:00:00:00:04 (h4)      22.0.0.4   s7/2 (s7-eth2)   0 minute
```

### 创建两个租户以及VNS

![530px-Onv-configuration-image4](https://qiniu.li-rui.top/530px-Onv-configuration-image4.png)

```bash
oscp# conf t
#创建red
oscp(config)# tenant red
#在住户red中创建Virtual Network Segments (VNSes)  devtest-vns
oscpconfig-tenant)# onv-definition devtest-vns
oscp(config-tenant-def-onv)# exit
oscp(config-tenant)# exit
oscp(config)# tenant green
oscp(config-tenant)# onv-definition web-vns
oscp(config-tenant-def-onv)# exit
oscp(config-tenant)# onv-definition app-vns
oscp(config-tenant-def-onv)# exit
```

### 在vns中添加vm

在vns添加vm的过程就是虚拟拓扑于逻辑拓扑映射的过程，onv的映射方式比ovn更灵活

有多种映射规则

- port-based rules (port 42 on switch 3),
- header-based rules (ip-subnet = 22.0.1.0/24),
- meta-data that has been populated from an external source, or
- any combination of the above

相关映射命令

```bash
switch
ports
ip-subnet
mac
vlans
tags
```

### 根据子网动态分配

将h1放入red下的devtest-vns中

```bash
oscp# conf t
#进入red租户配置
oscp(config)# tenant red
#进入red租户中devtest-vns 的vns配置
oscp(config-tenant)# onv-definition devtest-vns
#定义出规则 名称为mac-1
oscp(config-tenant-def-onv)# interface-rule mac-1
#使用match mac命令将h1的mac放入devtest-vns中
oscpconfig-tenant-def-onv-if-rule)# match mac 00:00:00:00:00:01
oscp(config-tenant-def-onv-if-rule)# exit
#仍然在devtest-vns内又添加一条规则 ip-3
oscp(config-tenant-def-onv)# interface-rule ip-3
#添加一个主机IP，主要掩码32位
oscp(config-tenant-def-onv-if-rule)# match ip-subnet 22.0.0.3/32
oscp(config-tenant-def-onv-if-rule)# end
oscp# 
```

添加后效果如下

![530px-Onv-configuration-image5](https://qiniu.li-rui.top/530px-Onv-configuration-image5.png)

### 静态交换机端口添加

```bash
oscp# conf t
oscpconfig)# tenant green
oscp(config-tenant)# onv-definition web-vns
oscpconfig-tenant-def-onv)# interface-rule host-dot-2
#直接将s6 上端口s6-eth2添加到green的web-vns内
oscp(config-tenant-def-onv-if-rule)# match switch s6 s6-eth2
```

### 静态标签添加

```bash
oscp# conf t
#创建标签
oscp(config)# tag default.type=appserver 
#关联标签和h4的mac
oscp(config-tag)# match mac 00:00:00:00:00:04
oscp(config-tag)# exit
oscp(config)# tenant green
oscp(config-tenant)# onv-definition app-vns 
oscp(config-tenant-def-onv)# interface-rule appservers
#直接将标签添加到green的app-vns
oscp(config-tenant-def-onv-if-rule)# match tags type=appserver
oscp(config-tenant-def-onv-if-rule)#
```

最终效果

![530px-Onv-configuration-image7](https://qiniu.li-rui.top/530px-Onv-configuration-image7.png)

# 租户路由

需要四种类型的路由方式

![530px-Onv-concepts-image6](https://qiniu.li-rui.top/530px-Onv-concepts-image6.png)

## 创建路由

- 在租户内创建路由
- 创建路由上的接口
- 配置路由规则