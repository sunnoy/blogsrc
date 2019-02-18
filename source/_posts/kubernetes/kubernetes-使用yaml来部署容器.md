---
title: kubernetes-使用yaml来部署容器
date: 2018-11-04 12:12:28
tags:
- kubernetes
---
## kubernetes-使用yaml来部署容器

### 1. 创建一个deployedment

- 创建一个yaml文件deployment.yaml
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: webapp1
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: webapp1
    spec:
      containers:
      - name: webapp1
        image: katacoda/docker-http-server:latest
        ports:
        - containerPort: 80
```
<!--more-->

- 依据文件进行创建**deployment**

```bash
kubectl create -f deployment.yaml
#查看已经存在的deployment
kubectl get deployment
#查看具体deployment
kubectl describe deployment webapp1
```

### 2 创建服务

- 创建服务yaml文件

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp1-svc
  labels:
    app: webapp1
spec:
  type: NodePort
  ports:
  - port: 80
    nodePort: 30080
  selector:
    app: webapp1
```
- 执行yaml文件创建服务

```bash
kubectl create -f service.yaml
```
- 同样使用命令监控服务

```bash
kubectl get svc
kubectl describe svc webapp1-svc
#访问服务
curl host01:30080
```
### 3. 扩容

- deployment.yaml文件添加副本数量

```yaml
replicas: 1
```

- 执行启用配置

```bash

kubectl apply -f deployment.yaml
```

- 查看pods

```bash
kubectl get pods
```




