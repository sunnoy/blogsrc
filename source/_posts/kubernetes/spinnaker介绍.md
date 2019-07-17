---
title: spinnaker介绍
date: 2019-07-16 12:12:28
tags:
- kubernetes
---

# spinnaker概览

是个cd工具，可以对接多种运行环境，包含容器和虚拟机。核心关注两大方面

- 应用管理
- 应用部署

<!--more-->

# 概念封装

## 应用 application

应用是核心，一组vm或者pod是server group。多个server group是一个cluster。一个应用包含多个cluster。

请求流量通过 load balancer 经过firewall后到达sever group。 

![clusters](https://qiniu.li-rui.top/clusters.jpg)

## pipeline

pipeline是由多个stage组成的，每个stage可以包含多种类型的操作。

![pipelines](https://qiniu.li-rui.top/pipelines.jpg)

## strategy

可以划分多种部署策略，蓝绿部署以及金丝雀发布等

![deployment-strategies](https://qiniu.li-rui.top/deployment-strategies.jpg)


# spinnaker架构

spinnaker本身就是一组微服务的集合，并有专门的部署工具 halyard。部署环境需要代理工具。

![spinnaker-ar](https://qiniu.li-rui.top/spinnaker-ar.png)

# kubernetes使用

与kubernetes的集成有两种方式

- legacy 对应api版本为v1，这种方式创建k8s对象主要是通过ui来完成
- manifets based 对应api版本为v2，这种方式创建对象是通过k8s的yaml文件来完成

通过helm安装默认会使用 mainifest 方式集成k8s

## application

使用spinnaker是从应用开始的，然后是创建pipeline，以及一些插件的使用。

[这里](https://www.spinnaker.io/guides/tutorials/codelabs/)有官方提供的一些示例

创建应用后就会到 cluster 页面

## cluster

### server group

在k8s中server group可以是下面两种对象

- deployment
- replicaSet

通过 kubernetes Manifest 来创建server group 

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

部署完后就是这样

![spin-cluster](https://qiniu.li-rui.top/spin-cluster.png)





