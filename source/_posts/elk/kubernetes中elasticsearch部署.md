---
title: kubernetes中elasticsearch部署
date: 2019-05-08 12:12:28
tags:
- efk
---

# elasticsearch介绍

elasticsearch是一个集中存储的工具，是elk技术栈中的核心组件

<!--more-->

# 部署

这里使用chart部署，进行分布式部署。[chart地址](https://github.com/helm/charts/tree/master/stable/elasticsearch)

## 动态pv准备

elasticsearch是有状态的，因此需要使用statefulsets部署。所以整一个动态pv就很方便

下面给出nfs部署参考，注意替换相关字段

```bash
helm install nfs-client-provisioner --name nfs --set nfs.server=x.x.x.x --set nfs.path=/data/xxxx
```

## 分布式

### master选举

单节点可以部署，不参与选举。

在分布式部署中，master法定节点数量计算为

```bash
(master_nodes / 2) + 1 
```

计算结果去整六零，比如存在3个master node。那么就是2

```yaml
  env:
    # 最小master节点，防止脑裂
    MINIMUM_MASTER_NODES: "2"
```

### shard副本数

要达到法定的分片才可以接受写

```bash
int( (primary + 索引中配置的number_of_replicas) / 2 ) + 1
```

## 变量文件准备

文件 val.yaml 内容

```yaml
cluster:
  #集群名称
  name: "es-on-k8s"
  env:
    # 最小master节点，防止脑裂
    MINIMUM_MASTER_NODES: "1"

client:
  serviceType: NodePort
  replicas: 1
master:
  name: master
  replicas: 1
  heapSize: "512m"
  persistence:
    enabled: true
    accessMode: ReadWriteOnce
    name: data
    size: "4Gi"
    storageClass: "nfs-client"

data:
  name: data
  replicas: 1
  heapSize: "512m"
  persistence:
    enabled: true
    accessMode: ReadWriteOnce
    name: data
    size: "40Gi"
    storageClass: "nfs-client"
  terminationGracePeriodSeconds: 3600
  resources:
    limits:
      cpu: "1"
      # memory: "2048Mi"
    requests:
      cpu: "25m"
      #memory: "1536Mi"
  podDisruptionBudget:
    enabled: false
    # minAvailable: 1
    #maxUnavailable: 1
```

## 部署

```bash
helm install --name es -f es-cluster.yaml elasticsearch --namespace=efkdd
```