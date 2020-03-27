---
title: kubernetes-使用kubectl部署容器
date: 2018-11-04 12:12:28
tags:
- kubernetes
---
## kubernetes-使用kubectl部署容器

### 1 启动一个单节点集群

```bash
#启动集群
minikub start
#查看节点
kubectl get nodes
```
<!--more-->

### 2 执行一个容器部署
`kubectl run`命令将会创建一个**deployment**，该命令相等于集群级别的`docker run`

-根据镜像来创建一个deployment
```bash
#该命令创建一个名字为 http 的deployment，指定镜像为katacoda/docker-http-server:latest，复制为1
kubectl run http --image=katacoda/docker-http-server:latest --replicas=1
```
- 查看已经创建的deployment
```bash
kubectl get deployments
```
- 查看一个deployment的详细信息
```bash
kubectl describe deployment http
```
### 3 创建服务

在**deployment**的基础上使用`kubectl`命令去创建一个服务，该服务回开发端口让外界访问

- 创建一个服务并暴露端口
```bash
kubectl expose deployment http --external-ip="173.17.0.30" --port=8000 --target-port=80
```
- 访问这个服务
```bash
curl http://173.17.0.30:8000
```
### 4 执行deployment创建服务二合一

- 执行命令

```bash
kubectl run httpexposed --image=katacoda/docker-http-server:latest --replicas=1 --port=80 --hostport=8001
```
- 访问二合一命令下的服务

```bash
curl http://173.17.0.30:8001
```
- 服务状态
该命令使用了docker的端口映射
无法使用命令`kubectl get svc`看到这个服务
使用`docker ps | grep httpexposed`来查看服务的运行

### 5 扩容容器
- 增加副本
```bash
kubectl scale  deployment http --replicas=3
```
- 查看副本状态

```bash
kubectl get pods
```

- 查看具体的服务状态

```bash
kubectl describe svc http
```
