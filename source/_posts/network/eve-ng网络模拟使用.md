---
title: eve-ng网络模拟使用
tags:
  - openflow
---

# 介绍

eve-ng是一个网络设备模拟平台

<!--more-->

# 安装

需要下载的文件

## eve-ng镜像

镜像下载iso直接安装到机器上或者虚拟机上面，也可以直接下载虚拟机镜像，使用VMware player来运行

- [社区版下载](http://www.eve-ng.net/downloads/eve-ng-2)
- [工具包下载](http://www.eve-ng.net/downloads/windows-client-side-pack) 工具包用来连接操作平台中的设备

## 导入虚拟机镜像

需要开启虚拟机虚拟化设置

![vem](https://qiniu.li-rui.top/vem.png)

网络可以选择桥接

![netbri](https://qiniu.li-rui.top/netbri.png)

启动虚拟机，用户root密码eve

直接访问呢pnet0上的地址就可以进行模拟了

# 设备镜像导入

## 下载并解压

[常用设备下载](https://pan.baidu.com/s/1cUGk4NZoDsRHV73qf8VnAg)  提取码: `86sf`

## 导入

导入到eve虚拟机目录`/opt/unetlab/addons`下面

```bash
root@eve-ng:/opt/unetlab/addons# ll qemu/
total 52
drwxr-xr-x 13 root root 4096 Jan  3 12:11 ./
drwxr-xr-x  5 root root 4096 Jun  9  2018 ../
drwxr-xr-x  2 root root 4096 Jan  3 12:10 asa-8.42/
drwxr-xr-x  2 root root 4096 Jan  3 12:09 asav-941-200/
drwxr-xr-x  2 root root 4096 Jan  3 12:10 asav-953-9/
drwxr-xr-x  2 root root 4096 Jan  3 12:11 asav-963-1/
drwxr-xr-x  2 root root 4096 Jan  3 12:10 fortinet-5.2/
drwxr-xr-x  2 root root 4096 Jan  3 12:10 fortinet-5.4/
drwxr-xr-x  2 root root 4096 Jan  3 12:10 pfsense-2.3.3/
drwxr-xr-x  2 root root 4096 Jan  3 12:09 vios-15.5.3M/
drwxr-xr-x  2 root root 4096 Jan  3 12:09 viosl2-15.2.4.55e/
drwxr-xr-x  2 root root 4096 Jan  3 12:09 vwaas-200-5.5.3/
drwxr-xr-x  2 root root 4096 Jan  3 12:09 vyos-117/

```

其中vios是思科的路由器交换机管理平台ios的虚拟机

每次导入都要进行执行一下权限

```bash
/opt/unetlab/wrappers/unl_wrapper -a fixpermissions
```

## node无法关机

新加的node都是关机状态，这样才可以新建拓扑，如遇到无法关机需要清空`/opt/unetlab/tmp/`，然后重启

```bash
rm -rf  /opt/unetlab/tmp/* 
reboot
```

