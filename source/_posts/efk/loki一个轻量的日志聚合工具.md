---
title: kubernetes中fluentd部署
date: 2019-11-26 12:12:28
tags:
- efk
---

# es太重了

毫无疑问es是当前日志集合的标准，但是对于一些场景es还是有些过于重了。

- 运行资源消耗大
- 不可以实时查看日志
- 查询与聚合语法复杂

<!--more-->

# loki出现了

Like Prometheus, but for logs!

## 特性

- 不对日志进行全文索引。通过存储压缩非结构化日志和仅索引元数据，Loki 操作起来会更简单，更省成本。
- 通过使用与 Prometheus 相同的标签记录流对日志进行索引和分组，这使得日志的扩展和操作效率更高。
- 特别适合储存 Kubernetes Pod 日志; 诸如 Pod 标签之类的元数据会被自动删除和编入索引。

## 架构

loki的包含一下组件

- Distributor 客户端连接的组件，用于收集日志，可以多个部署
- Ingester 接收来自Distributor的日志流，并且存放到所连接的存储后端
- Querier 用来查询日志，可以直接从Ingester或者其相连的存储直接查询数据

## 一致性哈希

loki使用一致性哈希来保证数据流和Ingester的一致性，他们共同在一个哈希环上，哈希环的信息可以存放到etcd或者Consul或者内存中

## 数据存储

### 格式

一条日志在loki里面分为两部分

- 索引，也就是标签
- 数据，日志内容

![chunks_diagram](https://qiniu.li-rui.top/chunks_diagram.png)

### 后端存储

- 对于索引可以使用 Amazon DynamoDB Google Bigtable Apache Cassandra
- 数据存储可以使用 Amazon DynamoDB Google Bigtable Apache Cassandra Amazon S3 Google Cloud Storage

# 客户端

loki附带了一个日志收集工具 Promtail，除此外还支持

- Docker Driver
- Fluent Bit
- Fluentd

# 可视化

grafana原生支持loki数据源，也是目前唯一的可视化工具


# 多租户

loki天然支持多租户，通过http header来指定不同的租户HTTP header (X-Scope-OrgID) 

# 测试部署

```yaml
version: "3"

networks:
  loki:

services:
  loki:
    image: grafana/loki:latest
    ports:
      - "3100:3100"
    command: -config.file=/etc/loki/local-config.yaml
    networks:
      - loki

  promtail:
    image: grafana/promtail:latest
    volumes:
      - /var/log:/var/log
    command: -config.file=/etc/promtail/docker-config.yaml
    networks:
      - loki

  grafana:
    image: grafana/grafana:master
    ports:
      - "3000:3000"
    networks:
      - loki
```

# fluentd集成

