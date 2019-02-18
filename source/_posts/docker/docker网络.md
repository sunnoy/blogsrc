---
title: docker网络
date: 2018-11-04 12:12:28
tags:
- docker
---
# docker网络

## 架构

![dockernoet](https://qiniu.li-rui.top/dockernoet.png)

## 网络驱动
bridge驱动、host驱动、overlay驱动、remote驱动、null驱动

### bridge
 此驱动为Docker的默认设置驱动，使用这个驱动的时候，libnetwork将创建出来的Docker容器连接到Docker网桥上。

### host

 使用这种驱动的时候，libnetwork将不为Docker容器创建网络协议栈，即不会创建独立的network namespace。Docker容器中的进程处于宿主机的网络环境中，相当于Docker容器和宿主机共同用一个network namespace，使用宿主机的网卡、IP和端口等信息。

但是，容器其他方面，如文件系统、进程列表等还是和宿主机隔离的。

### overlay
 此驱动采用IETE标准的VXLAN方式，并且是VXLAN中被普遍认为最适合大规模的云计算虚拟化环境的SDN controller模式。在使用过程中，需要一个额外的配置存储服务，例如Consul、etcd和zookeeper。还需要在启动Docker daemon的时候额外添加参数来指定所使用的配置存储服务地址。

### remote
 这个驱动实际上并未做真正的网络服务实现，而是调用了用户自行实现的网络驱动插件，使libnetwork实现了驱动的可插件化，更好地满足了用户的多种需求。用户只需要根据libnetwork提供的协议标准，实现其所要求的各个接口并向Docker daemon进行注册。

### null
 使用这种驱动的时候，Docker容器拥有自己的network namespace，但是并不为Docker容器进行任何网络配置。也就是说，这个Docker容器除了network namespace自带的loopback网卡名，没有其他任何网卡、IP、路由等信息，需要用户为Docker容器添加网卡、配置IP等。
 这种模式如果不进行特定的配置是无法正常使用的，但是优点也非常明显，它给了用户最大的自由度来自定义容器的网络环境。
<!--more-->

## 理解
用一个例子来理解docker网络使用，网络架构图

![容器网络架构](https://qiniu.li-rui.top/容器网络架构.png)

### 网络创建

#### 创建两个自定义网络

```bash
docker network create backend
docker network create frontend
```
#### 查看网络

查看创建的网络，其中host，none，bridge为docker启动就创建的网络

```bash
docker network ls

NETWORK ID          NAME                DRIVER              SCOPE
be655f6e64de        backend             bridge              local
6c49cf7311ce        bridge              bridge              local
b9e18f47b3bb        frontend            bridge              local
6d9d1d115de1        host                host                local
523df4a8ecbf        none                null                local

```

宿主机上会创建`br-NETWORK ID`的网桥

```bash
#安装brctl工具
yum install bridge-utils -y
#查看网桥，存在br-b9e18f47b3bb和br-be655f6e64de分别是frontend和backend的NETWORK ID
[root@docker-pcp ~]# brctl show
bridge name     bridge id               STP enabled     interfaces
br-b9e18f47b3bb         8000.02425f2b3ff4       no
br-be655f6e64de         8000.02420fe2ad3c       no
docker0         8000.024239d76759       no

#docker会为每个网桥配置IP
[root@docker-pcp ~]# ip a | grep br-
5: br-be655f6e64de: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    inet 172.19.0.1/16 brd 172.19.255.255 scope global br-be655f6e64de
6: br-b9e18f47b3bb: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    inet 172.20.0.1/16 brd 172.20.255.255 scope global br-b9e18f47b3bb

```

详细查看某一个网络

```bash
[root@docker-pcp ~]# docker network inspect backend
[
    {
        "Name": "backend",
        "Id": "be655f6e64def08825ce1d4b4f38f5b0b737f8de9cb26bcffeb4f05d9fc5cdb2",
        "Created": "2018-10-17T11:19:52.6955544+08:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.19.0.0/16",
                    "Gateway": "172.19.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {},
        "Options": {},
        "Labels": {}
    }
]

```

### 容器创建

#### 创建容器

按照架构图进行创建容器，并且与docker网络进行绑定

```bash
docker run -itd --name c1 --net backend busybox
docker run -itd --name c2 --net backend busybox
docker run -itd --name c3 --net frontend busybox
```

#### 容器内的网络

容器内会存在一个lo网卡和eth0网卡，eth0网卡和backend进行绑定，IP和网桥br-be655f6e64de同网路段，容器的默认路由会指向网桥br-be655f6e64de的IP。c1和c2连接相同的网络具有相同的IP段和默认路由

```bash
#进入c1
docker exec -it c1 sh
#查看容器的网卡
# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
7: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue
    link/ether 02:42:ac:13:00:02 brd ff:ff:ff:ff:ff:ff
    inet 172.19.0.2/16 brd 172.19.255.255 scope global eth0
       valid_lft forever preferred_lft forever
#查看路由
# ip route
default via 172.19.0.1 dev eth0
172.19.0.0/16 dev eth0 scope link  src 172.19.0.2

#进入c2
# docker exec -it c2 sh
#查看IP
/ # ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
9: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue
    link/ether 02:42:ac:13:00:03 brd ff:ff:ff:ff:ff:ff
    inet 172.19.0.3/16 brd 172.19.255.255 scope global eth0
       valid_lft forever preferred_lft forever

#查看路由
/ # ip route
default via 172.19.0.1 dev eth0
172.19.0.0/16 dev eth0 scope link  src 172.19.0.3

```

### 容器联通

c1和c2在同一个网络里面是可以通的，c2和c3在不同网络是不同的，docker会自动将容器name作为容器的hostname

**同一个网络可以使用容器name进行通信**

```bash
#c1到c2
/ # ping c2
PING c2 (172.19.0.3): 56 data bytes
64 bytes from 172.19.0.3: seq=0 ttl=64 time=0.113 ms

#c1到c3
/ # ping c3
ping: bad address 'c3'

```

### 自定义网络和默认网络区别

- 自定义网络内的容器之间所有端口都是互通的
- 自定义网络内的容器具有自动的dns解析，就是可以直接访问容器的name来访问容器
- 运行中的容器可以直接绑定和解绑自定义网络
- 自定义网络会创建宿主机网桥

### c2连接网络

加入frontend

```bash
#c2加入网络frontend
docker network connect frontend c2
#退出
docker network disconnect frontend c2
```

查看c2网络与c3联通
c2中加入了一块新的网卡，网卡和frontend的网桥绑定
```bash
#查看IP
/ # ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
9: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue
    link/ether 02:42:ac:13:00:03 brd ff:ff:ff:ff:ff:ff
    inet 172.19.0.3/16 brd 172.19.255.255 scope global eth0
       valid_lft forever preferred_lft forever
13: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue
    link/ether 02:42:ac:14:00:03 brd ff:ff:ff:ff:ff:ff
    inet 172.20.0.3/16 brd 172.20.255.255 scope global eth1
       valid_lft forever preferred_lft forever
#查看路由
/ # ip route
default via 172.19.0.1 dev eth0
172.19.0.0/16 dev eth0 scope link  src 172.19.0.3
172.20.0.0/16 dev eth1 scope link  src 172.20.0.3

#与c3和c1联通
/ # ping c3
PING c3 (172.20.0.2): 56 data bytes
64 bytes from 172.20.0.2: seq=0 ttl=64 time=0.136 ms
64 bytes from 172.20.0.2: seq=1 ttl=64 time=0.099 ms
/ # ping c1
PING c1 (172.19.0.2): 56 data bytes
64 bytes from 172.19.0.2: seq=0 ttl=64 time=0.123 ms
64 bytes from 172.19.0.2: seq=1 ttl=64 time=0.064 ms

```

### 宿主机网桥

查看此时宿主机的网桥，此时每个br-的网桥都有两个接口

```bash
[root@docker-pcp ~]# brctl show
bridge name     bridge id               STP enabled     interfaces
br-b9e18f47b3bb         8000.02425f2b3ff4       no              veth1f8051a
                                                        veth47abc77
br-be655f6e64de         8000.02420fe2ad3c       no              veth7c91340
                                                        vethc752986

```

通过docker network查看，可见docker网络上绑定的容器和IP信息

```bash
[root@docker-pcp ~]# docker network inspect backend
[
    {
        "Name": "backend",
        "Id": "be655f6e64def08825ce1d4b4f38f5b0b737f8de9cb26bcffeb4f05d9fc5cdb2",
        "Created": "2018-10-17T11:19:52.6955544+08:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.19.0.0/16",
                    "Gateway": "172.19.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "a8f2f6e3e7c2edfba52e82f5492b9958327356446e20658d9624693838b833ef": {
                "Name": "c2",
                "EndpointID": "38eab6613bb71c93aa3d9acc3b6c60d636410b3fbd60203b73c3b19a5c16b197",
                "MacAddress": "02:42:ac:13:00:03",
                "IPv4Address": "172.19.0.3/16",
                "IPv6Address": ""
            },
            "b8adc9330ca6be48131d73e0603ef984216de672d53a29b6c2e57f532b8142bb": {
                "Name": "c1",
                "EndpointID": "74acec1f5acc8686f057dc7595ea5ada9155303a25fa2e0f95332304c8333fce",
                "MacAddress": "02:42:ac:13:00:02",
                "IPv4Address": "172.19.0.2/16",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {}
    }
]

```

## iptables

### 容器对外联通

```bash
[root@docker-pcp ~]# docker exec -it c2 sh
/ # ping 114.114.114.114
PING 114.114.114.114 (114.114.114.114): 56 data bytes
64 bytes from 114.114.114.114: seq=0 ttl=70 time=19.160 ms

```

### 查看iptables规则

#### 容器的包出去
```bash
-A POSTROUTING -s 172.20.0.0/16 ! -o br-b9e18f47b3bb -j MASQUERADE
```
数据包从容器出来后进入到br-网桥，br-网桥根据目的地址去找宿主机的默认网关，从和默认网关相同网段接口出去的时候进行了SANT转换，然后进入下一跳路由。

#### 容器之间通信

```bash
-A FORWARD -i br-b9e18f47b3bb -o br-b9e18f47b3bb -j ACCEPT
```

#### 端口绑定

启动tocmat，将主机端口:容器端口做映射
```bash
#启动一个tomcat
docker run -itd --rm --name  tomcat01 -p 8080:8080 tomcat:latest

```

查看宿主iptables规则

```bash
#目的和源都是tomcat容器且端口是8080就进行SNAT转换，出包
-A POSTROUTING -s 172.17.0.2/32 -d 172.17.0.2/32 -p tcp -m tcp --dport 8080 -j MASQUERADE
#外边访问主机8080的包都送到172.17.0.2:8080，来包
-A DOCKER ! -i docker0 -p tcp -m tcp --dport 8080 -j DNAT --to-destination 172.17.0.2:8080
#允许网桥docker0上8080通过
-A DOCKER -d 172.17.0.2/32 ! -i docker0 -o docker0 -p tcp -m tcp --dport 8080 -j ACCEPT
```

## 桥接方式和宿主机同网段

一定要先创建宿主机网桥，然后绑定宿主机网卡，然后使用docker绑定该网桥，因为docker创建的网桥再和网卡绑定不通

### 宿主机建桥

```bash
ip link add name pcp type bridge
ip link set pcp up
ip link set dev eth3 master pcp
ip addr add 172.16.1.119/16 dev pcp
#删除网桥
ip link delete pcp type bridge
```

### docker创建桥接网络

```bash
docker network create --subnet=172.16.0.0/16 --gateway=172.16.1.119 -o com.docker.network.bridge.name=pcp test
```

### 启动容器测试

```bash
docker run -itd --rm --name aa --network test  --ip 172.16.1.120 busybox
docker run -itd --rm --name bb --network test  --ip 172.16.1.121 busybox
```

### 一次性脚本

```bash
host_net=eth3
bri_name=pcp
bri_ip=172.16.1.119
docker_net=app_net

ip link add name $bri_name type bridge
ip link set $bri_name up
ip link set dev $host_net master $bri_name
ip addr add $bri_ip/16 dev $bri_name
 
docker network create --subnet=172.16.0.0/16 --gateway=$bri_ip -o com.docker.network.bridge.name=$bri_name $docker_net
```





