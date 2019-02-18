---
title: kubernetes-pv和pvc
date: 2018-11-04 12:12:28
tags:
- kubernetes
---
## kubernetes-pv和pvc

### pv

#### 新建

```yml
apiVersion: v1
kind: PersistentVolume
metadata:
  name:  pv1
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    path: /data/k8s
    server: 10.151.30.57
```
<!--more-->

#### 访问模式

- ReadWriteOnce（RWO）：读写权限，但是只能被单个节点挂载
- ReadOnlyMany（ROX）：只读权限，可以被多个节点挂载
- ReadWriteMany（RWX）：读写权限，可以被多个节点挂载

#### Capacity

理解为存储空间

#### persistentVolumeReclaimPolicy

回收策略



