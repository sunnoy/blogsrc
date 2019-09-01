---
title: kubernetes中pod网络实现
date: 2019-08-31 12:12:28
tags:
- kubernetes
---

# pod网络

## 网络模型

- pod之间为二层网络
- node中访问pod不需要nat

<!--more-->

# 环境准备

从node 10.9.1.161 的pod 172.20.1.116 到node 10.9.1.156的pod 172.20.0.7

下面我们主要来看pod之间的流量转发

pod 172.20.1.116 到 pod 172.20.0.7

## pod 172.20.1.116的网络情况


```bash
#网络地址
ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
3: eth0@if397: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default
    link/ether fe:0b:7e:49:95:c0 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.20.1.116/24 scope global eth0
       valid_lft forever preferred_lft forever
#路由
ip route
default via 172.20.1.1 dev eth0
172.20.0.0/16 via 172.20.1.1 dev eth0
172.20.1.0/24 dev eth0 proto kernel scope link src 172.20.1.116

#到172.20.0.7的路由
# 需要通过172.20.1.1来转发
ip route get 172.20.0.7
172.20.0.7 via 172.20.1.1 dev eth0 src 172.20.1.116

```

## node 10.9.1.161上的网络

```bash
ip addr show cni0
7: cni0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP qlen 1000
    link/ether 62:e0:1a:fe:c7:41 brd ff:ff:ff:ff:ff:ff
    inet 172.20.1.1/24 scope global cni0
       valid_lft forever preferred_lft forever

ip addr show flannel.1
6: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN
    link/ether da:8c:e6:f9:b8:dc brd ff:ff:ff:ff:ff:ff
    inet 172.20.1.0/32 scope global flannel.1
       valid_lft forever preferred_lft forever
```

可见 172.20.1.1 是 cni0，node上的网桥，node上的容器都和cni0相连

## flannel.1设备

数据包出了cni0然后会转发到哪里呢？

```bash
ip route get 172.20.0.7
172.20.0.7 via 172.20.0.0 dev flannel.1  src 172.20.1.0
```

flannel.1是vxlan设备，其vlan为1 ，flannel.1同样也是一个网桥设备

```bash
ip -d link show flannel.1
6: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN mode DEFAULT
    link/ether da:8c:e6:f9:b8:dc brd ff:ff:ff:ff:ff:ff promiscuity 0
    vxlan id 1 local 10.9.1.161 dev eth0 srcport 0 0 dstport 8472 nolearning ageing 300 addrgenmode eui64

# fdb用于填充普通以太网内的字段
bridge fdb | grep flannel.1
76:3a:70:b3:0b:99 dev flannel.1 dst 10.9.1.156 self permanent
```

数据包到达flannel.1后就会进行封包，最后通过fdb中的信息转发pod 172.20.0.7所在的远程主机 10.9.1.156

综上使用vxlan的网络模型加入了封装和解封的过程，势必会对网络性能造成一些影响

下面来看一下容器网络的创建

# 容器网络的创建

## 集群网络初始化

在创建集群的时候就要指定pod的网络段，以及service的网络段，kubelet指定cni插件

```bash
kube-controller-manager --cluster-cidr=172.20.0.0/16 --service-cluster-ip-range=10.68.0.0/16

kubelet \
  --cni-bin-dir=/opt/kube/bin \
  --cni-conf-dir=/etc/cni/net.d \
  --network-plugin=cni
```

基于pod的网络模型，不通的网络模型就有不通实现，这里介绍cni插件flannel。

## kubelet创建容器

kubelet创建容器的时候会通过cni来创建，本文所用的环境里面cni插件为[flannel](https://github.com/containernetworking/plugins/tree/master/plugins/meta/flannel)，flannel插件会调用flanneld进程进行相关操作。

## flannel配置网络

作为cni网络插件，flannel要实现下面几个功能

- pod创建的时候为pod分配网络
    - 向容器添加网卡
    - 配置网卡IP
- 管理pod相互访问的流量
- 管理node到pod访问的流量

# 为pod分配网络地址

## 部署flannel

flannel作为DaemonSet以hostNetwork: true的方式部署在集群中，并且还需要一些可以访问pod和node的ClusterRole权限，主要进行相关配置，这样api也是flannel的后端存储(通过设定 flanneld --kube-subnet-mgr 来实现)，flanneld为flannel的服务进程name

[部署yaml](https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml)

## flannel工作过程

### 创建环境变量

通过/etc/kube-flannel/net-conf.json来一些环境变量，cni会将16位掩码的大网段按照字段SubnetLen来划分，该字段默认24，[详细配置](https://github.com/coreos/flannel/blob/master/Documentation/configuration.md)

```json
//cat /etc/kube-flannel/net-conf.json
{
  //这里定义总的pod网络范围，这个在创建集群的时候指定
  "Network": "172.20.0.0/16",
  "Backend": {
      //这里定义node上的pod子网大小
    "SubnetLen": "24",
    "Type": "vxlan"
  }
}
```
当然还需要向api server读取每个节点的PodCIDR 172.20.1.1/24

读取完以后新建一些变量

```bash
cat /run/flannel/subnet.env
FLANNEL_NETWORK=172.20.0.0/16
FLANNEL_SUBNET=172.20.1.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=true
```

### 为其他委托插件创建配置文件



接着flannel根据上面的环境变量以及传入flannel的配置生成下面的配置，其中/etc/kube-flannel/cni-conf.json在宿主机上会变为cat /etc/cni/net.d/10-flannel.conflist，这个是让flannel委托的插件来使用的

[配置信息](https://github.com/containernetworking/plugins/blob/master/plugins/meta/flannel/README.md)


```json
//cat /etc/kube-flannel/cni-conf.json 
//宿主机上为 cat /etc/cni/net.d/10-flannel.conflist
{
  "name": "cbr0",
  "plugins": [
    {
    //配置容器网络
      "type": "flannel",
      //flannel委托其他cni插件干活，下面是给他们传递的参数
      "delegate": {
        "hairpinMode": true,
        "isDefaultGateway": true
      }
    },
    {
    //进行端口映射
      "type": "portmap",
      "capabilities": {
        "portMappings": true
      }
    }
  ]
}
```

flannel会读取上面的环境变量去丰富下面的cni-conf.json配置文件，以便让他委托的插件可以工作，本质上是flannel调用其他的cni插件来完成创建网桥分配pod IP的工作，使用的插件

- [host-loacl](https://github.com/containernetworking/plugins/tree/master/plugins/ipam/host-local) 
- [bridge](https://github.com/containernetworking/plugins/tree/master/plugins/main/bridge)
- [loopback](https://github.com/containernetworking/plugins/tree/master/plugins/main/loopback)

```json
//
{
    "hairpinMode":true,
    "ipMasq":false,
    "ipam":{
        "routes":[
            {
                "dst":"172.20.0.0/16"
            }
        ],
        "subnet":"172.20.1.1/24",
        "type":"host-local"
    },
    "isDefaultGateway":true,
    "isGateway":true,
    "mtu":1410,
    "name":"cbr0",
    "type":"bridge"
}
```

### vxlan处理

flanneld启动后会在宿主机上做三件事[详见](https://github.com/coreos/flannel/blob/master/backend/vxlan/vxlan.go)

- 为集群内的所有的远端的node包含的子网创建路由

```bash
172.20.0.0/24 via 172.20.0.0 dev flannel.1 onlink
```

- 创建静态的arp，主要是vxlan网络内的arp

```bash
arp -n | grep flannel.1
172.20.0.0               ether   76:3a:70:b3:0b:99   CM                    flannel.1
```

- 创建所有远端的fdb，这个用来将封装好的vxlan包送到远程主机

```bash
bridge fdb | grep flannel.1
76:3a:70:b3:0b:99 dev flannel.1 dst 10.9.1.156 self permanent
```

### 集群信息同步

如果新加一台主机，他会上传自己的信息到api server 然后下载其他的主机的信息在本地上创建上面的东西。这些信息在node的注释里面

```yaml
apiVersion: v1
kind: Node
metadata:
  annotations:
    flannel.alpha.coreos.com/backend-data: '{"VtepMAC":"62:c2:e9:27:3c:39"}'
    flannel.alpha.coreos.com/backend-type: vxlan
    flannel.alpha.coreos.com/kube-subnet-manager: "true"
    flannel.alpha.coreos.com/public-ip: 10.9.1.161
```

# 参考资料

```bash
https://ieevee.com/tech/2017/08/12/k8s-flannel-src.html#22-fdb-%E8%BD%AC%E5%8F%91%E6%95%B0%E6%8D%AE%E5%BA%93
https://www.cnblogs.com/aguncn/p/10548009.html
https://www.cnblogs.com/xzkzzz/p/9936467.html
http://bamboox.online/k8s-13-cni.html
```






