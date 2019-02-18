---
title: ceph-ansible官方使用
date: 2018-08-28 12:12:28
tags:
- ansible
- ceph

---

### 基本环境准备

### ansible安装

需要安装2.5版本的ansible，这里使用pip安装

```bash
yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm

yum install -y python-devel openssl-devel libffi-devel python-pip

pip install --upgrade pip

pip install ansible==2.5.3

```

<!--more-->

### palybook拉取

```bash
wget https://github.com/ceph/ceph-ansible/archive/v3.1.0.tar.gz
```

### playbook配置

#### Inventory

主机信息在文件ceph-ansible-stable-3.1/hosts.ini里面

根据需要取消注释组，加入主机

```ini
#可选的组有mons，agents，osds，mdss，rgws，nfss，restapis，rbdmirrors，clients，mgrs，iscsi-gws
```

配置示例

ansible主机上加入hosts解析即可写入主机名，不加入就写IP

```ini
[mons]
node1
node2
node3
[mgrs]
node2
[osds]
node1
node2
node3
[clients]
node1
node2
node3

```

### group_vars配置

#### ceph-ansible-stable-3.1/group_vars/all.yml
##### ceph源

这里我们选择阿里云的ceph源

```yml
ceph_origin: repository
ceph_repository: community
ceph_mirror: http://mirrors.aliyun.com/ceph
ceph_stable_key: http://mirrors.aliyun.com/ceph/keys/release.asc
ceph_stable_release: luminous
ceph_stable_repo: "{{ ceph_mirror }}/rpm-{{ ceph_stable_release }}"

```

##### fsid

我们自己生成fsid

```yml
fsid: 6ed9e8b2-1f5c-4358-be73-5c3f20278ae4
generate_fsid: false
```

##### 开启cephx

```yml
cephx: true
```
##### osd网络

public_network要和mon同网段

```yml
public_network: 192.168.66.0/24
cluster_network: 192.168.66.0/24
```

##### ceph.conf

在此加入自定义ceph.conf选项

```yml
ceph_conf_overrides:
  global:
    osd journal size: 1024
    osd pool default size: 3
    osd pool default min size: 1
    osd pool default pg num: 128
    osd pool default pgp num: 128
    osd crush chooseleaf type: 1
    bluestore block create: true
    bluestore block db size: 73014444032
    bluestore block db create: true
    bluestore block wal size: 107374182400
    bluestore block wal create: true

```

**在ceph-ansible-stable-3.1/group_vars/目录下对不同的组件可以单独配置**

### host_vars配置

该目录下定义了对单个主机变量的配置，如osd磁盘划分，mon地址等

示例：

devices为数据盘，dedicated_devices为db盘，bluestore_wal_devices为wal盘

**一定要保证三个盘的数量相等，即一个数据盘必须要配置一个db和一个wal盘**

```yml
---
osd_scenario: non-collocated
osd_objectstore: bluestore
devices:
  - /dev/sdc
  - /dev/sdd
  - /dev/sde
dedicated_devices:
  - /dev/sdf
  - /dev/sdf
  - /dev/sdf
bluestore_wal_devices:
  - /dev/sdg
  - /dev/sdg
  - /dev/sdg

monitor_address: 192.168.66.125
```

### ceph-ansible-stable-3.1/site.yml

#### 配置

这是playbook主执行文件，在此选选择是否安装制定组件，下面的组件的paly也要取消注释

示例：

```yml
- hosts:
  - mons
  # - agents
  - osds
  # - mdss
  # - rgws
  # - nfss
  # - restapis
  # - rbdmirrors
  - clients
  - mgrs
  # - iscsi-gws
```

### 安装

```bash
cd ceph-ansible-stable-3.1

ansible-playbook -i hosts.ini site.yml
```

### 清空集群

清空集群的配置在ceph-ansible-stable-3.1/infrastructure-playbooks/purge-cluster.yml

目录ceph-ansible-stable-3.1/infrastructure-playbooks内的文件需要复制到ceph-ansible-stable-3.1下才可以执行。

示例：

```bash
cp ceph-ansible-stable-3.1/infrastructure-playbooks/purge-cluster.yml
ceph-ansible-stable-3.1/purge-cluster.yml
ansible-playbook -i hosts.ini purge-cluster.yml
```




