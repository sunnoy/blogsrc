---
title: ovn子网以及三层网关
date: 2018-12-16 12:12:28
tags:
- openflow
---

# 总拓扑

![ovn](https://qiniu.li-rui.top/ovn.png)

# 控制节点

## 启动

```bash
systemctl start ovn-northd openvswitch
```
<!--more-->

## 创建连接

```bash
ovn-nbctl set-connection ptcp:6641:172.16.1.102
ovn-sbctl set-connection ptcp:6642:172.16.1.102
```

# 计算节点

## 启动服务

```bash
systemctl start ovn-controller openvswitch
```

## 创建br-int网桥

```bash
ovs-vsctl add-br br-int -- set Bridge br-int fail-mode=secure
```

## 和控制端连接

### 配置控制端连接

在ovsdb数据库Open_vSwitch表中写入数据
ovn-encap-ip为和控制端同网段IP

```bash
#第一台
ovs-vsctl set Open_vSwitch . external-ids:ovn-remote=tcp:172.16.1.102:6642
ovs-vsctl set Open_vSwitch . external-ids:ovn-encap-type=geneve
ovs-vsctl set Open_vSwitch . external-ids:ovn-encap-ip=172.16.1.101
#第二台
ovs-vsctl set Open_vSwitch . external-ids:ovn-remote=tcp:172.16.1.102:6642
ovs-vsctl set Open_vSwitch . external-ids:ovn-encap-type=geneve
ovs-vsctl set Open_vSwitch . external-ids:ovn-encap-ip=172.16.1.103
```

# 控制节点查看

### 查看端口监听

```bash
[root@matrix_02 openvswitch]# netstat -anp | grep 664
tcp        0      0 172.16.1.102:6641       0.0.0.0:*               LISTEN      1584905/ovsdb-serve
tcp        0      0 172.16.1.102:6642       0.0.0.0:*               LISTEN      1584913/ovsdb-serve
tcp        0      0 172.16.1.102:6642       172.16.1.101:52936      ESTABLISHED 1584913/ovsdb-serve
tcp        0      0 172.16.1.102:6642       172.16.1.103:48560      ESTABLISHED 1584913/ovsdb-serve

```

### 查看sbdb信息

```bash
[root@matrix_02 openvswitch]# ovn-sbctl show
Chassis "fbf35122-d736-4293-a835-ce6c0bd07261"
    hostname: "matrix_01"
    Encap geneve
        ip: "172.16.1.101"
        options: {csum="true"}
Chassis "f75f6ead-80b2-4225-a898-cd911d0a0f59"
    hostname: "matrix_03"
    Encap geneve
        ip: "172.16.1.103"
        options: {csum="true"}

```

# 拓扑创建

## 创建子网

一个租户

- user1

两个子网

- vpc1 172.66.1.0/24 vm1 vm3 
- vpc2 172.77.1.0/24 vm2 vm4 

四个虚拟机

- vm1 vm2 在101上
- vm3 vm4 在103上


## 创建组件

### 创建子网交换机与路由

```bash
ovn-nbctl lr-add user1
ovn-nbctl ls-add vpc1
ovn-nbctl ls-add vpc2
```

### vpc1连接user1

路由这边

```bash
#创建路由连接到vpc1端口，并分配mac 02:ac:10:ff:34:01 IP 172.66.1.10
ovn-nbctl lrp-add user1 user1-vpc1 02:ac:10:ff:34:01 172.66.1.10/24
```
交换机就这边

```bash
ovn-nbctl lsp-add vpc1 vpc1-user1
ovn-nbctl lsp-set-type vpc1-user1 router
ovn-nbctl lsp-set-addresses vpc1-user1 02:ac:10:ff:34:01
ovn-nbctl lsp-set-options vpc1-user1 router-port=user1-vpc1
```

### vpc2连接user1

路由这边

```bash
#创建路由连接到vpc2端口，并分配mac 02:ac:10:ff:34:02 IP 172.77.1.10
ovn-nbctl lrp-add user1 user1-vpc2 02:ac:10:ff:34:02 172.77.1.10/24
```

交换机这边

```bash
ovn-nbctl lsp-add vpc2 vpc2-user1
ovn-nbctl lsp-set-type vpc2-user1 router
ovn-nbctl lsp-set-addresses vpc2-user1 02:ac:10:ff:34:02
ovn-nbctl lsp-set-options vpc2-user1 router-port=user1-vpc2
```

## 为虚拟机创建端口

创建虚拟机所在的交换机的端口同时指定IP和mac，并开启端口安全

### vpc1

vm1

```bash
ovn-nbctl lsp-add vpc1 vpc1-vm1
ovn-nbctl lsp-set-addresses vpc1-vm1 "02:ac:10:ff:01:30 172.66.1.101"
ovn-nbctl lsp-set-port-security vpc1-vm1 "02:ac:10:ff:01:30 172.66.1.101"
```

vm3

```bash
ovn-nbctl lsp-add vpc1 vpc1-vm3
ovn-nbctl lsp-set-addresses vpc1-vm3 "02:ac:10:ff:01:33 172.66.1.103"
ovn-nbctl lsp-set-port-security vpc1-vm3 "02:ac:10:ff:01:33 172.66.1.103"
```

### vpc2

vm2

```bash
ovn-nbctl lsp-add vpc2 vpc2-vm2
ovn-nbctl lsp-set-addresses vpc2-vm2 "02:ac:10:ff:01:22 172.77.1.102"
ovn-nbctl lsp-set-port-security vpc2-vm2 "02:ac:10:ff:01:22 172.77.1.102"
```

vm4

```bash
ovn-nbctl lsp-add vpc2 vpc2-vm4
ovn-nbctl lsp-set-addresses vpc2-vm4 "02:ac:10:ff:01:44 172.77.1.104"
ovn-nbctl lsp-set-port-security vpc2-vm4 "02:ac:10:ff:01:44 172.77.1.104"
```

## 拓扑查看

```bash
[root@matrix_02 openvswitch]# ovn-nbctl show
switch 27c7499b-c42b-4422-9ca8-359b5d33e249 (vpc1)
    port vpc1-user1
        type: router
        addresses: ["02:ac:10:ff:34:01"]
        router-port: user1-vpc1
    port vpc1-vm1
        addresses: ["02:ac:10:ff:01:30 172.66.1.101"]
    port vpc1-vm3
        addresses: ["02:ac:10:ff:01:33 172.66.1.103"]
switch 1a402073-2ba3-4a1f-a6ee-65cf80d64a93 (vpc2)
    port vpc2-vm2
        addresses: ["02:ac:10:ff:01:22 172.77.1.102"]
    port vpc2-vm4
        addresses: ["02:ac:10:ff:01:44 172.77.1.104"]
    port vpc2-user1
        type: router
        addresses: ["02:ac:10:ff:34:02"]
        router-port: user1-vpc2
router 502be54f-87bd-4e3a-bf88-390eaa29c531 (user1)
    port user1-vpc2
        mac: "02:ac:10:ff:34:02"
        networks: ["172.77.1.10/24"]
    port user1-vpc1
        mac: "02:ac:10:ff:34:01"
        networks: ["172.66.1.10/24"]
```

## 为vpc创建dhcp选项

server_id和router均为交换机在路由器上端口IP，server_mac为交换机在路由器上的端口mac

### vpc1

创建选项

```bash
ovn-nbctl create DHCP_Options cidr=172.66.1.0/24 \
options="\"server_id\"=\"172.66.1.10\" \"server_mac\"=\"02:ac:10:ff:34:01\" \
\"lease_time\"=\"3600\" \"router\"=\"172.66.1.10\""

#返回选项id
e354c0ff-846e-4e7e-af23-28b5f9b569d6
```

对虚拟机端口应用选项

```bash
ovn-nbctl lsp-set-dhcpv4-options vpc1-vm1 e354c0ff-846e-4e7e-af23-28b5f9b569d6
ovn-nbctl lsp-set-dhcpv4-options vpc1-vm3 e354c0ff-846e-4e7e-af23-28b5f9b569d6
```

查看选项

```bash
ovn-nbctl lsp-get-dhcpv4-options vpc1-vm1
ovn-nbctl lsp-get-dhcpv4-options vpc1-vm3

#返回选项
e354c0ff-846e-4e7e-af23-28b5f9b569d6 (172.66.1.0/24)
e354c0ff-846e-4e7e-af23-28b5f9b569d6 (172.66.1.0/24)
```

### vpc2

创建选项

```bash
ovn-nbctl create DHCP_Options cidr=172.77.1.0/24 \
options="\"server_id\"=\"172.77.1.10\" \"server_mac\"=\"02:ac:10:ff:34:02\" \
\"lease_time\"=\"3600\" \"router\"=\"172.77.1.10\""

#返回选项id
997e31dd-0d11-410c-9455-9c775085bed1
```

对虚拟机端口应用选项

```bash
ovn-nbctl lsp-set-dhcpv4-options vpc2-vm2 997e31dd-0d11-410c-9455-9c775085bed1
ovn-nbctl lsp-set-dhcpv4-options vpc2-vm4 997e31dd-0d11-410c-9455-9c775085bed1
```

查看选项

```bash
ovn-nbctl lsp-get-dhcpv4-options vpc2-vm2
ovn-nbctl lsp-get-dhcpv4-options vpc2-vm4

#返回选项
997e31dd-0d11-410c-9455-9c775085bed1 (172.77.1.0/24)
997e31dd-0d11-410c-9455-9c775085bed1 (172.77.1.0/24)
```

## 端口查看

```bash
Chassis "fbf35122-d736-4293-a835-ce6c0bd07261"
    hostname: "matrix_01"
    Encap geneve
        ip: "172.16.1.101"
        options: {csum="true"}
    Port_Binding "vpc1-vm1"
    Port_Binding "vpc2-vm2"
Chassis "f75f6ead-80b2-4225-a898-cd911d0a0f59"
    hostname: "matrix_03"
    Encap geneve
        ip: "172.16.1.103"
        options: {csum="true"}
    Port_Binding "vpc1-vm3"
    Port_Binding "vpc2-vm4"

```

# 虚拟机创建

## 101上

创建vm1 vm2

```bash
ip netns add vm1
ovs-vsctl add-port br-int vm1 -- set interface vm1 type=internal
ip link set vm1 address 02:ac:10:ff:01:30
ip link set vm1 netns vm1
ovs-vsctl set Interface vm1 external_ids:iface-id=vpc1-vm1
ip netns exec vm1 dhclient vm1
ip netns exec vm1 ip addr show vm1
ip netns exec vm1 ip route show

ip netns add vm2
ovs-vsctl add-port br-int vm2 -- set interface vm2 type=internal
ip link set vm2 address 02:ac:10:ff:01:22
ip link set vm2 netns vm2
ovs-vsctl set Interface vm2 external_ids:iface-id=vpc2-vm2
ip netns exec vm2 dhclient vm2
ip netns exec vm2 ip addr show vm2
ip netns exec vm2 ip route show
```

## 103上

创建vm3 vm4

```bash
ip netns add vm3
ovs-vsctl add-port br-int vm3 -- set interface vm3 type=internal
ip link set vm3 address 02:ac:10:ff:01:33
ip link set vm3 netns vm3
ovs-vsctl set Interface vm3 external_ids:iface-id=vpc1-vm3
ip netns exec vm3 dhclient vm3
ip netns exec vm3 ip addr show vm3
ip netns exec vm3 ip route show

ip netns add vm4
ovs-vsctl add-port br-int vm4 -- set interface vm4 type=internal
ip link set vm4 address 02:ac:10:ff:01:44
ip link set vm4 netns vm4
ovs-vsctl set Interface vm4 external_ids:iface-id=vpc2-vm4
ip netns exec vm4 dhclient vm4
ip netns exec vm4 ip addr show vm4
ip netns exec vm4 ip route show
```

# 测试

## 测试vpc1

在103上执行

```bash
ip netns exec vm3 ping 172.66.1.101
```

在101上查看

```bash
[root@matrix_01 ~]# ip netns exec vm1 arp -n
Address                  HWtype  HWaddress           Flags Mask            Iface
172.66.1.103             ether   02:ac:10:ff:01:33   C                  
```

## 测试vpc2

在103上执行

```bash
ip netns exec vm4 ping 172.77.1.102
```

在101上查看

```bash
[root@matrix_01 ~]# ip netns exec vm2 arp -n
Address                  HWtype  HWaddress           Flags Mask            Iface
172.77.1.104             ether   02:ac:10:ff:01:44   C        
```

## 测试vpc1和vpc2之间

在103上执行

```bash
ip netns exec vm3 ping 172.77.1.102
```

在101上查看

```bash
[root@matrix_01 ~]# ip netns exec vm2 arp -n
Address                  HWtype  HWaddress           Flags Mask            Iface
172.77.1.104             ether   02:ac:10:ff:01:44   C                     vm2
172.77.1.10              ether   02:ac:10:ff:34:02   C                     vm2
```
路由器内不同端口之间是可以路由的

# 路由操作

## 静态路由

加入错误路由

```bash
ovn-nbctl  lr-route-add user1 172.77.1.0/24 172.66.10
```

在103上测试，发现不通

```bash
ip netns exec vm3 ping 172.77.1.102
```

删除错误路由

```bash
ovn-nbctl  lr-route-del user1 172.77.1.0/24
#查看路由
ovn-nbctl lr-route-list user1
```

在103上测试，发现通

```bash
ip netns exec vm3 ping 172.77.1.102
```

## 路由端口状态

禁用vpc2路由端口

```bash
#状态为enabled disabled
ovn-nbctl lrp-set-enabled user1-vpc2 disabled

#获取状态
ovn-nbctl lrp-get-enabled user1-vpc2
```

测试vpc内，通

```bash
[root@matrix_03 ~]# ip netns exec vm4 ping 172.77.1.102
PING 172.77.1.102 (172.77.1.102) 56(84) bytes of data.
64 bytes from 172.77.1.102: icmp_seq=1 ttl=64 time=1.42 ms
64 bytes from 172.77.1.102: icmp_seq=2 ttl=64 time=0.314 ms

[root@matrix_03 ~]# ip netns exec vm3 ping 172.66.1.101
PING 172.66.1.101 (172.66.1.101) 56(84) bytes of data.
64 bytes from 172.66.1.101: icmp_seq=1 ttl=64 time=1.39 ms
64 bytes from 172.66.1.101: icmp_seq=2 ttl=64 time=0.262 ms

```

测试vpc间，不通

```bash
[root@matrix_03 ~]# ip netns exec vm3 ping 172.77.1.102
PING 172.77.1.102 (172.77.1.102) 56(84) bytes of data.
^C
--- 172.77.1.102 ping statistics ---
1 packets transmitted, 0 received, 100% packet loss, time 0ms

```

启用vpc2端口

```bash
ovn-nbctl lrp-set-enabled user1-vpc2 enabled
```

测试vpc间，通

```bash
[root@matrix_03 ~]# ip netns exec vm3 ping 172.77.1.102
PING 172.77.1.102 (172.77.1.102) 56(84) bytes of data.
64 bytes from 172.77.1.102: icmp_seq=1 ttl=63 time=1.84 ms
^C
--- 172.77.1.102 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 1.844/1.844/1.844/0.000 ms

```

可见，可以使用路由上子网交换机端口状态来配置租户的子网之间通信状态

# ovn L3网关

三层网关需要在特定主机上，并且三层网关和物理网卡和路由连接需要中间交换机作为媒介，这里我们设置为102，确保`openvswitch-ovn-host`已经安装，并且`systemctl start ovn-controller`已经启动

## 102配置

### 创建br-int

```bash
ovs-vsctl set Open_vSwitch . external-ids:ovn-remote=tcp:172.16.1.102:6642
ovs-vsctl set Open_vSwitch . external-ids:ovn-encap-type=geneve
ovs-vsctl set Open_vSwitch . external-ids:ovn-encap-ip=172.16.1.102
ovs-vsctl add-br br-int -- set Bridge br-int fail-mode=secure

```

### 状态查看

```bash
[root@matrix_02 openvswitch]# netstat -anp | grep 664
tcp        0      0 172.16.1.102:6641       0.0.0.0:*               LISTEN      1584905/ovsdb-serve
tcp        0      0 172.16.1.102:6642       0.0.0.0:*               LISTEN      1584913/ovsdb-serve
tcp        0      0 172.16.1.102:6642       172.16.1.102:53268      ESTABLISHED 1584913/ovsdb-serve
tcp        0      0 172.16.1.102:53268      172.16.1.102:6642       ESTABLISHED 1595069/ovn-control
tcp        0      0 172.16.1.102:6642       172.16.1.101:52936      ESTABLISHED 1584913/ovsdb-serve
tcp        0      0 172.16.1.102:6642       172.16.1.103:48560      ESTABLISHED 1584913/ovsdb-serve
```

### 获取102Chassis uuid

1859fa77-e381-405c-b08b-ef7029658426

```bash
[root@matrix_02 openvswitch]# ovn-sbctl show
Chassis "fbf35122-d736-4293-a835-ce6c0bd07261"
    hostname: "matrix_01"
    Encap geneve
        ip: "172.16.1.101"
        options: {csum="true"}
    Port_Binding "vpc1-vm1"
    Port_Binding "vpc2-vm2"
Chassis "1859fa77-e381-405c-b08b-ef7029658426"
    hostname: "matrix_02"
    Encap geneve
        ip: "172.16.1.102"
        options: {csum="true"}
Chassis "f75f6ead-80b2-4225-a898-cd911d0a0f59"
    hostname: "matrix_03"
    Encap geneve
        ip: "172.16.1.103"
        options: {csum="true"}
    Port_Binding "vpc1-vm3"
    Port_Binding "vpc2-vm4"

```

## 添加l3网关

### 创建l3网关路由

```bash
ovn-nbctl create Logical_Router name=gateway_route options:chassis=1859fa77-e381-405c-b08b-ef7029658426
```

### 创建交换机连接gateway_route和user1

```bash
ovn-nbctl ls-add transit

#gateway_route路由增加transit端口
ovn-nbctl lrp-add gateway_route gateway_transit 02:ac:10:ff:00:01 172.88.1.10/24

#transit交换机连接路由gateway_route
ovn-nbctl lsp-add transit transit-gateway
ovn-nbctl lsp-set-type transit-gateway router
ovn-nbctl lsp-set-addresses transit-gateway 02:ac:10:ff:00:01
ovn-nbctl lsp-set-options transit-gateway router-port=gateway_transit


#user1和transit连接
ovn-nbctl lrp-add user1 user1-transit 02:ac:10:ff:00:02 172.88.1.20/24

ovn-nbctl lsp-add transit transit-user1
ovn-nbctl lsp-set-type transit-user1 router
ovn-nbctl lsp-set-addresses transit-user1 02:ac:10:ff:00:02
ovn-nbctl lsp-set-options transit-user1 router-port=user1-transit
```

### 静态路由添加

需要vm1-4到gateway_route上端口172.88.1.10通。注意添加去包路由和回包路由，transit可认为是线缆。

主要是在路由器gateway_route和user1之间添加互通路由

#### 对user1

需要添加默认网关到172.88.1.10，不是20，20本来就通，其实是本地接口

```bash
ovn-nbctl lr-route-add user1 "0.0.0.0/0" 172.88.1.10
```

#### 对gateway_route

重点添加回包路由，对于到两个子网的包都从user1上的口出

```bash
ovn-nbctl lr-route-add gateway_route "172.77.1.0/24" 172.88.1.20
ovn-nbctl lr-route-add gateway_route "172.66.1.0/24" 172.88.1.20
```

### 测试

两个vpc到172.88.1.10

```bash
#vpc1
[root@matrix_03 ~]# ip netns exec vm4 ping 172.88.1.10
PING 172.88.1.10 (172.88.1.10) 56(84) bytes of data.
64 bytes from 172.88.1.10: icmp_seq=1 ttl=253 time=1.46 ms
64 bytes from 172.88.1.10: icmp_seq=2 ttl=253 time=0.607 ms
^C
--- 172.88.1.10 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 0.607/1.036/1.466/0.430 ms

#vpc2
[root@matrix_03 ~]# ip netns exec vm3 ping 172.88.1.10
PING 172.88.1.10 (172.88.1.10) 56(84) bytes of data.
64 bytes from 172.88.1.10: icmp_seq=1 ttl=253 time=1.73 ms
64 bytes from 172.88.1.10: icmp_seq=2 ttl=253 time=0.635 ms
^C
--- 172.88.1.10 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 0.635/1.185/1.735/0.550 ms

```

### 创建连接外网交换机

#### gateway_route添加端口

主要是添加到外网段内的端口以及IP

```bash
ovn-nbctl lrp-add gateway_route gateway_out 02:0a:7f:00:01:88 192.168.66.45/23 
```

#### 添加out交换机

out交换机主要作为gateway路由和外网网桥的线缆

```bash
ovn-nbctl ls-add out
ovn-nbctl lsp-add out out-gateway
ovn-nbctl lsp-set-type out-gateway router
ovn-nbctl lsp-set-addresses out-gateway 02:0a:7f:00:01:88
ovn-nbctl lsp-set-options out-gateway router-port=gateway_out
```

#### 添加外网网桥

外网网桥主要和out交换机相连

创建交换机out上端口，用于和外网相连

addresses 为unknown是因为所连的br-em1上有mac，避免冲突，端口类型设置为localnet需要相应的选项(不同端口类型的必须选项不同)network_name。

```bash
ovn-nbctl lsp-add out outs-wan
ovn-nbctl lsp-set-addresses outs-wan unknown
ovn-nbctl lsp-set-type outs-wan localnet
ovn-nbctl lsp-set-options outs-wan network_name=wanNet
```

创建网桥

```bash
ovs-vsctl add-br br-em1
ovs-vsctl set Open_vSwitch . external-ids:ovn-bridge-mappings=wanNet:br-em1
ovs-vsctl add-port br-em1 em1

#配置网桥IP
ip link set br-em1 up
ip addr add 192.168.66.111/23 dev br-em1
```

测试，在111上连接45

```bash
[root@matrix_02 ~]# ping 192.168.66.45
PING 192.168.66.45 (192.168.66.45) 56(84) bytes of data.
64 bytes from 192.168.66.45: icmp_seq=1 ttl=254 time=0.645 ms
64 bytes from 192.168.66.45: icmp_seq=2 ttl=254 time=0.309 ms
64 bytes from 192.168.66.45: icmp_seq=3 ttl=254 time=0.340 ms
^C
--- 192.168.66.45 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2065ms
rtt min/avg/max/mdev = 0.309/0.431/0.645/0.152 ms
[root@matrix_02 ~]# arp -n | grep 66.45
192.168.66.45                    (incomplete)                              em1
192.168.66.45            ether   02:0a:7f:00:01:88   C                     br-em1

```

### 处理L3网关nat

此处L3网关相当于路由器，需要做IP分享，vm需要从这里出去到外网，然后回来，因此需要配置SNAT

```bash
#对vpc1
ovn-nbctl -- --id=@nat create nat type="snat" logical_ip=172.66.1.0/24 \
external_ip=192.168.66.45 -- add logical_router gateway_route nat @nat
#会返回uuid
56ad6c5b-8417-4314-95c4-a0d780b5ef0b

#对vpc2
ovn-nbctl -- --id=@nat create nat type="snat" logical_ip=172.77.1.0/24 \
external_ip=192.168.66.45 -- add logical_router gateway_route nat @nat
#会返回uuid
ca82de02-f527-480a-9af7-5261517bab37
```

也可以对vm绑定外网地址

```bash
#对vm3 172.66.1.103  绑定外网 192.168.66.46 
ovn-nbctl -- --id=@nat create nat type="dnat_and_snat" logical_ip=172.66.1.103 \
external_ip=192.168.66.46 -- add logical_router gateway_route nat @nat
```

测试

```bash
[root@matrix_03 ~]# ip netns exec vm4 ping 192.168.66.112
PING 192.168.66.112 (192.168.66.112) 56(84) bytes of data.
64 bytes from 192.168.66.112: icmp_seq=1 ttl=62 time=2.57 ms
64 bytes from 192.168.66.112: icmp_seq=2 ttl=62 time=0.362 ms
^C
--- 192.168.66.112 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 0.362/1.468/2.574/1.106 ms


[root@matrix_03 ~]# ip netns exec vm3 ping 192.168.66.112
PING 192.168.66.112 (192.168.66.112) 56(84) bytes of data.
64 bytes from 192.168.66.112: icmp_seq=1 ttl=62 time=2.13 ms
64 bytes from 192.168.66.112: icmp_seq=2 ttl=62 time=0.636 ms
^C
--- 192.168.66.112 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 0.636/1.385/2.135/0.750 ms

#ping网关192.168.66.1
[root@matrix_03 ~]# ip netns exec vm3 ping 192.168.66.1
PING 192.168.66.1 (192.168.66.1) 56(84) bytes of data.
64 bytes from 192.168.66.1: icmp_seq=1 ttl=62 time=2.72 ms
64 bytes from 192.168.66.1: icmp_seq=2 ttl=62 time=0.578 ms
^C
--- 192.168.66.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1003ms
rtt min/avg/max/mdev = 0.578/1.649/2.721/1.072 ms
[root@matrix_03 ~]# ip netns exec vm4 ping 192.168.66.1
PING 192.168.66.1 (192.168.66.1) 56(84) bytes of data.
64 bytes from 192.168.66.1: icmp_seq=1 ttl=62 time=2.55 ms
64 bytes from 192.168.66.1: icmp_seq=2 ttl=62 time=0.752 ms
^C
--- 192.168.66.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 0.752/1.654/2.557/0.903 ms
```

### 当前拓扑

```bash
switch a1395d52-e659-47f7-a217-042004a2e324 (transit)
    port transit-user1
        type: router
        addresses: ["02:ac:10:ff:00:02"]
        router-port: user1-transit
    port transit-gateway
        type: router
        addresses: ["02:ac:10:ff:00:01"]
        router-port: gateway_transit
switch 1a402073-2ba3-4a1f-a6ee-65cf80d64a93 (vpc2)
    port vpc2-vm2
        addresses: ["02:ac:10:ff:01:22 172.77.1.102"]
    port vpc2-vm4
        addresses: ["02:ac:10:ff:01:44 172.77.1.104"]
    port vpc2-user1
        type: router
        addresses: ["02:ac:10:ff:34:02"]
        router-port: user1-vpc2
switch f812571b-ac0d-464a-9fbb-a7fbf436c481 (out)
    port out-gateway
        type: router
        addresses: ["02:0a:7f:00:01:88"]
        router-port: gateway_out
    port outs-wan
        type: localnet
        addresses: ["unknown"]
switch 27c7499b-c42b-4422-9ca8-359b5d33e249 (vpc1)
    port vpc1-user1
        type: router
        addresses: ["02:ac:10:ff:34:01"]
        router-port: user1-vpc1
    port vpc1-vm1
        addresses: ["02:ac:10:ff:01:30 172.66.1.101"]
    port vpc1-vm3
        addresses: ["02:ac:10:ff:01:33 172.66.1.103"]
router 5d07c35c-f2d4-4339-b618-83ad2a09f704 (gateway_route)
    port gateway_out
        mac: "02:0a:7f:00:01:88"
        networks: ["192.168.66.45/23"]
    port gateway_transit
        mac: "02:ac:10:ff:00:01"
        networks: ["172.88.1.10/24"]
    nat 56ad6c5b-8417-4314-95c4-a0d780b5ef0b
        external ip: "192.168.66.45"
        logical ip: "172.66.1.0/24"
        type: "snat"
    nat ca82de02-f527-480a-9af7-5261517bab37
        external ip: "192.168.66.45"
        logical ip: "172.77.1.0/24"
        type: "snat"
router 502be54f-87bd-4e3a-bf88-390eaa29c531 (user1)
    port user1-vpc2
        mac: "02:ac:10:ff:34:02"
        networks: ["172.77.1.10/24"]
    port user1-transit
        mac: "02:ac:10:ff:00:02"
        networks: ["172.88.1.20/24"]
    port user1-vpc1
        mac: "02:ac:10:ff:34:01"
        networks: ["172.66.1.10/24"]

```

# ACL

- 优先级0-32767
- 方向为 from-lport to-lport
- 匹配规则和ovs中流表一样，其中outport仅在to-lport方向，inport双向都可以用
- 动作为 allow-related, allow, drop, or reject

## ip层

```bash
ovn-nbctl acl-add vpc1 from-lport 1000 "inport == \"dmz-vm1\" && ip" allow-related

#对交换机vpc1 方向为目的端口  优先级为900 
ovn-nbctl acl-add vpc1 to-lport 900 "outport == \"vpc1-vm3\" && ip" drop

ovn-nbctl acl-add vpc1 to-lport 901 "outport == \"vpc1-vm3\" && ip" allow-related
```

## 服务层

安装监听工具

```bash
yum install nmap-ncat

#监听tcp
ncat -l 8080

#监听udp
ncat -l -u 1234
```

### 配置示例

仅仅开放80端口，其他都drop掉

```bash
#创建一个地址集合
ovn-nbctl create Address_Set name=app addresses=\"172.16.255.130/31\"

#优先通过web通过
ovn-nbctl acl-add vpc1 to-lport 1000 'outport == "vpc1-vm3" && tcp.dst == 80' allow-related

#其他统统扔掉
ovn-nbctl acl-add vpc1 to-lport 900 'outport == "vpc1-vm3" && ip' drop
```



