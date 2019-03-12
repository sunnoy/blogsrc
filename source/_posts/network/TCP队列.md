---
title: TCP队列
date: 2019-03-12 12:12:28
tags:
- net
---

# tcp交互过程

![tcp-sync-queue-and-accept-queue-small-1024x747](https://qiniu.li-rui.top/tcp-sync-queue-and-accept-queue-small-1024x747.jpg)

<!--more-->

服务端通过bind()函数来绑定端口和IP，通过listen()函数来获取客户端的请求。

客户端通过connect()函数来向服务端发送请求，发送完SYN以后客户端就是SYN_SENT状态

服务端接受客户端的SYN后，其状态为SYN_RCVD，此时将该连接存放到syns_queue(半连接队列)的同时向客户端发送SYN+ACK

客户端收到服务端的SYN+ACK后向服务端发送ACK

服务端接收到ACK后，其状态为ESTABLISHED的同时，将半连接队列中的连接转移到accept_queue(全链接队列)

服务端accept()函数从accept_queue队列中取出连接，并为该连接分配文件描述符

客户端通过write()函数来向服务端发送数据，此时的连接状态仍然是ESTABLISHED

服务端通过read()函数来读取客户端发来的数据

# 队列溢出

## 查看

如何查看队列已经满了

```bash
netstat -s | egrep "listen|LISTEN" 
#全连接队列满了
667399 times the listen queue of a socket overflowed
#半连接队列满了，一般情况下半连接溢出会大于全连接溢出
667690 SYNs to LISTEN sockets ignored


ss -lnt
#Send-Q就是全连接队列的容量，
#Recv-Q为当前全连接队列已经使用的量
Recv-Q Send-Q Local Address:Port  Peer Address:Port 
0        50               *:3306             *:* 
```

## 相关参数

对于半连接队列内核参数为

```bash
max(64, /proc/sys/net/ipv4/tcp_max_syn_backlog)
```

对于全连接参数为,backlog为应用程序中的配置的数值

```bash
min(backlog, /proc/sys/net/core/somaxconn )
```

如果全连接队列满了，服务端如何处理客户端连接

```bash
/proc/sys/net/ipv4/tcp_abort_on_overflow
# 0 直接丢弃
# 1 发送rst
```





