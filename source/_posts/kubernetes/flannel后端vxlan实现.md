---
title: flannel后端vxlan实现
date: 2020-01-13 12:12:28
tags:
- kubernetes
---

![flannel](https://qiniu.li-rui.top/flannel.png)

# 新的node加入的时候

需要加入三中信息

<!--more-->

```bash
# 路由
ip route list | grep "173.20.4.0"
173.20.4.0/24 via 173.20.4.0 dev flannel.1 onlink

# arp静态表
ip -d neigh show dev flannel.1 | grep "99:2f:91:c3:ca:6e"
173.20.4.0 lladdr 99:2f:91:c3:ca:6e PERMANENT

# fdb表
bridge fdb show | grep "99:2f:91:c3:ca:6e"
99:2f:91:c3:ca:6e dev flannel.1 dst 173.25.16.3 self permanent
```

# 数据包流程

从一个node上的pod到另一个node上的pod

## 通过路由计算获取对端IP

```bash
ip route get 173.20.4.88
173.20.4.88 via 173.20.4.0 dev flannel.1 src 173.20.0.0
```

## 通过对端IP获取静态arp

其实到这里都是pod网段在玩

```bash
ip neigh show dev flannel.1 | grep "173.20.4"
173.20.4.0 lladdr 99:2f:91:c3:ca:6e PERMANENT
```

# 通过fdb获取对端nodeIP

```bash
# 进去flannel容器里面执行
bridge fdb show | grep "99:2f:91:c3:ca:6e"
99:2f:91:c3:ca:6e dev flannel.1 dst 173.25.16.3 self permanent
```