---
title: kvm使用
date: 2018-11-04 12:12:28
tags:
- kvm
---
# kvm使用


## 创建虚拟机

### 创建磁盘

```bash
qemu-img create -f qcow2 /vm/centos71.img 80G

for dd in b c d e f g 
do
     #mkdir /opt/sd$dd -p
	# mkfs.ext4 /dev/sd$dd
	# mount /dev/sd$dd /opt/sd$dd
	 qemu-img create -f qcow2 /opt/sd$dd/$dd.img 100G
done

for dd in b c d e f g ;do qemu-img create -f qcow2 /opt/sd$dd/$dd.img 100G ;done
```
<!--more-->

### 创建虚拟机

```bash
virt-install --virt-type kvm --name centos71 --vcpus=2 --ram 2048 --disk path=/opt/sdb/b.img,size=100,format=qcow2 --network network=default --graphics vnc,port=5916,listen='0.0.0.0' --autostart --noautoconsole --os-type=linux --os-variant=rhel7 --cdrom=/root/CentOS-7-x86_64-DVD-1708.iso
```

通过它来找合适的xml文件格式
```bash
virt-install -n win2012 --vcpus=2 --ram=1024 --os-type=windows --os-variant=win2k12  --disk path=dfs1-data.img,format=qcow2,bus=virtio --pxe
```

## 网络

### 创建网桥

```bash
ip link add name br-wan type bridge
ip link set br-wan up
ip addr add 192.168.12.68/24 dev br-wan
ip link set dev eth2 master br-wan
ip route del default via 192.168.12.1 dev eth2
ip route add default via 192.168.12.1 dev br-wan

ping 173.18.22.1 -c 3 > /dev/null

if $? -ne 0; then

    #del br-wan
	  ip link delete br-wan type bridge
  	ip route add default via 192.168.12.1 dev eth2
	
	#reserve br-wan
	#ip link set eth2 nomaster
	#ip addr del 192.168.12.68/24 dev br-wan
	#ip route add default via 192.168.12.1 dev eth2
	
fi
```

### 查看虚拟机网卡

```bash
virsh domiflist 3
#可以获取网卡mac和接口
Interface  Type       Source     Model       MAC
-------------------------------------------------------
vnet83     bridge     br0KAkXqfjwbjfg virtio      52:54:00:a3:40:ec
vnet84     bridge     br1KAkXqfjwbjfg virtio      52:54:00:16:c3:a5

```

### 查看网桥

```bash
bridge link show 
```

### 添加网卡

```bash
virsh attach-interface --domain KhywifhRyvHv --type bridge --source br_mg --model virtio --persistent --config
```

### 去掉网卡

```bash
virsh detach-interface --domain tpl_server --type bridge --mac 52:54:00:cc:3c:5d --persistent
```
### 克隆后需要删除udev规则文件

```bash
rm -rf /etc/udev/rules.d/70-persistent-net.rules
```

## 虚拟机信息统计

### 查看虚拟机信息

```bash
virsh dominfo 68
```

### 开机启动

```bash
virsh autostart kvm-1
```

### 删除虚拟机

```bash
virsh undefine kvm-1
```

### 创建虚拟机

```bash
virsh define kvm-1
#创建并开机
virsh create kvm-1
```

### 编辑虚拟机

```bash
virsh edit DomainName
```

### 显示虚拟机当前配置

```bash
virsh dumpxml kvm1
```
### 增加内存

```bash
#增加到4g
virsh setmem dfs2 4G
```

### 增加cpu

```bash
#增加到4核
virsh setvcpus dfs2 4
```

### 开机启动

```bash
virsh autostart dfs2
```

### 开关机

```bash
#强制关机/重启/正常关机
virsh destroy /reboot/shutdown 
```

### 克隆

```bash
virt-clone -o DomainName -n new-DomainName –file /data/DomainName01.img
```

### cpu时间查看

```bash
virsh cpu-stats test02
```

### 磁盘统计

```bash
#查看磁盘错误
virsh domblkerror test
#查看磁盘列表
virsh domblklist test
#查看某一块磁盘信息
virsh domblkinfo test vda
Capacity:       42949672960
Allocation:     33431175168
Physical:       42949672960
#查看磁盘操作统计
virsh domblkstat test vda
```

### 内存统计

```go
virsh dommemstat test
```

### cpu 

```bash
#cpu时间
virsh cpu-stats 33
#cpu核数
virsh vcpucount 33
virsh vcpuinfo 33
#

```

##　api交互

### 本地api

```bash
#连接地址
qemu+unix:///system?socket=/var/run/libvirt/libvirt-sock
#连接方法
virsh -c qemu+unix:///system?socket=/var/run/libvirt/libvirt-sock
#非root用户需要加入组，比如alex用户
usermod -a -G libvirtd alex
```
### tcpapi

#### 开启tcp配置`/etc/libvirt/libvirtd.conf`

```bash
vim /etc/libvirt/libvirtd.conf

listen_tls = 0
listen_tcp = 1
tcp_port = "16509"
listen_addr = "0.0.0.0"
auth_tcp = "none"
```

#### 启动参数修改`vim /etc/sysconfig/libvirtd`

```bash
vim /etc/sysconfig/libvirtd
LIBVIRTD_ARGS="--listen"
```

#### 重启libvirtd服务

```bash
systemctl restart libvirtd
```

#### 连接测试

```bash
virsh -c qemu+tcp://127.0.0.1:16509/system list
```


#### 连接加密

tcp 认证改为

```bash
auth_tcp = "sasl"
```

再次连接报错，**留坑**

```bash
virsh -c qemu+tcp://127.0.0.1:16509/system
error: failed to connect to the hypervisor
error: authentication failed: authentication failed
```



## 使用xml定义开通

#### 准备xml文件

```xml
<domain type='kvm' id='5'>
  <name>centos72</name>
  <memory unit='KiB'>2097152</memory>
  <currentMemory unit='KiB'>2097152</currentMemory>
  <vcpu placement='static'>2</vcpu>
  <resource>
    <partition>/machine</partition>
  </resource>
  <os>
    <type arch='x86_64' machine='pc-i440fx-rhel7.0.0'>hvm</type>
    <boot dev='hd'/>
  </os>
  <features>
    <acpi/>
    <apic/>
  </features>
  <cpu mode='custom' match='exact' check='full'>
    <model fallback='forbid'>Haswell</model>
    <feature policy='disable' name='hle'/>
    <feature policy='disable' name='rtm'/>
    <feature policy='require' name='hypervisor'/>
    <feature policy='require' name='xsaveopt'/>
  </cpu>
  <clock offset='utc'>
    <timer name='rtc' tickpolicy='catchup'/>
    <timer name='pit' tickpolicy='delay'/>
    <timer name='hpet' present='no'/>
  </clock>
  <on_poweroff>destroy</on_poweroff>
  <on_reboot>restart</on_reboot>
  <on_crash>destroy</on_crash>
  <pm>
    <suspend-to-mem enabled='no'/>
    <suspend-to-disk enabled='no'/>
  </pm>
  <devices>
    <emulator>/usr/libexec/qemu-kvm</emulator>
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2'/>
      <source file='/opt/sdc/c.img'/>
      <backingStore/>
      <target dev='vda' bus='virtio'/>
      <alias name='virtio-disk0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x06' function='0x0'/>
    </disk>
    <disk type='file' device='cdrom'>
      <driver name='qemu' type='raw'/>
      <backingStore/>
      <target dev='hda' bus='ide'/>
      <readonly/>
      <alias name='ide0-0-0'/>
      <address type='drive' controller='0' bus='0' target='0' unit='0'/>
    </disk>
    <controller type='usb' index='0' model='ich9-ehci1'>
      <alias name='usb'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x04' function='0x7'/>
    </controller>
    <controller type='usb' index='0' model='ich9-uhci1'>
      <alias name='usb'/>
      <master startport='0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x04' function='0x0' multifunction='on'/>
    </controller>
    <controller type='usb' index='0' model='ich9-uhci2'>
      <alias name='usb'/>
      <master startport='2'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x04' function='0x1'/>
    </controller>
    <controller type='usb' index='0' model='ich9-uhci3'>
      <alias name='usb'/>
      <master startport='4'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x04' function='0x2'/>
    </controller>
    <controller type='pci' index='0' model='pci-root'>
      <alias name='pci.0'/>
    </controller>
    <controller type='ide' index='0'>
      <alias name='ide'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x01' function='0x1'/>
    </controller>
    <controller type='virtio-serial' index='0'>
      <alias name='virtio-serial0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x05' function='0x0'/>
    </controller>
    <interface type='network'>
      <source network='default' bridge='virbr0'/>
      <target dev='vnet0'/>
      <model type='virtio'/>
      <alias name='net0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
    </interface>
    <interface type='bridge'>
      <source bridge='br_mg'/>
      <target dev='vnet1'/>
      <model type='virtio'/>
      <alias name='net1'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x08' function='0x0'/>
    </interface>
    <serial type='pty'>
      <source path='/dev/pts/0'/>
      <target port='0'/>
      <alias name='serial0'/>
    </serial>
    <console type='pty' tty='/dev/pts/0'>
      <source path='/dev/pts/0'/>
      <target type='serial' port='0'/>
      <alias name='serial0'/>
    </console>
    <channel type='unix'>
      <source mode='bind' path='/var/lib/libvirt/qemu/channel/target/domain-5-centos71/org.qemu.guest_agent.0'/>
      <target type='virtio' name='org.qemu.guest_agent.0' state='connected'/>
      <alias name='channel0'/>
      <address type='virtio-serial' controller='0' bus='0' port='1'/>
    </channel>
    <input type='tablet' bus='usb'>
      <alias name='input0'/>
      <address type='usb' bus='0' port='1'/>
    </input>
    <input type='mouse' bus='ps2'>
      <alias name='input1'/>
    </input>
    <input type='keyboard' bus='ps2'>
      <alias name='input2'/>
    </input>
    <graphics type='vnc' autoport='yes' listen='0.0.0.0'>
      <listen type='address' address='0.0.0.0'/>
    </graphics>
    <video>
      <model type='cirrus' vram='16384' heads='1' primary='yes'/>
      <alias name='video0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x02' function='0x0'/>
    </video>
    <memballoon model='virtio'>
      <alias name='balloon0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x07' function='0x0'/>
    </memballoon>
  </devices>
  <seclabel type='dynamic' model='selinux' relabel='yes'>
    <label>system_u:system_r:svirt_t:s0:c8,c479</label>
    <imagelabel>system_u:object_r:svirt_image_t:s0:c8,c479</imagelabel>
  </seclabel>
  <seclabel type='dynamic' model='dac' relabel='yes'>
    <label>+0:+0</label>
    <imagelabel>+0:+0</imagelabel>
  </seclabel>
</domain>

```

#### 准备数据盘

```bash
qemu-img create -f qcow2 dfs1-data.img 350G
```


