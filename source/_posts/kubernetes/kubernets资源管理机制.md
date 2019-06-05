---
title: kubernets资源管理机制
date: 2019-6-5 12:12:28
tags:
- kubernetes
---

![资源](https://qiniu.li-rui.top/资源.jpg)

<!--more-->

# node资源划分

## 总分类

node上的资源在kubernetes中被分为

- Node Capacity node的资源总和
- System-Reserved system使用的预留资源
- Kube-Reserved 集群组件使用的资源如 docker daemon, kubelet, kube proxy
- Kubelet Allocatable 供kubelet使用的资源，也就是创建pod所使用的资源

其中还有一个阈值 Hard-Eviction-Threshold，表示node上的剩余资源量，当小于这个资源量的时候就会发生pod驱逐

这几种资源的关系是

```bash
[Allocatable] = [Node Capacity] - [Kube-Reserved] - [System-Reserved] - [Hard-Eviction-Threshold]
```

## 相关配置

上面的资源类型配置可以通过kubelet启动参数来配置

- --kube-reserved=cpu=500m,memory=5Mi --kube-reserved-cgroup
- --system-reserved --system-reserved-cgroup

# 容器资源请求

## 配置请求

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend
spec:
  containers:
  - name: db
    image: mysql
    resources:
      requests:
        memory: "64Mi"
        # 1个虚拟内核就是1000m
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

## 容器的qos

针对创建资源的类型可以为容pod分为三个qos等级 

### Guaranteed

优先级最高

- 所有容器均需要配置资源限制
- 资源配置的requests和limits相等

### Burstable

- 至少一个容器配置资源限制
- 资源配置需要request

### BestEffort

优先级最低

- 一个容器都没有配置

## 调度机制

Kubernetes scheduler 会根据资源请求去寻找node，判断node的资源余量是根据pod中的资源resources来计算，尽管实际使用的会小于请求的用量。当发现没有node可以使用的时候，pod就会进入pending状态。

## 超限机制

### 内存

- 超过 memory limit 直接杀掉 出现oom
- 超过 memory request node上发生资源不够的时候，最先被驱逐

### cpu

- 超过cpu不会被杀掉

# namespace 资源限制

为一个namespace进行总的资源配置，对整个集群进行资源配置，使用Resource Quotas对象。主要作用是

- 为整个namespace创建资源限额
- 当整个namespace中的资源限额用完的时候，如果再进行创建的时候就会进行创建pod

# 全局默认资源限制

为了防止某一个容器或者pod将namespace的资源抢占完全，需要进行创建Limit Ranges对象。主要作用为

- pod或者容器级别，创建pod的时候自动加上限制
- 如果创建的pod或者容器所申请的资源超过了Limit Ranges对象的配置，就会禁止该pod的创建

