---
title: API网关kong介绍
date: 2019-04-22 12:12:28
tags:
- mesh
---

# kong是什么

kong是一个微服务抽象层，也是API网关。相对于平常的反向代理，kong还有一些其他的作用

![kongingres](https://qiniu.li-rui.top/kongingres.png)

<!--more-->

kong依赖数据库，一般使用postgres，kong默认开放的端口有：

接受客户端流量的端口，proxy部分

- :8000 http端口
- :8443 https端口

admin API 端口 admin部分
- :8001 http端口
- :8444 https端口

通过端口可以知道，kong为配置信息的操作专门做API接口的封装

# kong安装

## compose安装

安装主要有下面几部分：

- 安装postgres数据库
- 使用kong容器进行数据库初始化
- kong部署kong
- kong ui konga数据库准备
- konga 部署

```yaml
version: "3"

networks:
 kong-net:
  driver: bridge

services:
  kong-database:
    image: postgres:9.6
    restart: always
    networks:
      - kong-net
    environment:
      POSTGRES_USER: kong
      POSTGRES_DB: kong
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "kong"]
      interval: 5s
      timeout: 5s
      retries: 5

  kong-migration:
    image: kong:latest
    command: [ "/bin/sh", "-c", "kong migrations bootstrap" ]
    networks:
      - kong-net
    restart: on-failure
    environment:
      KONG_PG_HOST: kong-database
    links:
      - kong-database
    depends_on:
      - kong-database

  kong:
    image: kong:latest
    restart: always
    networks:
      - kong-net
    environment:
      KONG_PG_HOST: kong-database
      KONG_PROXY_LISTEN: 0.0.0.0:8000
      KONG_PROXY_LISTEN_SSL: 0.0.0.0:8443
      KONG_ADMIN_LISTEN: 0.0.0.0:8001
    depends_on:
      - kong-migration
      - kong-database
    healthcheck:
      test: ["CMD", "curl", "-f", "http://kong:8001"]
      interval: 5s
      timeout: 2s
      retries: 15
    ports:
      - "8001:8001"
      - "8000:8000"


  konga-prepare:
    image: pantsel/konga:next
    command: "-c prepare -a postgres -u postgresql://kong@kong-database:5432/konga_db"
    networks:
      - kong-net
    restart: on-failure
    links:
      - kong-database
    depends_on:
      - kong-database


  konga:
    image: pantsel/konga:next
    restart: always
    networks:
        - kong-net
    environment:
      DB_ADAPTER: postgres
      DB_HOST: kong-database
      DB_USER: kong
      TOKEN_SECRET: km1GUr4RkcQD7DewhJPNXrCuZwcKmqjb
      DB_DATABASE: konga_db
      NODE_ENV: production
    depends_on:
      - kong-database
    ports:
      - "1337:1337"
```

## helm安装

在kubernetes中分为有数据库和无数据库安装，这里选用无数据库安装

```bash
helm install ms/kong --name kong --set ingressController.enabled=true \
  --set postgresql.enabled=false --set env.database=off
```

# kong 组件

- client 请求proxy端口的流量发源地
- upstream service kong后面的服务，也是client流量的归属地
- Service 对后端upstream的抽象封装
- Route 总是与service配合使用，一个service可以有个多个route，可以配置的字段有
    - hosts
    - paths
    - methods
- Plugin 插件作用在整个流量代理周期，从kong入口流量到kong出口流量

client请求的流量通过route指向与之相关的service，如果配置插件的话就会作用插件，service接到流量后给相应的upstream的服务上面。


# 服务创建

使用kong首先是需要创建服务，接着是创建路由

## 创建服务

向URL 8001/services/ 发送post请求 -i 在此为输出时显示协议头

- name 为service的名称
- url 后端upstream的url

添加成功后会返回一串json字符串，其中 id 为创建的service的唯一标识

```bash
curl -i -X POST http://localhost:8001/services/ \
    -d 'name=foo-service' \
    -d 'url=http://foo-service.com'

{
    "connect_timeout": 60000,
    "created_at": 1515537771,
    "host": "foo-service.com",
    "id": "d54da06c-d69f-4910-8896-915c63c270cd",
    "name": "foo-service",
    "path": "/",
    "port": 80,
    "protocol": "http",
    "read_timeout": 60000,
    "retries": 5,
    "updated_at": 1515537771,
    "write_timeout": 60000
}
```

## 创建路由

路由是指向service的，因此需要传入service的id

```bash
curl -i -X POST http://localhost:8001/routes/ \
    -d 'hosts[]=example.com' \
    -d 'paths[]=/foo' \
    -d 'service.id=d54da06c-d69f-4910-8896-915c63c270cd'
```



