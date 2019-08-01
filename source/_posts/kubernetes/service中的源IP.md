---
title: service中的源IP
date: 2019-08-01 12:12:28
tags:
- kubernetes
---

# kube-proxy

kube-proxy就是个服务的反向代理，那么就涉及到反向代理中的源IP问题

<!--more-->

# 默认iptables模式

集群中的服务的默认模式是iptables模式，也就是说通过改变iptables规则来实现“负载均衡”

# nodeport的流量

## 默认状态

```bash
         client
             \ ^
              \ \
               v \
   node 1 <--- node 2
    | ^   SNAT
    | |   --->
    v |
 endpoint
```

可以看到，endpoint的client请求被node2代理了，因此endpoint服务中看到的远程IP为node2里面的IP而不是client的

## 获取源IP

### 配置

```yaml
#默认为 Cluster
service.spec.externalTrafficPolicy: Local
```

### 流量状态

```bash
        client
       ^ /   \
      / /     \
     / v       X
   node 1     node 2
    ^ |
    | |
    | v
 endpoint
```

