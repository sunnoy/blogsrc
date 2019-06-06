---
title: kubernetes强制删除namespace
date: 2019-6-6 12:12:28
tags:
- kubernetes
---

# 级联对象

kubernetes中对象是有包含关系的，比如创建一个deployment对象就会创建一个rs对象，rs对象去创建pod对象。

在集群中，一个对象创建了另外对象，这个对象就是owner，他所创建的对象就是该owner的Dependent。

可以通过查看Dependent对象的metadata.ownerReferences字段来看他的owner对象。

<!--more-->

# 垃圾回收

在集群中进行删除owner对象操作时，当班他的Dependent对象也删除掉时，这种删除叫级联删除。

集群支持下面几种级联删除策略，通过字段 deleteOptions.propagationPolicy 控制

- orphan 删除对象时，不自动删除它的 Dependent
- Foreground 删除对象时会立即删除 Owner 对象，然后垃圾收集器会在后台删除这些 Dependent
- Background 跟对象首先进入Terminating状态，对象metadata.finalizers字段是非空。然后集群会删除其Dependent对象，最终删除owner对象

## 通过api删除

```bash
# background
kubectl proxy --port=8080
curl -X DELETE localhost:8080/apis/extensions/v1beta1/namespaces/default/replicasets/my-repset \
-d '{"kind":"DeleteOptions","apiVersion":"v1","propagationPolicy":"Background"}' \
-H "Content-Type: application/json"

# Foreground
kubectl proxy --port=8080
curl -X DELETE localhost:8080/apis/extensions/v1beta1/namespaces/default/replicasets/my-repset \
-d '{"kind":"DeleteOptions","apiVersion":"v1","propagationPolicy":"Foreground"}' \
-H "Content-Type: application/json"

# Orphan
kubectl proxy --port=8080
curl -X DELETE localhost:8080/apis/extensions/v1beta1/namespaces/default/replicasets/my-repset \
-d '{"kind":"DeleteOptions","apiVersion":"v1","propagationPolicy":"Orphan"}' \
-H "Content-Type: application/json"
```

## 通过kubectl删除

kubectl默认就是级联删除，通过选项参数 --cascade 禁用级联删除。

```bash
#下面是 Orphan 删除
kubectl delete replicaset my-repset --cascade=false
```

## 默认策略

ReplicationController、ReplicaSet、StatefulSet、DaemonSet 和 Deployment，默认的垃圾收集策略是 orphan

# 无法删除的namespace

## metadata.finalizers非空

遇到metadata.finalizers字段为非空的时候，删除后对象并未删除，而是在Terminating状态。

这时只需要通过下面的命令进行编辑对象，然后将字段metadata.finalizers的数组进行清空

```bash
kubectl edit object
```

## metadata.finalizers 为 kubernetes

有一种情况是，字段metadata.finalizers的值为kubernetes，即便删除了也不会将object删除，这时需要通过API来删除。

对于namespace对象，删除方法详见[此仓库](https://github.com/ctron/kill-kube-ns)

```bash
# 使用
yum install jq culr -y
./kill-kube-ns myproject
```

关键位置

```bash
kubectl get namespace "$PROJECT" -o json | jq 'del(.spec.finalizers[] | select("kubernetes"))' | curl -s -k -H "Content-Type: application/json" -X PUT -o /dev/null --data-binary @- http://localhost:8001/api/v1/namespaces/$PROJECT/finalize 
```