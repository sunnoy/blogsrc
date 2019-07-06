---
title: etcd集群维护
date: 2019-7-6 12:12:28
tags:
- kubernetes
---

etcd是一个基于raft算法的分布式简直对数据库

<!--more-->

# etcdctl使用

## 配置变量

etcdctl是etcd的命令行，他支持一些变量，如证书和etcd api版本

```bash
export ETCDCTL_CACERT=/etc/kubernetes/ssl/ca.pem
export ETCDCTL_CERT=/etc/kubernetes/ssl/kubernetes.pem
export ETCDCTL_KEY=/etc/kubernetes/ssl/kubernetes-key.pem
export ETCDCTL_API=3
```

## 循环查看node状态

```bash
while true; do etcdctl endpoint status --endpoints https://xxx.xxx.xx.xxx:2379  --cluster -w="table"; sleep 2 ; date "+ %H:%M:%S" ;  done
```

## 指定master

```bash
etcdctl move-leader 331fc45c0bc5ef3 --endpoints https://xxx.xxx.xx.xxx:2379
```

## 获取key目录

```bash
etcdctl get / --prefix --keys-only --endpoints https://xxx.xxx.xx.xxx:2379
```

## 获取k8s中的信息

```bash
etcdctl get /registry/namespaces/default -w="json" --endpoints https://xxx.xxx.xx.xxx:2379
```

# etcd节点数量

就是集群中的node数量中down掉多少个集群可以运行，down掉多少个集群就会异常。这个数量和ceph mon的是一致的，都是过半数才可以

半数就是可以选举的数量为 > 集群node数量/2

| node数量 | 允许down的最多个数 | down多少个异常 |
|--|--| -- |
| 10 | 4 | 5 |
| 9 | 4 | 5 |
| 8 | 3 | 4 |
| 7 | 3 | 4 |
| 6 | 2 | 3 |
| 5 | 2 | 3 |
| 4 | 1 | 2 |
| 3 | 1 | 2 |
| 2 | 0 | 0 |
| 1 | 0 | 0 |

# 备份etcd

## 备份

```bash
etcdctl snapshot save snapshot.db
cp -fp snapshot.db snapshot-$(date +'%Y%m%d%H%M').db
```

## 恢复

```bash
!/bin/bash

ETCD_1=10.1.0.5
ETCD_2=10.1.0.6
ETCD_3=10.1.0.7

for i in ETCD_1 ETCD_2 ETCD_3
do

export ETCDCTL_API=3
etcdctl snapshot restore snapshot.db \
--data-dir=/var/lib/etcd \
--name $i \
--initial-cluster ${ETCD_1}=http://${ETCD_1}:2380,${ETCD_2}=http://${ETCD_2}:2380,${ETCD_3}=http://${ETCD_3}:2380 \
--initial-cluster-token k8s_etcd_token \
--initial-advertise-peer-urls http://$i:2380 && \
mv /var/lib/etcd/ etcd_$i

done
```





