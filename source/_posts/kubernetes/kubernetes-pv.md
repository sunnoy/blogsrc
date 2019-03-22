---
title: kubernetes-pv
date: 2018-11-04 12:12:28
tags:
- kubernetes
---

# 持久化存储

是个对象，支持的后端有多个

![TIM截图20180828155146](https://qiniu.li-rui.top/TIM截图20180828155146.png)

<!--more-->
nfs示例

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
    path: /data
    server: 11.11.11.11
```
查看

```bash
kubectl get pv
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS    CLAIM              STORAGECLASS   REASON    AGE
pv1       1Gi        RWO            Recycle          Bound     default/pvc2-nfs                            5m

```

# 相关字段配置

## capacity

描述该pv属性，暂时支持存储空间

## accessModes

访问模式主要描述pv是否可以多个node读数据和写数据，分为

**ReadWriteOnce(RWO)**：单节点读写，多节点读
**ReadOnlyMany(ROX)**：多节点读
**ReadWriteMany(RWX)**：多节点读写

## persistentVolumeReclaimPolicy

当pod不需要pv的时候，里面的资源怎么办，三个策略

**Retain**：保留数据，需要管理员手工清理数据
**Recycle**：资源回收，清除 PV 中的数据，效果相当于执行 rm -rf /pv/*
**Delete**：删除，直接删除pv

# 状态

pv是有生命周期的，也就出现了pv的状态流转

**Available**：表示可用状态，还未被任何 PVC 绑定
**Bound**：表示 PV 已经被 PVC 绑定
**Released**：PVC 被删除，但是资源还未被集群重新声明
**Failed**： 表示该 PV 的自动回收失败

# pvc

也是一个对象

示例：

```yml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvc2-nfs
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```
查看

```bash
kubectl get pvc
NAME       STATUS    VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc2-nfs   Bound     pv1       1Gi        RWO                           6s

```

建立pvc的时候，系统会自动寻找可用的pv去和pvc去绑定，pvc的空间要小于pv的空间大小。

# 使用pvc

利用nginx来使用

示例：

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 1
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
        volumeMounts:
        - name: www
          subPath: nginxpvc-test
          mountPath: /usr/share/nginx/html
      volumes:
      - name: www
        persistentVolumeClaim:
          claimName: pvc2-nfs
---
apiVersion: v1
kind: Service
metadata:  
  name: nginx-svc
spec:
  selector:    
    run: my-nginx
  type: NodePort
  ports:  
  - name: http
    port: 80
    targetPort: 80
    nodePort: 20086
    protocol: TCP
```

## dvp(Dynamic Volume Provisioning)

对接使用ceph rbd作为dvp

### cephx认证

```bash
ceph auth get-key client.admin |base64
QVFDUWFsMWFuUWlhRHhBQXpFMGpxMSsybEFjdHdSZ3J3M082YWc9PQ==
```

### k8s集群密钥

```yml
apiVersion: v1
kind: Secret
metadata:
  name: ceph-secret-admin
type: "kubernetes.io/rbd"  
data:
  key: QVFDUWFsMWFuUWlhRHhBQXpFMGpxMSsybEFjdHdSZ3J3M082YWc9PQ==
```

查看

```bash
kubectl get secret
```

### 创建Storage Classes

Storage Classes存在两种类型，一个是k8s官方支持的，一个是社区支持的

```bash
#官方
https://kubernetes.io/docs/concepts/storage/storage-classes/#provisioner

#社区
https://github.com/kubernetes-incubator/external-storage
```

rbd官方支持的不要需要使用社区的

先下载

```bash
git clone https://github.com/kubernetes-incubator/external-storage.git
cd external-storage/ceph/rbd/deploy/

kubectl apply -f rbac/
```

```yml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: rbd
provisioner: ceph.com/rbd
parameters:
  monitors: 10.222.78.12:6789
  adminId: admin
  #adminSecretName为上面创建的认证
  adminSecretName: ceph-secret-admin
  adminSecretNamespace: default
  pool: rbd
  userId: admin
  userSecretName: ceph-secret-admin
```

查看

```bash
kubectl get storageclass
```

### 创建dyn-pv-claim

```yml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ceph-rbd-dyn-pv-claim
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: rbd
  resources:
    requests:
      storage: 1Gi
```

查看详细信息

```bash
kubectl describe pvc/ceph-rbd-dyn-pv-claim
```

### 使用dpvc

```yml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: rbd-dyn-pvc-pod
  name: ceph-rbd-dyn-pv-pod1
spec:
  containers:
  - name: ceph-rbd-dyn-pv-busybox1
    image: busybox
    command: ["sleep", "60000"]
    volumeMounts:
    - name: ceph-dyn-rbd-vol1
      mountPath: /mnt/ceph-dyn-rbd-pvc/busybox
      readOnly: false
  volumes:
  - name: ceph-dyn-rbd-vol1
    persistentVolumeClaim:
      claimName: ceph-rbd-dyn-pv-claim
```

### 查看rbd

```bash
rbd list
```













