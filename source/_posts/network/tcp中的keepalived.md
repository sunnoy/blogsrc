---
title: tcp中的keepalived
date: 2020-01-120 12:12:28
tags:
- net
---

# 问题

客户端异常退出，导致服务端上连接没有释放

<!--more-->

# 系统配置

```bash
net.ipv4.tcp_keepalive_time = 200 最后一次数据交换到TCP发送第一个保活探测包的间隔，即允许的持续空闲时长
net.ipv4.tcp_keepalive_probes = 9 开始发送后的发送次数
net.ipv4.tcp_keepalive_intvl = 75 发送频率
```

# 查看是否开启

```bash
netstat -altpno | grep "10.173.27.254:22"
ss -aoen|grep 10.24.160.254:55098|grep ESTAB
```