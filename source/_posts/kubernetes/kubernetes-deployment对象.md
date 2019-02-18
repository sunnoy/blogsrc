---
title: kubernetes-deployment对象
date: 2018-11-04 12:12:28
tags:
- kubernetes
---
## kubernetes-deployment对象

#### 字段解释

```yml
apiVersion: apps/v1
kind: Deployment
#说明对象的一些属性数据
metadata:
  #对象的名称
  name: my-nginx1
spec:
  selector:
    matchLabels:
      #下面的键值对要和template.metadata.labels一致
      run: my-nginx2
  replicas: 3
  template:
    metadata:
      labels:
        #这里的键值对要和spec.selector.matchLabels一致
        run: my-nginx3
    spec:
      containers:
      #下面的name是docker创建容器指定的name
      - name: my-nginx4
        image: nginx
        ports:
        - containerPort: 80
```

<!--more-->

#### 创建

```bash
kubectl create -f deployment-nginx.yml
```
#### 删除

```bash
kubectl delete deployment nginx-deployment
```

#### 查看

```bash
#总的
kubectl get deployments
#查看创建状态
kubectl rollout status deployment/nginx-deployment
#查看详细信息
kubectl describe deployments
#输出到文件
kubectl get deployment nginx-deployment -o yaml
```

#### 更新

```bash
#使用命令
kubectl set image deployment/nginx-deployment nginx=nginx:1.9.1
#直接修改文件
kubectl edit deployment/nginx-deployment
```

#### 缩放

```bash
kubectl scale deployment nginx-deployment --replicas=10
```

#### 暂停

```bash
#先暂停
kubectl rollout pause deployment/nginx-deployment
#再更新
kubectl set image deploy/nginx-deployment nginx=nginx:1.9.1
#恢复
kubectl rollout resume deploy/nginx-deployment
```

#### 回滚

```bash
#版本查看
kubectl rollout history deployment/nginx-deployment
#查看指定版本信息
kubectl rollout history deployment/nginx-deployment --revision=2
#回到上次版本
kubectl rollout undo deployment/nginx-deployment
#回到指定版本
kubectl rollout undo deployment/nginx-deployment --to-revision=2

```

