---
title: kvm中的网络
date: 2018-07-19 12:14:28
tags:
- kvm
- net

---

# kvm中的网络连接有三种方式

- host-only 所有的虚拟机组成一个封闭的局域网
- nat 虚拟机通过nat方式通过宿主机访问网络
- bridge 虚拟机和宿主机在同一个局域网络内

<!--more-->

### 1. host-only

虚拟机们抱团，不和宿主机通讯

![20160814130755667](https://qiniu.li-rui.top/20160814130755667.png)

### 2. 网桥模式

虚拟机通过宿主机上的网桥和宿主机在同一个局域网络中

![brig](https://qiniu.li-rui.top/brig.png)

- 宿主机操作

下面样例中网桥br0和宿主机上的网卡eth0绑定

```bash
#创建网桥文件ifcfg-br0
vi /etc/sysconfig/network-scripts/ifcfg-br1

DEVICE=br1
TYPE=Bridge
ONBOOT=yes

#修改网络参数后不会立即生效，yes为立即生效，不用重启network服务
NM_CONTROLLED=no

BOOTPROTO=static
IPADDR=192.168.8.112
NETMASK=255.255.255.0
GATEWAY=192.168.8.2
DNS1=192.168.8.2
DNS2=8.8.8.8

```

命令行操作

```bash
#创建网桥
ip link add name br1 type bridge
#启动网桥
ip link set br1 up
#给网桥添加端口
ip link set dev veth0 master br1
#给网桥添加IP
ip addr add 192.168.3.101 dev br1

```

**此时eth0被br0绑定过后相当于网线，不必配置IP，将IP转移到br0上**

```bash
vim /etc/sysconfig/network-scripts/ifcfg-eth0

DEVICE=eth0
TYPE=Ethernet
ONBOOT=yes
NM_CONTROLLED=no
BOOTPROTO=none
#将 eth0 绑定到网桥接口 br0 上
BRIDGE=br0
```

- 在客户机

客户机安装时绑定网桥

```bash
virt-install \
--name  CentOS6.8 \
--ram 512 \
--disk path=/kvm_data/CentOS6.8.img,size=30 \
--vcpus 1 \
--os-type linux \
--os-variant rhel6 \
--network bridge=br0 \    // 桥接网络
--graphics none \
--console pty,target_type=serial \
--location 'http://mirrors.aliyun.com/centos/6/os/x86_64/' \      // 这里使用网络镜像
--extra-args 'console=ttyS0,115200n8 serial'
```

客户机已经安装好后配置xml

```xml
<-- virsh edit coohx --/>

  <interface type='bridge'>
      <mac address='52:54:00:ad:4c:86'/>
      <source bridge='br0'/>
      <model type='virtio'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
    </interface>


```

客户机网卡配置

需要和宿主机配置一样，除了IP不一样

```bash
#vi /etc/sysconfig/network-scripts/ifcfg-eth0

DEVICE="eth0"
BOOTPROTO="static"
BROADCAST="192.168.8.255"
DNS1="192.168.8.2"
GATEWAY="192.168.8.2"                  // 网关为宿主机所在局域网的网关
HWADDR="52:54:00:AD:4C:86"
IPADDR="192.168.8.16"                 // 与宿主机处于同一局域网
IPV6INIT="yes"
IPV6_AUTOCONF="yes"
NETMASK="255.255.255.0"
NM_CONTROLLED="yes"
ONBOOT="yes"
TYPE="Ethernet"
```

### 3. NAT配置

- 理解

nat方式是kvm安装后的默认网络方式，外界无法访问虚拟机

默认方式会创建两个网桥virbr0 ，其中virbr0是个nat服务器，相当于虚拟机的负载均衡器

virbr0创建了和虚拟机创建了一组局域网，网段默认是192.168.122.0，virbr0的IP为192.168.122.1

![20160814125506928](https://qiniu.li-rui.top/20160814125506928.png)

![20160814130755667](https://qiniu.li-rui.top/20160814130755667.png)

```bash
#查看路由
route -n
192.168.122.0   0.0.0.0         255.255.255.0   U     0      0        0 virbr0

#查看nat表，将会转发virbr0上的数据包，将他们的ip转化为外网IP，并不是使用SNAT/DNA而是MASQUERADE
Chain POSTROUTING (policy ACCEPT 349 packets, 22016 bytes)
 pkts bytes target     prot opt in     out     source               destination
    0     0 RETURN     all  --  *      *       192.168.122.0/24     224.0.0.0/24
    0     0 RETURN     all  --  *      *       192.168.122.0/24     255.255.255.255
    0     0 MASQUERADE  tcp  --  *      *       192.168.122.0/24    !192.168.122.0/24     masq ports: 1024-65535
    0     0 MASQUERADE  udp  --  *      *       192.168.122.0/24    !192.168.122.0/24     masq ports: 1024-65535
    0     0 MASQUERADE  all  --  *      *       192.168.122.0/24    !192.168.122.0/24

#使用dnsmasq来完成客户机的DNS和DHCP
cat /var/lib/libvirt/dnsmasq/default.conf

strict-order
pid-file=/var/run/libvirt/network/default.pid
except-interface=lo
bind-dynamic
interface=virbr0
dhcp-range=192.168.122.2,192.168.122.254
dhcp-no-override
dhcp-lease-max=253
dhcp-hostsfile=/var/lib/libvirt/dnsmasq/default.hostsfile
addn-hosts=/var/lib/libvirt/dnsmasq/default.addnhosts

```



- 配置

在宿主机上面

```bash
#在文件/usr/share/libvirt/networks/default.xml
#可配置网络段
<network>
  <name>default</name>     // 网络名
  <bridge name="virbr0" />
  <forward/>
  <ip address="192.168.122.1" netmask="255.255.255.0">
    <dhcp>
      <range start="192.168.122.2" end="192.168.122.254" />
    </dhcp>
  </ip>
</network>

#标记自动启动
virsh net-autostart default

#启动网络
virsh net-start default

#查看网络
virsh net-list


```

虚拟机配置

```bash
#编辑xml文件

  <interface type='network'>
      <mac address='52:54:00:d5:0d:34'/>  // 这张网卡对应宿主机的vnet1
      <source network='default'/>             //加载NAT网络配置
      <model type='virtio'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x07' function='0x0'/>
    </interface>
```

启动虚拟机以后会自动绑定一个接口vnet1到virbr0网桥上面

```bash
#绑定一个网卡
vi /etc/sysconfig/network-scripts/ifcfg-eth1

DEVICE="eth1"
BOOTPROTO="static"
BROADCAST="192.168.122.255"
DNS1="192.168.122.1"
GATEWAY="192.168.122.1"           // 网关为 virbr0
HWADDR="52:54:00:d5:0d:34"
IPADDR="192.168.122.112"
IPV6INIT="yes"
IPV6_AUTOCONF="yes"
NETMASK="255.255.255.0"
NM_CONTROLLED="no"
ONBOOT="yes"
TYPE="Ethernet"
```


### 4. 网卡

cat /etc/udev/rules.d/70-persistent-ipoib.rules文件
