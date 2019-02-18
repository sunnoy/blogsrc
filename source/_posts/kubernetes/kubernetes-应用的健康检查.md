---
title: kubernetes-应用的健康检查
date: 2018-11-04 12:12:28
tags:
- kubernetes
---

## kubernetes-应用的健康检查
 kubernates使用Liveness 和 Readiness来检查应用健康状态，后端pod健康检查
 Readiness用来检查应用是否为处理过来的请求做好准备，是否在运行
 Liveness确保应用是健康的而且可以出来过来的请求，是否可以返回正确状态码

### 1. 启动应用

- 创建yaml文件
```yaml
kind: List
apiVersion: v1
items:
- kind: ReplicationController
  apiVersion: v1
  metadata:
    name: frontend
    labels:
      name: frontend
  spec:
    replicas: 1
    selector:
      name: frontend
    template:
      metadata:
        labels:
          name: frontend
      spec:
        containers:
        - name: frontend
          image: katacoda/docker-http-server:health
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 1
            timeoutSeconds: 1
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 1
            timeoutSeconds: 1
- kind: ReplicationController
  apiVersion: v1
  metadata:
    name: bad-frontend
    labels:
      name: bad-frontend
  spec:
    replicas: 1
    selector:
      name: bad-frontend
    template:
      metadata:
        labels:
          name: bad-frontend
      spec:
        containers:
        - name: bad-frontend
          image: katacoda/docker-http-server:unhealthy
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 1
            timeoutSeconds: 1
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 1
            timeoutSeconds: 1
- kind: Service
  apiVersion: v1
  metadata:
    labels:
      app: frontend
      kubernetes.io/cluster-service: "true"
    name: frontend
  spec:
    type: NodePort
    ports:
    - port: 80
      nodePort: 30080
    selector:
      app: frontend
```
<!--more-->

- 部署应用

```bash
master $ kubectl apply -f deploy.yaml
replicationcontroller "frontend" created
replicationcontroller "bad-frontend" created
service "frontend" created
```

### 3. Readiness探针
- 上述yaml文件中的关键配置
```yaml
livenessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 1
  timeoutSeconds: 1
  
readinessProbe:  
  tcpSocket:  
    port: 8080  
  initialDelaySeconds: 5  
  periodSeconds: 10  
```
- 获取pod的状态
```bash
kubectl get pods --selector="name=bad-frontend"
```
- 获取某个pod的详细信息
```bash
pod=$(kubectl get pods --selector="name=bad-frontend" --output=jsonpath={.items..metadata.name})
kubectl describe pod $pod
```
- 进入某个pod执行命令
```bash
pod=$(kubectl get pods --selector="name=frontend" --output=jsonpath={.items..metadata.name})
kubectl exec $pod -- /usr/bin/curl -s localhost/unhealthy
```
