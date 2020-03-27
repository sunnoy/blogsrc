---
title: kubernetes中service的ipvs实现
date: 2019-09-01 12:12:28
tags:
- kubernetes
---

# ipvs介绍

[ipvs](http://www.linuxvirtualserver.org/software/ipvs.html)是lvs在内核空间负载均衡实现，其包含一个在用户空间的命令行工具[ipvsadm](https://linux.die.net/man/8/ipvsadm)。除此还有[ktcpvs实现](http://www.linuxvirtualserver.org/software/ktcpvs/ktcpvs.html)

ipvs负载均衡的方式有三种

- [NAT](http://www.linuxvirtualserver.org/VS-NAT.html)
- [TUN](http://www.linuxvirtualserver.org/VS-IPTunneling.html)
- [DR](http://www.linuxvirtualserver.org/VS-DRouting.html)

<!--more-->

上面的负载均衡方式中只有nat可以进行端口映射，因此kubernetes中service中的ipvs实现是利用的nat模式。用来service IP 和service port和container ip和container port进行映射。

ipvs的nat是一种dnat，因此到了后端的realserver要想回包就要将realserver的默认网关设置为vip在的主机

## netfilter模块

ipvs和iptables一样都是基于[netfilter内核模块](https://www.netfilter.org/)

## ipvs和iptables

iptables和ipvs在过滤链的区别

- iptables

![iptables](https://qiniu.li-rui.top/iptables.png)

- ipvs

![ipvs](https://qiniu.li-rui.top/ipvs.png)

ipvs是工作在INPUT链上的，在INPUT链上直接处理数据包然后直接送至POSTROUTING链

# kube-proxy使用ipvs

集群中使用ipvs是通过kube-proxy来配置的，涉及的参数

```bash
--proxy-mode=ipvs
--ipvs-scheduler
--cleanup-ipvs
--ipvs-sync-period
--ipvs-min-sync-period
--ipvs-exclude-cidrs
# 下面的参数用来对访问service IP的数据包都进行伪装-snat
–masquerade-all
```

ivps模块的介绍[详见](https://kubernetes.io/blog/2018/07/09/ipvs-based-in-cluster-load-balancing-deep-dive/)，ipvs的实现是由华为的工程师维护的。

kube-prxoy会动态的根据service IP来动态增删ipvs的规则，当前有删除service ipvs规则么有删除的[bug](https://github.com/kubernetes/kubernetes/issues/78421)

## 和iptables结合

ipvs并不能进行snat操作，因此需要iptables的配合。ipvs模式中又加入了ipset，通过ipset就可以将一些需要snat的IP和端口写到一个集合里面，这样以来宿主机上的iptables的规则数量就是恒定的。

## kube-ipvs0虚拟网卡

iptables中service的实现service ip直接写在iptables规则中，但是在ipvs中要让内核知道vip是本机的，因此需要一个虚拟网卡动态绑定service ip作为vip，默认的虚拟网卡为kube-ipvs0。

为了保证任何一个node上的pod访问到集群内所有的service IP，因此就需要将集群内的service IP添加到每个node上的kube-ipvs0网卡上，因此需要将service IP的子网掩码设置为32位来限制广播域和节省service IP资源

```bash
5: kube-ipvs0: <BROADCAST,NOARP> mtu 1500 qdisc noop state DOWN group default
    link/ether aa:97:b4:6d:54:1c brd ff:ff:ff:ff:ff:ff
    inet 22.68.0.1/32 brd 22.68.0.1 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
```

# 访问流程

## 访问初始链

访问流程分为两种

- 集群内访问service IP， 直接到OUTPUT链
- 集群外访问nodeport，直接到PREROUTING链

上面两种都会进入KUBE-SERVICES链，然后进行一定的条件分流

```bash
iptables -S -t nat | grep KUBE-SERVICES

#通过PREROUTING到KUBE-SERVICES链 访问nodeport
-A PREROUTING -m comment --comment "kubernetes service portals" -j KUBE-SERVICES
#通过OUTPUT到KUBE-SERVICES链 访问cluser ip
-A OUTPUT -m comment --comment "kubernetes service portals" -j KUBE-SERVICES
```

## KUBE-CLUSTER-IP


KUBE-CLUSTER-IP为一个ipset集合

```bash
ipset list KUBE-CLUSTER-IP
Name: KUBE-CLUSTER-IP
Type: hash:ip,port
Revision: 2
Header: family inet hashsize 1024 maxelem 65536
Size in memory: 24496
References: 2
Members:
#以下面为例
22.68.141.68,tcp:8090
... ...
```

下面主要进行对访问目的地址的一些匹配

```bash
# src,dst 表示符合源地址22.68.141.68 目的端口为 8090 的数据包进入到KUBE-MARK-MASQ链
-A KUBE-SERVICES -m comment --comment "Kubernetes service cluster ip + port for masquerade purpose" -m set --match-set KUBE-CLUSTER-IP src,dst -j KUBE-MARK-MASQ

# des地址在本地的化也就是如果node ip加nodeport就会进入KUBE-NODE-PORT链
-A KUBE-SERVICES -m addrtype --dst-type LOCAL -j KUBE-NODE-PORT

#dst,dst 表示目的地址22.68.141.68 目的端口为 8090 的数据包就直接允许通过
#这种情况就是直接访问service IP的形式
#这里的ACCEPT之后，因为des地址就在本地，因此就会进入下一个INPUT链
#到此就会通INPUT链通过ipvs接管，进行dnat操作
-A KUBE-SERVICES -m set --match-set KUBE-CLUSTER-IP dst,dst -j ACCEPT
```

经过dnat的之后的数据包大概是

```bash
src: 173.20.1.33
des: 173.20.2.44
```
下面就会进行路由选择通过flannel.1进行相关的封包操作发送到远程主机

# snat情况

- kube-proxy starts with --masquerade-all=true
- Specify cluster CIDR in kube-proxy startup
- Load Balancer type service
- NodePort type service
- Service with externalIPs specified

[详细请参考](https://github.com/kubernetes/kubernetes/tree/master/pkg/proxy/ipvs)

# 参考资料

```bash
http://www.zsythink.net/archives/2134
https://blog.csdn.net/u011563903/article/details/87904873
```