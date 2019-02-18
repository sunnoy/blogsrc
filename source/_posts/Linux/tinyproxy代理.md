---
title: tinyproxy代理
date: 2018-11-04 12:12:28
tags:
- linux
---
## tinyproxy代理

有公网的主机提供代理，供没有公网的主机上网，或者赋能

### 1. tinyproxy安装

该工具提供http/s类型的代理

```bash
yum install tinyproxy
```
<!--more-->
### 2. tinyproxy配置

```bash
vi /etc/tinyproxy/tinyproxy.conf

#简单使用只需要把下面行注释掉，允许任何人连接
#Allow 127.0.0.1

#可以改变端口号
Port 8888

#重启服务
systemctl restart tinyproxy

#添加开机启动
systemctl enable tinyproxy
```

### 3. 使用代理

```bash
#在环境变量中配置代理/etc/profile最后添加
 export http_proxy=172.18.252.82:8888

#激活环境变量
source /etc/profile
```
