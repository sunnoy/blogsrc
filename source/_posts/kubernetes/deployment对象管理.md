---
title: deployment对象管理
date: 2019-4-25 12:12:28
tags:
- kubernetes
---

deployment是管理pod副本的高级对象，他通过副本控制器replicaset来管理pod

<!--more-->

# 创建deploymnet

## 通过文件

```bash
# apply也可以用于更新对象
kubectl apply -f xxx.yaml
```

## 通过run

```bash
#生成交互式，完了删除
kubectl run test --rm -it --image=alpine /bin/sh

# -- 后面可以接在容器内运行的命令
kubectl run test  --image=alpine -- sleep 9999

# 容器暴露端口
--port=5701
#设定环境变量
--env="DNS_DOMAIN=cluster"
#添加标签
--labels="app=hazelcast,env=prod"
#指定副本
--replicas=5
#重启策略
--restart=OnFailure
#记录升级版本
--record
#组成服务，下面一起使用
--expose --port
```

新建一个nginx

```bash
kubectl run nginx --image=nginx:1.14 --record
```

# 升级

## 升级策略

- rollingupdate 滚动升级 默认
- recreate 完全删掉旧的 然后创建新的

升级镜像版本


```bash
# 要使用 --record 来记录版本更新历史
kubectl set image deployment nginx nginx=1.16 --record
```

## 升级的操作方法

```bash
#直接编辑对象
kubectl edit

# 部分更新使用 patch
# 使用json 是该节点不被调度
kubectl patch node k8s-node-1 -p '{"spec":{"unschedulable":true}}'

# replace
# 和apply的区别是，reloace会要求对象存在，否则报错，apply不要求对象存在 ，如果没有就创建

kubectl replace -f xxx.yaml
```

## 暂停升级

```bash
kubectl rollout pause deployment/nginx
```

## 恢复升级

```bash
kubectl rollout resume deployment/nginx
```

# 回滚

## 查看历史

```bash
# 查看更新历史
kubectl rollout history deployment nginx

#查看某一个版本的详细
kubectl rollout history deployment nginx --revision=1
```
## 回滚

```bash
# 会滚到上一次版本
kubectl rollout undo deployment/abc

# 会滚到制定版本
kubectl rollout undo daemonset/abc --to-revision=3
```

# 暴露服务

```bash
#初级暴露 targetport和port 自动为80 service 那name 为sdgg
kubectl expose deployment nginx --port=80 --name=sdgg
#分离
kubectl expose deployment nginx --port=80 --target-port=8000 --name=nginx
#向service追加端口
kubectl expose service nginx --port=443 --target-port=8443 --name=nginx-https
#通过nodeport方式来暴露 --type= 为ClusterIP, NodePort, LoadBalancer, or ExternalName. 默认为ClusterIP
#--port 默认为将容器的target port和service port配置一样
#如果要分别指定 --port=443 --target-port=8443
kubectl expose deployment nginx --type=NodePort --port=80
#制定协议
--protocol=udp
#将创建的yaml格式的文件打印出来 并创建service
kubectl expose deployment nginx --port=80 --type=NodePort -o yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: "2019-04-25T03:18:45Z"
  labels:
    run: nginx
  name: nginx
  namespace: default
  resourceVersion: "4025928"
  selfLink: /api/v1/namespaces/default/services/nginx
  uid: d58f8ffc-6708-11e9-ac77-525400f209ec
spec:
  clusterIP: 22.68.132.230
  externalTrafficPolicy: Cluster
  ports:
  - nodePort: 27008
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    run: nginx
  sessionAffinity: None
  type: NodePort
status:
  loadBalancer: {}
```

# 水平缩放

## 手动

```bash
# 设置为三个副本
kubectl scale deployment nginx --replicas=3 --record
#如果当前副本是2 设置为3个副本
kubectl scale --current-replicas=2 --replicas=3 deployment/mysql --record
```

## 基于cpu自动缩放

### 必备条件

- 需要安装metrics-server
- 设定资源请求 --requests=cpu=200m

HPA会抓取metrics-server的监控指标中的cpu使用量，然后和这个容器requests=cpu进行对比，利用公式 

```bash
desiredReplicas = ceil[currentReplicas * ( currentMetricValue / desiredMetricValue )] 
```
反之出发缩容操作。需要注意的是缩容操作的时间间隔为5min，扩容操作的时间间隔为3min。

### 创建deployment

```bash
#200m (五分之一核) 
kubectl run nginx --image=nginx:1.14 --requests=cpu=200m --expose --port=80
```

### 创建HPA

```bash
kubectl autoscale deployment nginx --max=3 --min=1 --cpu-percent=5  

#获取状态
#可以通过kubectl scale deployment nginx --replicas=3 --record 来手动调整 
kubectl get horizontalpodautoscalers.autoscaling
NAME    REFERENCE          TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
nginx   Deployment/nginx   <unknown>/10%   1         3         1          2m41s
```

### 压测

循环请求

```bash
while true; do wget -q -O- http://nginx; done
```

伸缩过程

```bash
kubectl get horizontalpodautoscalers.autoscaling --watch
#开始压测
NAME    REFERENCE          TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
nginx   Deployment/nginx   0%/5%     1         3         1          109s
nginx   Deployment/nginx   5%/5%   1     3     1     3m30s
nginx   Deployment/nginx   27%/5%   1     3     1     4m30s
nginx   Deployment/nginx   27%/5%   1     3     3     4m45s
nginx   Deployment/nginx   4%/5%   1     3     3     5m30s

#停止压测后，恢复到单个副本
nginx   Deployment/nginx   0%/5%   1     3     3     6m30s
nginx   Deployment/nginx   0%/5%   1     3     3     10m
nginx   Deployment/nginx   0%/5%   1     3     3     11m
nginx   Deployment/nginx   0%/5%   1     3     1     11m
```



