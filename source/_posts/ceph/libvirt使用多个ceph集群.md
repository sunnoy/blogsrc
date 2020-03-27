---
title: libvirt使用多个ceph集群
date: 2018-09-20 12:12:28
tags:
- kvm
- ceph
- libvirt
---
## 

### kvm架构

![kvm架构](https://qiniu.li-rui.top/kvm架构.png)

### qemu

其实 QEMU 原本不是 KVM 的一部分，它自己就是一个纯软件实现的虚拟化系统，所以其性能低下。但是，QEMU 代码中包含整套的虚拟机实现，包括处理器虚拟化，内存虚拟化，以及 KVM需要使用到的虚拟设备模拟（网卡、显卡、存储控制器和硬盘等）。 为了简化代码，KVM 在 QEMU 的基础上做了修改。VM 运行期间，QEMU 会通过 KVM 模块提供的系统调用进入内核，由 KVM 负责将虚拟机置于处理的特殊模式运行。当虚机进行 I/O 操作时，KVM 会从上次系统调用出口处返回 QEMU，由 QEMU 来负责解析和模拟这些设备。从 QEMU 角度看，也可以说是 QEMU 使用了 KVM 模块的虚拟化功能，为自己的虚机提供了硬件虚拟化加速。除此以外，虚机的配置和创建、虚机运行所依赖的虚拟设备、虚机运行时的用户环境和交互，以及一些虚机的特定技术比如动态迁移，都是 QEMU 自己实现的.

 
 <!--more-->
### kvm管理工具

- libvirt：操作和管理KVM虚机的虚拟化 API，使用 C 语言编写，可以由 Python,Ruby, Perl, PHP, Java 等语言调用。可以操作包括 KVM，vmware，XEN，Hyper-v, LXC 等在内的多种 Hypervisor。
- Virsh：基于 libvirt 的 命令行工具 （CLI）
- Virt-Manager：基于 libvirt 的 GUI 工具
- virt-v2v：虚机格式迁移工具
- virt-* 工具：包括 Virt-install （创建KVM虚机的命令行工具）， Virt-viewer （连接到虚机屏幕的工具），Virt-clone（虚机克隆工具），virt-top 等
- sVirt：安全工具

### 安装

```bash
yum install qemu-kvm qemu-img -y
yum install virt-manager libvirt libvirt-python python-virtinst libvirt-client -y
```

## ceph边节点

![ceph-kvm](https://qiniu.li-rui.top/ceph-kvm.png)

#### 创建虚拟机所用的pool

在mon节点上或者clent节点上执行

```bash
#创建池
ceph osd pool create libvirt-pool 128 128
#定义池类型
ceph osd pool application enable libvirt-pool rbd
rbd pool init libvirt-pool
```

#### 在pool创建一个rbd

```bash
#13240mb
rbd create librbd --size 30720 --pool libvirt-pool --image-format 2
```

#### 创建ceph中的用户

```bash
#针对pool libvirt-pool，会返回key
ceph auth get-or-create client.libvirt mon 'allow *' osd 'allow *' mgr 'allow *'

#认证查看，确保对mon，osd，mgr都有权限
ceph auth list 
#没有就补充上
ceph auth caps client.libvirt mon 'allow *' osd 'allow *' mgr 'allow *'

```

## kvm边

### 依赖安装

```bash
#ceph
yum insatll ceph-common
#kvm 包 
```

### 创建没有存储的虚拟机

```bash
virt-install --virt-type kvm --name centos80 --vcpus=2 --ram 2048 --network network=default --graphics vnc,port=5916,listen='0.0.0.0' --noautoconsole --os-type=linux --os-variant=rhel7 --disk none --cdrom=/root/7.iso
```

### 接入ceph

#### 添加virsh密钥

secret-define为libvirt接入volume，ceph，iscsi的接口

#### 创建两个xml文件

用来创建两个libvirt用户对接两个集群

name 对应ceph的用户

```xml
<secret ephemeral='no' private='no'>
  <uuid>77085c23-2177-242d-7661-c785df7f6230</uuid>
  <usage type='ceph'>
    <name>client.admin secret</name>
  </usage>
</secret>
```

```xml
<secret ephemeral='no' private='no'>
  <uuid>88085c23-2177-242d-7661-c785df7f6230</uuid>
  <usage type='ceph'>
    <name>client.libvirt secret</name>
  </usage>
</secret>

```

##### 一种生成方法


```bash
cat > secret.xml <<EOF
<secret ephemeral='no' private='no'>
      <usage type='ceph'>
            <name>client.libvirt secret</name>
      </usage>
</secret>
EOF

#定义出来,会返回Secret 2c8b6fa3-1dc7-4107-a0e0-8c7688a7bf5f created
virsh secret-define --file secret.xml
#列出
virsh secret-list
#用ceph用户来签发libvirt用户的secret
#签发命令
virsh secret-set-value --secret <libvirt_uuid> --base64 <ceph_user_key>
#获取ceph用户key，然后重定向到文件client.admin.key
ceph auth get-key client.admin | tee client.admin.key

virsh secret-set-value --secret <libvirt_uuid> --base64 <ceph_user_key>
```

##### 自动生成方法

```bash
#在此固定住uuuid
cat << EOF > /etc/ceph/secret.xml
<secret ephemeral='no' private='no'>
    <uuid>6a085c23-2177-242d-7661-c785df7f6230</uuid>
    <usage type='ceph'>
        <name>client.admin secret</name>
    </usage>
</secret>
EOF

#先定义出libvirt的secret，然后获取virsh secret-define标准输出的第二列中的uuid给变量secretuuid
secretuuid=`virsh secret-define --file /etc/ceph/secret.xml | awk '{print $2}'`
#获取用来签发libvirt的secret的ceph用户key，传入到文件client.admin.key中
ceph auth get-key client.admin | tee client.admin.key
#进行签发流程
virsh secret-set-value --secret $secretuuid --base64 $(cat client.admin.key)
```

#### secret相关命令

```bash
#导出xml
virsh secret-dumpxml uuid
#导出ceph用户key
virsh secret-get-value uuid
#删除secret
virsh secret-undefine uuid

```
#### kvm虚拟机添加磁盘

```bash
#编辑
virsh edit centos80
```

##### 加入虚拟机的devices块

停掉虚拟机然后加入xml配置

```xml
<disk type='network' device='disk'>
      <driver name='qemu' type='raw'/>
          <auth username='libvirt'>
              <secret type='ceph' uuid='727c2e12-a6ac-4f57-a553-8b6fd13a1da9'/>
          </auth>
      <source protocol='rbd' name='libvirt-pool/librbd'>
          <host name='MON1' port='6789'/>
      </source>
      <target dev='vda' bus='virtio'/>
</disk>

```

##### 示例

```xml
<disk type='network' device='disk'>
    <driver name='qemu' type='raw' cache='none'/>
    <auth username='admin'>
    <secret type='ceph' uuid='77085c23-2177-242d-7661-c785df7f6230'/>
    </auth>
    <source protocol='rbd' name='rbd/librbda'>
    <host name='22.216.19.23' port='6789'/>
    </source>
    <target dev='vdb' bus='virtio'/>
    <address type='pci' domain='0x0000' bus='0x00' slot='0x12' function='0x0'/>
</disk>

<disk type='network' device='disk'>
    <driver name='qemu' type='raw' cache='none'/>
    <auth username='libvirt'>
    <secret type='ceph' uuid='88085c23-2177-242d-7661-c785df7f6230'/>
    </auth>
    <source protocol='rbd' name='libvirt-pool/librbd'>
    <host name='22.216.19.26' port='6789'/>
    </source>
    <target dev='vdc' bus='virtio'/>
    <address type='pci' domain='0x0000' bus='0x00' slot='0x13' function='0x0'/>
</disk>


```

### 参考资料

```bash
https://access.redhat.com/documentation/en-us/red_hat_ceph_storage/1.3/html/block_device_guide/qemu

https://libvirt.org/formatsecret.html#CephUsageType
```