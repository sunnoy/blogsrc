---
title: kubernets资源配额
date: 2019-08-09 12:12:28
tags:
- kubernetes
---

# 容器其外的资源

我们知道创建pod的时候可以配置资源限制，那么如果没有配置呢？是不是就会疯狂占用呢？集群通过资源配额和限制来管理

<!--more-->

# 资源配额介绍

## 启用

资源配额是对一个namespace来的资源总量来限制。因此是一个namespace资源，要想使用资源配额需要在api server上开启

```bash
--enable-admission-plugins=ResourceQuota
```

## 可以配额的资源

### 计算资源

- limits.cpu	Across all pods in a non-terminal state, the sum of CPU limits cannot exceed this value.
- limits.memory	Across all pods in a non-terminal state, the sum of memory limits cannot exceed this value.
- requests.cpu	Across all pods in a non-terminal state, the sum of CPU requests cannot exceed this value.
- requests.memory	Across all pods in a non-terminal state, the sum of memory requests cannot exceed this value.

### 存储资源

- requests.storage	Across all persistent volume claims, the sum of storage requests cannot exceed this value.
- persistentvolumeclaims	The total number of persistent volume claims that can exist in the namespace.
- <storage-class-name>.storageclass.storage.k8s.io/requests.storage	Across all persistent volume claims associated with the storage-class-name, the sum of storage requests cannot exceed this value.
- <storage-class-name>.storageclass.storage.k8s.io/persistentvolumeclaims	Across all persistent volume claims associated with the storage-class-name, the total number of persistent volume claims that can exist in the namespace.

### 对象资源

- configmaps	The total number of config maps that can exist in the namespace.
- persistentvolumeclaims	The total number of persistent volume claims that can exist in the namespace.
- pods	The total number of pods in a non-terminal state that can exist in the namespace. A pod is in a terminal state if .status.phase in (Failed, Succeeded) is true.
- replicationcontrollers	The total number of replication controllers that can exist in the namespace.
- resourcequotas	The total number of resource quotas that can exist in the namespace.
- services	The total number of services that can exist in the namespace.
- services.loadbalancers	The total number of services of type load balancer that can exist in the namespace.
- services.nodeports	The total number of services of type node port that can exist in the namespace.
- secrets	The total number of secrets that can exist in the namespace.

# 使用资源配额

## 创建一个测试namespace

```bash
kubectl create ns res
```

## 相关字段

```bash
kubectl explain ResourceQuota.spec

hard	<map[string]string>
    hard is the set of desired hard limits for each named resource. More info:
    https://kubernetes.io/docs/concepts/policy/resource-quotas/

scopeSelector	<Object>
    scopeSelector is also a collection of filters like scopes that must match
    each object tracked by a quota but expressed using ScopeSelectorOperator in
    combination with possible values. For a resource to match, both scopes AND
    scopeSelector (if specified in spec), must be matched.

scopes	<[]string>
    A collection of filters that must match each object tracked by a quota. If
    not specified, the quota matches all objects.

```

## 设定配额

ResourceQuota后期可以动态修改

```bash
cat <<EOF > compute-resources.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-resources
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "1"
    limits.memory: 1Gi
EOF
kubectl create -f ./compute-resources.yaml --namespace=res
```

创建完以后就会出现配额使用情况

```yaml
apiVersion: v1
items:
- apiVersion: v1
  kind: ResourceQuota
  metadata:
    creationTimestamp: "2019-08-09T03:34:55Z"
    name: compute-resources
    namespace: res
    resourceVersion: "1091691"
    selfLink: /api/v1/namespaces/res/resourcequotas/compute-resources
    uid: 8038ac77-6745-4e84-8b44-43e2a64558af
  spec:
    hard:
      limits.cpu: "1"
      limits.memory: 1Gi
      requests.cpu: "1"
      requests.memory: 1Gi
  status:
    hard:
      limits.cpu: "1"
      limits.memory: 1Gi
      requests.cpu: "1"
      requests.memory: 1Gi
    used:
      limits.cpu: "0"
      limits.memory: "0"
      requests.cpu: "0"
      requests.memory: "0"
```

## 测试配额

一旦配置了配额新建的pod必须配置相关的资源限制，否则无法创建pod

```bash
Event(v1.ObjectReference{Kind:"ReplicaSet", Namespace:"res", Name:"nginx-7bb7cd8db5", UID:"d599d590-68f2-4c46-b7e3-a66f3acbabfb", APIVersion:"apps/v1", ResourceVersion:"1095977", FieldPath:""}): type: 'Warning' reason: 'FailedCreate' Error creating: pods "nginx-7bb7cd8db5-v4vhn" is forbidden: failed quota: compute-resources: must specify limits.cpu,limits.memory,requests.cpu,requests.memory
```

### 创建pod

```bash
cat <<EOF > pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: quota-mem-cpu-demo
spec:
  containers:
  - name: quota-mem-cpu-demo-ctr
    image: nginx
    resources:
      limits:
        memory: "800Mi"
        cpu: "800m" 
      requests:
        memory: "600Mi"
        cpu: "400m"
EOF
```

继续第二个相同资源请求的pod就会报错禁止创建

```bash
Error from server (Forbidden): error when creating "pod.yaml": pods "111quota-mem-cpu-demo" is forbidden: exceeded quota: compute-resources, requested: limits.cpu=800m,limits.memory=800Mi,requests.memory=600Mi, used: limits.cpu=800m,limits.memory=800Mi,requests.memory=600Mi, limited: limits.cpu=1,limits.memory=1Gi,requests.memory=1Gi
```

# 分级限额

资源配额默认是对所有的对象进行资源管理，生活告诉我们“一刀切”不是一个很好的解决办法。于是就有了资源限额中的分类管理

本质上是一个对象过滤器，包含两种

- scopeSelector
- scopes

## scopes

### 四个作用域

根据pod的一些元数据，系统分了四个scopes，可以理解为四个装不同类型的pod的桶

存活时间：
- Terminating	Match pods where .spec.activeDeadlineSeconds >= 0
- NotTerminating	Match pods where .spec.activeDeadlineSeconds is nil
资源限制配置：
- BestEffort	Match pods that have best effort quality of service.
- NotBestEffort	Match pods that do not have best effort quality of service.

其中，activeDeadlineSeconds为pod从创建到被杀掉的存活的时间，pod根据资源的限制情况可以形成三类qos

- BestEffort 不加任何资源限制
- Burstable memory or CPU request 至少有一个容器配置
- Guaranteed memory/cpu limit = memory/cpu request 所有容器都配置

### 作用域和配额资源类型

不同的作用域对应不同的配额资源类型

- BestEffort 只可以使用到 pod 配额上面
- Terminating, NotTerminating, 和 NotBestEffort 可以使用到下面的配额资源上面
    - cpu
    - limits.cpu
    - limits.memory
    - memory
    - pods
    - requests.cpu
    - requests.memory

如何理解呢？我们创建一个cpu和memory的配额，但是我们指定BestEffort的作用域，系统会报错。换成NotBestEffort就行

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-resources
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "1"
    limits.memory: 1Gi
  scopes:
    - BestEffort
#报错信息
The ResourceQuota "compute-resources" is invalid: spec.scopes: Invalid value: []core.ResourceQuotaScope{"BestEffort"}: unsupported scope applied to resource
```

## scopeSelector

scopes只有四个作用域还是太少了，能不能自定义pod桶呢？答案是肯定的。我们知道集群里面有PriorityClass，可以给pod分优先级(一种类似标签的机制)，本质上划分等级，然后进行相关的抢占机制

那么我们就可以用PriorityClass对pod进行分类，也就是说我们创建了一个自定义的scope

### 三个PriorityClass

创建“low”, “medium”, “high”三个优先级

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high
value: 1000000
globalDefault: false
description: "This priority class should be used for high service pods only."
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: medium
value: 100000
globalDefault: false
description: "This priority class should be used for medium service pods only."
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low
value: 1000
globalDefault: false
description: "This priority class should be used for low service pods only."
```

### ResourceQuota创建

分别创建三种PriorityClass类型的ResourceQuota，形成一种“绑定”

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: pods-high
spec:
  hard:
    cpu: "1000"
    memory: 200Gi
    pods: "10"
  scopeSelector:
    matchExpressions:
    - operator : In
      scopeName: PriorityClass
      values: ["high"]
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: pods-medium
spec:
  hard:
    cpu: "10"
    memory: 20Gi
    pods: "10"
  scopeSelector:
    matchExpressions:
    - operator : In
      scopeName: PriorityClass
      values: ["medium"]
---    
apiVersion: v1
kind: ResourceQuota
metadata:
  name: pods-low
spec:
  hard:
    cpu: "5"
    memory: 10Gi
    pods: "10"
  scopeSelector:
    matchExpressions:
    - operator : In
      scopeName: PriorityClass
      values: ["low"]
```

### pod作用

创建pod的时候除了进行资源配置外还应该配置相应的PriorityClass

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: high-priority
spec:
  containers:
  - name: high-priority
    image: nginx
    resources:
      requests:
        memory: "1Gi"
        cpu: "500m"
      limits:
        memory: "1Gi"
        cpu: "500m"
  priorityClassName: high
```

查看priorityClass为high的资源占用

```yaml
#kubectl get resourcequotas pods-high -o yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  annotations:
  name: pods-high
  namespace: res
spec:
  hard:
    cpu: 1k
    memory: 200Gi
    pods: "10"
  scopeSelector:
    matchExpressions:
    - operator: In
      scopeName: PriorityClass
      values:
      - high
status:
  hard:
    cpu: 1k
    memory: 200Gi
    pods: "10"
  used:
    cpu: 500m
    memory: 1Gi
    pods: "1"
```






