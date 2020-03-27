---
title: 测试集群中dns
date: 2019-04-23 12:12:28
tags:
- kubernetes
---

介绍了测试dns的方法

<!--more-->

# alpine

```bash
kubectl run test --rm -it --image=alpine /bin/sh

#查询内部
nslookup kubernetes

Name:      kubernetes
Address 1: 22.68.0.1 kubernetes.default.svc.cluster.local

#查询外部
nslookup www.li-rui.top
nslookup: can't resolve '(null)': Name does not resolve

Name:      www.li-rui.top
Address 1: 185.199.122.153
Address 2: 185.199.108.153
Address 3: 185.199.111.153
Address 4: 185.199.109.153
```

# 不要使用busybox

[存在bug](https://github.com/kubernetes/dns/issues/109)

