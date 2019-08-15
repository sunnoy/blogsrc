---
title: 系统性能监测工具sysdig
date: 2019-08-14 08:12:28
tags:
- Linux
---

# sysdig概述

系统监控、分析和排障的工具，其实在 linux 平台上，已经有很多这方面的工具 strace、tcpdump、htop、iftop、lsof、netstat，它们都能用来分析 linux 系统的运行情况，而且还有很多日志、监控工具。为什么还需要 sysdig 呢？在我看来，sysdig 的优点可以归纳为三个词语：整合、强大、灵活。

本质上一个事件的解析器

```bash
https://github.com/draios/sysdig
```

<!--more-->

## 原理

sysdig 通过在内核的 driver 模块注册系统调用的 hook，这样当有系统调用发生和完成的时候，它会把系统调用信息拷贝到特定的 buffer，然后用户模块的组件对数据信息处理（解压、解析、过滤等），并最终通过 sysdig 命令行和用户进行交互。

![sysdig](https://qiniu.li-rui.top/sysdig.png)

# centos安装

```bash
rpm --import https://s3.amazonaws.com/download.draios.com/DRAIOS-GPG-KEY.public  
curl -s -o /etc/yum.repos.d/draios.repo https://s3.amazonaws.com/download.draios.com/stable/rpm/draios.repo
rpm -i https://mirror.us.leaseweb.net/epel/6/i386/epel-release-6-8.noarch.rpm
yum -y install kernel-devel*  dkms  sysdig
# 模块装载，会报错提示从s3下载失败
/usr/bin/sysdig-probe-loader

#需要下载依赖
https://s3.amazonaws.com/download.draios.com/stable/sysdig-probe-binaries/sysdig-probe-0.26.2-x86_64-3.10.0-693.2.2.el7.x86_64-9807bc1a2c8241700526cea7e11fbc8a.ko

#放到目录
/root/.sysdig

# 再次执行
/usr/bin/sysdig-probe-loader

# 安装完成
sysdig -h
```

# 使用

包含两个工具

- sysdig 
- csysdig 图形化的sysdig

## sysdig

既然是一个事件解析器，那么事件就相当于是数据库，因此需要相应的查询或者过滤的语法

### 事件输出格式

```bash
721481 15:10:41.141992017 2 sshd (19248) > selec
%evt.num %evt.outputtime %evt.cpu %proc.name (%thread.tid) %evt.dir %evt.type %evt.info
```

- evt.num： 递增的事件号
- evt.outputtime 事件发生的时间
- evt.cpu： 事件被捕获时所在的 CPU，也就是系统调用是在哪个 CPU 执行的。比较上面的例子中，值 2 代表机器的第3个 CPU
- proc.name： 生成事件的进程名字，也就是哪个进程在运行
- thread.tid： 线程的 id，如果是单线程的程序，这也是进程的 pid
- evt.dir： 事件的方向（direction），> 代表进入事件，< 代表退出事件
- evt.type： 事件的名称，比如 open、stat等，一般是系统调用
- evt.args： 事件的参数。如果是系统调用，这些对应着系统调用的参数

### 过滤语法

查看过滤器

```bash
sysdig -l
```

### 工具箱

常用的过滤使用lua脚本来整合就成为了Chisels

查看系统内置的Chisels

```bash
sysdig -cl

# 详解一个Chisels
sysdig -i topports_server

# 查看端口的读写速率
sysdig -c topports_server

# 安装网络情况进行排序
sysdig -c topprocs_net
```