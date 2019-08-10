---
title: kubernets资源限制
date: 2019-08-09 12:12:28
tags:
- kubernetes
---

# 资源限制

我们知道资源限额类似一个资源划区，区域的类型默认是namespace，还可以进行分级等更细微的区域划分

在有限额的情况下，如果一个pod没有进行资源限制是不会被创建的。资源限制可以为没有配置资源限制的pod加上资源限制。

还可以进行对pod的资源请求进行最大最小限制

<!--more-->

# LimitRange

LimitRange对象主要完成下面几个方面的内容

- 对容器或者pod限制最大/最小的计算/存储资源
- 在request and limit比率
- 对没有配置资源限制的pod和容器进行默认的计算资源限制注入

LimitRange创建完成后可以动态更改

# 启用

使用资源限制需要在api server上开启

```bash
--enable-admission-plugins=LimitRanger
```

# 关键字段

```bash
kubectl explain LimitRange.spec.limits
   default	<map[string]string>
     Default resource requirement limit value by resource name if resource limit
     is omitted.

   defaultRequest	<map[string]string>
     DefaultRequest is the default resource requirement request value by
     resource name if resource request is omitted.

   max	<map[string]string>
     Max usage constraints on this kind by resource name.

   maxLimitRequestRatio	<map[string]string>
     MaxLimitRequestRatio if specified, the named resource must have a request
     and limit that are both non-zero where limit divided by request is less
     than or equal to the enumerated value; this represents the max burst for
     the named resource.

   min	<map[string]string>
     Min usage constraints on this kind by resource name.

   type	<string>
     Type of resource that this limit applies to.
     Container 
```

# maxLimitRequestRatio

对于一种资源类型，maxLimitRequestRatio也就是最大的limit/request大小。如何理解呢？

比如：maxLimitRequestRatio=2

如下设置就会报错

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox3
spec:
  containers:
  - name: busybox-cnt01
    image: busybox
    resources:
      limits:
        memory: "300Mi"
      requests:
        memory: "100Mi"
```

报错

```bash
memory max limit to request ratio per Pod is 2, but provided ratio is 3.000000.
```

# min

min 限制的是requests的资源大小，示例

```yaml

spec:
  limits:
  - default:
      cpu: 700m
      memory: 200Mi
    defaultRequest:
      cpu: 110m
      memory: 111Mi
    max:
      cpu: 800m
      memory: 1Gi
    maxLimitRequestRatio:
      memory: "2"
    min:
      cpu: 100m
      memory: 99Mi
    type: Container
---
apiVersion: v1
kind: Pod
metadata:
  name: busybox3
spec:
  containers:
  - name: busybox-cnt01
    image: busybox
    resources:
      limits:
        cpu: 700m
        memory: 59Mi
      requests:
        cpu: 11m
        memory: 50Mi
```

报错

```bash
Error from server (Forbidden): error when creating "pod.yaml": pods "busybox3" is forbidden: [minimum cpu usage per Container is 100m, but request is 11m, minimum memory usage per Container is 99Mi, but request is 50Mi]
```

# max 

max限制的limits的资源大小

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox3
spec:
  containers:
  - name: busybox-cnt01
    image: busybox
    resources:
      limits:
        cpu: 910m
        memory: 10266Mi
      requests:
        cpu: 900m
        memory: 10245Mi
```

报错

```bash
Error from server (Forbidden): error when creating "pod.yaml": pods "busybox3" is forbidden: [maximum cpu usage per Container is 800m, but limit is 910m, maximum memory usage per Container is 1Gi, but limit is 10266Mi]
```

# pod

pod的资源限制会对pod内的容器进行按类别相加计算

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: limit-mem-cpu-per-pod
spec:
  limits:
  - max:
      cpu: "2"
      memory: "2Gi"
    type: Pod
```

# PersistentVolumeClaim

对pvc进行限制

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: storagelimits
spec:
  limits:
  - type: PersistentVolumeClaim
    max:
      storage: 2Gi
    min:
      storage: 1Gi
```

# 总的限制

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: storagelimits
spec:
  limits:
  - max:
      cpu: "2"
      memory: "2Gi"
    type: Pod
  - max:
      cpu: "800m"
      memory: "1Gi"
    min:
      cpu: "100m"
      memory: "99Mi"
    default:
      cpu: "700m"
      memory: "900Mi"
    defaultRequest:
      cpu: "110m"
      memory: "111Mi"
    maxLimitRequestRatio:
      memory: 2
    type: Container
  - type: PersistentVolumeClaim
    max:
      storage: 2Gi
    min:
      storage: 1Gi

```

