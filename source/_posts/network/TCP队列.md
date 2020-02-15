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

# nginx的设置

```bash
listen 后面
```

# 建立连接的重试

```bash
net.ipv4.tcp_synack_retries #内核放弃连接之前发送SYN+ACK包的数量
net.ipv4.tcp_syn_retries #内核放弃建立连接之前发送SYN包的数量
``

# syncookies开启的情况
net.ipv4.tcp_syncookies=1

#SYNcookie就是将连接信息编码在ISN(initialsequencenumber)中返回给客户端，这时server不需要将半连接保存在队列中，而是利用客户端随后发来的ACK带回的ISN还原连接信息，以完成连接的建立，避免了半连接队列被攻击SYN包填满。

# 
```

# 半连接队列满了

对于SYN半连接队列的大小是由（/proc/sys/net/ipv4/tcp_max_syn_backlog）这个内核参数控制的，有些内核似乎也受listen的backlog参数影响，取得是两个值的最小值。当这个队列满了，不开启syncookies的时候，Server会丢弃新来的SYN包，而Client端在多次重发SYN包得不到响应而返回（connection time out）错误。但是，当Server端开启了syncookies=1，那么SYN半连接队列就没有逻辑上的最大值了，并且/proc/sys/net/ipv4/tcp_max_syn_backlog设置的值也会被忽略。

Client端在多次重发SYN包得不到响应而返回connection time out错误

# 全连接队列满了

当accept队列满了之后，即使client继续向server发送ACK的包，也会不被响应，此时ListenOverflows+1，同时server通过/proc/sys/net/ipv4/tcp_abort_on_overflow来决定如何返回，0表示直接丢弃该ACK，1表示发送RST通知client；相应的，client则会分别返回read timeout 或者 connection reset by peer。

client则会分别返回read timeout 或者 connection reset by peer

## 对半连接的影响

```c
    /*tcp_syncookies为2 进行syn cookie
      tcp_syncookies为1 且request队列满了 进行syn cookie处理
      tcp_syncookies为0 且request队列满了 将该syn报文drop掉*/
```

## 后续处理

```bash
cat /proc/sys/net/ipv4/tcp_abort_on_overflow

#0 表示server drop掉客户端的ack 
#1 表示server发送rst给客户端
```


# 参考文章

```bash
https://segmentfault.com/a/1190000008224853
http://jm.taobao.org/2017/05/25/525-1/
```






