---
title: iSCSI服务端和客户端配置
date: 2018-07-30 12:12:28
tags:
- Linux
- iSCSI

---

### 1. 服务端
- 安装
```bash
yum install epel-release -y
yum --enablerepo=epel -y install scsi-target-utils
```
- 配置
```bash
<target iqn.2011-04.vbird.centos:vbirddisk>
    #backing-store/direct-store区别为整块盘为direct，推荐backing
    backing-store /dev/sdb1
    initiator-address 192.168.159.0/24
    write-cache off
</target>



```
<!-- more -->

- 启动
```bash
systemctl restart tgtd.service
#查看监听端口
netstat -tlunp | grep tgt
[root@localhost conf.d]# netstat -tlunp | grep tgt
tcp        0      0 0.0.0.0:3260            0.0.0.0:*               LISTEN      37388/tgtd          
tcp6       0      0 :::3260                 :::*                    LISTEN      37388/t
#可能需要设置防火墙
iptables -A INPUT -p tcp -s 192.168.100.0/24 --dport 3260 -j ACCEPT
#查看服务状态
tgt-admin --show

Target 1: iqn.2011-04.vbird.centos:vbirddisk
    System information:
        Driver: iscsi
        State: ready
    I_T nexus information:
    LUN information:
        LUN: 0
            Type: controller
            SCSI ID: IET     00010000
            SCSI SN: beaf10
            Size: 0 MB, Block size: 1
            Online: Yes
            Removable media: No
            Prevent removal: No
            Readonly: No
            SWP: No
            Thin-provisioning: No
            Backing store type: null
            Backing store path: None
            Backing store flags: 
        LUN: 1
            Type: disk
            SCSI ID: IET     00010001
            SCSI SN: beaf11
            Size: 10736 MB, Block size: 512
            Online: Yes
            Removable media: No
            Prevent removal: No
            Readonly: No
            SWP: No
            Thin-provisioning: No
            Backing store type: rdwr
            Backing store path: /dev/sdb1
            Backing store flags: 
    Account information:
    ACL information:
        192.168.159.0/24
```
### 2. 客户端
- 安装
```bash
yum install iscsi-initiator-utils  -y
```
- 配置
```bash
#扫描服务端
iscsiadm -m discovery -t sendtargets -p 192.168.159.133
#重启客户端
service iscsi restart
#登录服务端节点
iscsiadm -m node -T iqn.2011-04.vbird.centos:vbirddisk -p 192.168.159.133:3260  -l
#登录后查看
iscsiadm -m session
#登出
iscsiadm -m node -T iqn.2011-04.vbird.centos:vbirddisk -p 192.168.159.133:3260 -u
#删除
iscsiadm -m node -T iqn.2011-04.vbird.centos:vbirddisk -p 192.168.159.133:3260 -o delete
```

