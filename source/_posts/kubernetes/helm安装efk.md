---
title: helm安装efk
date: 2019-6-26 12:12:28
tags:
- kubernetes
---

本文介绍efk的安装

<!--more-->

# 用到的helm仓库

## 仓库地址

- [fluentd-elasticsearch](https://github.com/kiwigrid/helm-charts/tree/master/charts/fluentd-elasticsearch)
- [kibana](https://github.com/elastic/helm-charts/blob/master/kibana/README.md)
- [elasticsearch](https://github.com/elastic/helm-charts/tree/master/elasticsearch)

## 添加helm仓库

```bash
helm repo add kiwigrid https://kiwigrid.github.io
helm repo add elastic https://helm.elastic.co
```

# efk安装

## 安装fluentd-elasticsearch

```bash
helm install kiwigrid/fluentd-elasticsearch \
     --set elasticsearch.host=elasticsearch-coord \
     --set elasticsearch.sslVerify=false \
     --name fluentd
```

## 安装elasticsearch

使用分布式安装，每一种role都可以安装多个node，要保证release name 不一样

### master

用来管理集群

```bash
helm install \
     --name es-master \
     elastic/elasticsearch \
     --version 7.1.1 \
     --set replicas=1 \
     --set minimumMasterNodes=1 \
     --set clusterHealthCheckParams="wait_for_status=yellow&timeout=1s" \
     --set nodeGroup="master" \
     --set-string roles.master=true \
     --set-string roles.ingest=false \
     --set-string roles.data=false 
```

### data

用来存储数据

```bash
helm install \
     --name es-data \
     elastic/elasticsearch \
     --version 7.1.1 \
     --set replicas=1 \
     --set minimumMasterNodes=1 \
     --set clusterHealthCheckParams="wait_for_status=yellow&timeout=1s" \
     --set nodeGroup="data" \
     --set-string roles.master=false \
     --set-string roles.ingest=false \
     --set-string roles.data=true 


```

### coord

每个role默认接受client请求，可以专门负责client请求

```bash
helm install \
     --name es-coord \
     elastic/elasticsearch \
     --version 7.1.1 \
     --set replicas=1 \
     --set minimumMasterNodes=1 \
     --set clusterHealthCheckParams="wait_for_status=yellow&timeout=1s" \
     --set nodeGroup="coord" \
     --set-string roles.master=false \
     --set-string roles.ingest=false \
     --set-string roles.data=false 

```

### ingest

执行日志处理pipeline操作

```bash
helm install \
     --name es-ingest \
     elastic/elasticsearch \
     --version 7.1.1 \
     --set replicas=1 \
     --set minimumMasterNodes=1 \
     --set clusterHealthCheckParams="wait_for_status=yellow&timeout=1s" \
     --set nodeGroup="ingest" \
     --set-string roles.master=false \
     --set-string roles.ingest=true \
     --set-string roles.data=false 

```

## 安装kibana

```bash
helm install \
    --name kibana \
    elastic/kibana \
    --version 7.1.1 \
    --set service.type=NodePort 
```