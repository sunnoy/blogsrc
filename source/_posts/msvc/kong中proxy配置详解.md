---
title: kong中proxy配置详解
date: 2019-04-22 12:12:28
tags:
- kong
---

# proxy数据流向

流量从clien然后到route，route的终点是service。service会将流量发向相应的upstream。upstream后端还会有不同的target。

<!--more-->

# 路由创建

## 传入参数

创建路由需要的参数有

- hosts
- paths
- methods

这三个参数都是可选的，但是必须要有一个传入。路由在匹配的时候，也要全部匹配。如果一个参数中有个字段，至少要有一个可以匹配。

## 传入host

可以通过两种方式传入

### 有请求头

```bash
curl -i -X POST http://localhost:8001/routes/ \
    -H 'Content-Type: application/json' \
    -d '{"hosts":["example.com", "foo-service.com"]}'
```

### 直接传入数组

```bash
curl -i -X POST http://localhost:8001/routes/ \
   # 可以传入通配符
    -d 'hosts[]=*.example.com' \
    -d 'hosts[]=foo-service.com'
```

### 默认情况

client 向kong的请求中的 host 和kong向后端upstream的请求中的host 默认是不一样的。也就是说


```bash
#client -> kong
GET / HTTP/1.1
Host: service.com

#kong -> upstream
GET / HTTP/1.1
Host: <my-service-host.com>
```

当将路由进行 preserve_host=true 配置后

```bash
#client -> kong
GET / HTTP/1.1
Host: service.com

#kong -> upstream
GET / HTTP/1.1
Host: service.com
```

## 传入path

### 正则表达式

path的传入支持正则表达式，表达式规则遵循[Perl Compatible Regular Expression](http://pcre.org/)

多个正则表达配合的时候可以自定义优先级，通过使用参数 "regex_priority": 6 来确定

```json
[
    {
        "paths": ["/status/\d+"],
        "regex_priority": 0
    },
    {
        "paths": ["/version/\d+/status/\d+"],
        "regex_priority": 6
    },
    {
        "paths": ["/version"],
    },
    {
        "paths": ["/version/any/"],
    }
]
```

### strip_path

当创建一个路由前缀，但是后端的upstream却没有匹配的时候，需要用到

创建一个路由

```json
{
    "paths": ["/service"],
    "strip_path": true,
    "service": {
        "id": "..."
    }
}
```

请求是这样的

```bash
#client -> kong
GET /service/path/to/resource HTTP/1.1
Host: ...

#kong -> upstream 
#将/service进行剔除
GET /path/to/resource HTTP/1.1
Host: ...
```

## 传入http请求

这个较简单一些，就是匹配路由配置中的请求数组

```json
{
    "methods": ["GET", "HEAD"],
    "service": {
        "id": "..."
    }
}
```

## 路由匹配优先级

在定义路由的时候，定义的关键子越多，匹配的优先级越高

```json
{
    "hosts": ["example.com"],
    "service": {
        "id": "..."
    }
},

//优先匹配
{
    "hosts": ["example.com"],
    "methods": ["POST"],
    "service": {
        "id": "..."
    }
}
```

# 后端的负载均衡

kong支持两种负载均衡策略

- DNS-based loadbalancing
- Ring-balancer

本文主要介绍Ring-balancer

## Ring-balancer

Ring-balancer中和传统nginx的使用是类似的。一个upstream后面有好几个服务，这些服务在kong里面叫target。

### upstream概念

upstream相当与一个虚拟主机，可以被route中的host来使用。一个upstream可以附着多个target，每个target占用一个slots，在为target分配slots的时候，应该为100的倍数。比如有个6个target那么就是solts=600。upstream只是维护一个target的变更列表


### target概念

从上面知道，upstream维护的是一个target变更列表，因此target只可以增加不可以删除和修改。当需要增加新的target的时候配置上新weight即可。weight=0为完全禁用target。

### 负载均衡策略

kong支持多种负载均衡策略

- none: 基于权重轮询
- consumer: 基于consumer id
- ip: 基于远程IPhash
- header: 使用header hash
- cookie: 基于cookie name来轮询


## 蓝绿部署应用

upstream的名称就是service中host的名称，然后就是操作

![kong蓝绿部署](https://qiniu.li-rui.top/kong蓝绿部署.jpg)

### v1版本

```bash
# create an upstream
curl -X POST http://kong:8001/upstreams \
    --data "name=address.v1.service"

# add two targets to the upstream
curl -X POST http://kong:8001/upstreams/address.v1.service/targets \
    --data "target=192.168.34.15:80"
    --data "weight=100"
curl -X POST http://kong:8001/upstreams/address.v1.service/targets \
    --data "target=192.168.34.16:80"
    --data "weight=50"

# create a Service targeting the Blue upstream
curl -X POST http://kong:8001/services/ \
    --data "name=address-service" \
    --data "host=address.v1.service" \
    --data "path=/address"

# finally, add a Route as an entry-point into the Service
curl -X POST http://kong:8001/services/address-service/routes/ \
    --data "hosts[]=address.mydomain.com"
```

### v2版本

```bash
# create a new Green upstream for address service v2
curl -X POST http://kong:8001/upstreams \
    --data "name=address.v2.service"

# add targets to the upstream
curl -X POST http://kong:8001/upstreams/address.v2.service/targets \
    --data "target=192.168.34.17:80"
    --data "weight=100"
curl -X POST http://kong:8001/upstreams/address.v2.service/targets \
    --data "target=192.168.34.18:80"
    --data "weight=100"
```

### 蓝绿部署

```bash
# Switch the Service from Blue to Green upstream, v1 -> v2
curl -X PATCH http://kong:8001/services/address-service \
    --data "host=address.v2.service"
```

# 配置404路由

```bash
{
    "paths": ["/"],
    "service": {
        "id": "..."
    }
}
```

# 配置ssl路由

```bash
curl -i -X POST http://localhost:8001/certificates \
    -F "cert=@/path/to/cert.pem" \
    -F "key=@/path/to/cert.key" \
    -F "snis=ssl-example.com,other-ssl-example.com"
```










