---
title: podIP段更改
date: 2019-12-20 08:12:28
tags:
- kubernetes
---

# IP段更改

部署集群的时候没有规划好集群网络，导致和虚拟机的网段冲突

<!--more-->

182老的地址段，32为新的地址段

# node更改

主要是修改pod CIDR字段

```bash
kubectl get no -o yaml >> file.yaml | sed -i "s~podCIDR\:\ 182~podCIDR\:\ 32~" file.yaml| kubectl delete no $hostname && kubectl create -f file.yaml
```

# flannel配置修改

```bash
kubectl edit cm kube-flannel-cfg -n kube-system
```

修改为

```bash
net-conf.json: | { "Network": "32.22.0.0/16", "Backend": { "Type": "vxlan" } }
```

# 重新部署flannel和coredns

```bash
kubectl delete pod --selector=app=flannel -n kube-system --grace-period=0
kubectl delete pod --selector=k8s-app=kube-dns -n kube-system --grace-period=0
```

# 重新部署集群内的pod



