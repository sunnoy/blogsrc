---
title: kvm使用ovn
tags:
- openflow
---

# 虚拟机接入方式

ovn的端口不支持Linux 网络协议栈，因此无法对ovs端口使用iptables，ovs 2.4以前不支持Linux kernel netfiler中的conntrack特性，也就是说ovs并不可以追踪连接状态

<!--more-->

## ovs接入方式

![ovs](https://qiniu.li-rui.top/ovs.png)

ovs接入加入Linux brigade是为了可以让流量经过Linux协议栈，可以直接使用iptables，也可以追踪连接状态

## ovn接入方式

![ovnvm](https://qiniu.li-rui.top/ovnvm.png)

ovn在ovs2.5引入，已经支持Linux kernel netfiler中的conntrack特性。

这样在ovs中存在两种表，一种是ovs的静态流表，一种是netfilter维护的动态session表，ovs首先会静态流表来识别流是否需要被track，如果需要那么会将其交给netfiler来处理，处理完成后返回相应的flag，ovs根据返回的flag做下一步处理，丢弃，转发或是其他action

# libvirt接入

## 添加ovs接口

编辑虚拟机

```bash
virsh edit vm-dom
```
网络接口添加如下

```xml
<interface type='bridge'>
  <mac address='52:54:00:93:05:25'/>
  <source bridge='br-int'/>
  <virtualport type='openvswitch'/>  
  <model type='virtio'/>
  <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
</interface>
```

## 获取一些信息

### br-int上端口

该命令会返回ovs网桥br-int上该虚拟机对应的端口

```bash
vm-port=virsh domiflist KZD8dg3aFw8b | grep  52:54:00:93:05:26 | awk '{print $1}'
```

### 端口的mac

```bash
vm-mac=ovs-vsctl get interface $vm-port external_ids:attached-mac | sed s/\"//g
```

### 端口的iface-id

iface-id是br-int上端口对应的逻辑交换机上的逻辑端口，他们是绑定关系

```bash
vm-lp=ovs-vsctl get interface $vm-port external_ids:iface-id | sed s/\"//g
```

## 逻辑交换机配置

### 端口添加

此处选逻辑交换机vpc2

```bash
ovn-nbctl lsp-add vpc2 $vm-lp
ovn-nbctl lsp-set-addresses $vm-lp "$vm-mac 172.77.1.109"
ovn-nbctl lsp-set-port-security $vm-lp "$vm-mac 172.77.1.109"
```

### dhcp选项

```bash
#应用dhcp选项，可选
ovn-nbctl lsp-set-dhcpv4-options $vm-lp 997e31dd-0d11-410c-9455-9c775085bed1
```

## 虚拟机配置

虚拟机重启启动网络服务，或者dhclient新添加的网络接口

```bash
dhclient eth0
```