---
title: nfs的动态pv
date: 2019-05-08 12:12:28
tags:
- kubernetes
---

# 动态pv

动态pv与静态pv相比有着很多优势，尤其是当有多个pvc时。动态pv是需要StorageClass提供。

<!--more-->

到目前为止可以使用动态pv的后端存储

![pv后端](https://qiniu.li-rui.top/pv后端.png)

# nfs Provisioner

从上面可以看到nfs没有原生的Provisioner支持，因此需要第三方的，这里选择[Kubernetes NFS-Client Provisioner](https://github.com/kubernetes-incubator/external-storage/tree/master/nfs-client)

Kubernetes NFS-Client Provisioner 只是一个对接nfs server和kubernets中pvc的中间服务，所以需要外部的nfs服务。Kubernetes NFS-Client Provisioner会创建相应的StorageClass

# NFS-Client Provisioner 安装

使用helm安装

```bash
helm install stable/nfs-client-provisioner --set nfs.server=x.x.x.x --set nfs.path=/data/promd
```

安装后查看

```bash
kubectl get storageclasses.storage.k8s.io
NAME         PROVISIONER                                AGE
nfs-client   cluster.local/nfs-nfs-client-provisioner   119s
```

# 使用动态pv

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: nfs-client
  resources:
    requests:
      storage: 30Gi
```

# 如何运作

## StorageClass创建pv

pvc创建以后，StorageClass就会通知其所属的置备程序去创建持久卷

```bash
kubectl describe pv pvc-7f99e4e3-6f0b-11e9-ac77-525400f209ec
Name:            pvc-7f99e4e3-6f0b-11e9-ac77-525400f209ec
Labels:          <none>
Annotations:     pv.kubernetes.io/provisioned-by: cluster.local/nfs-nfs-client-provisioner
Finalizers:      [kubernetes.io/pv-protection]
StorageClass:    nfs-client
Status:          Bound
Claim:           default/test
Reclaim Policy:  Delete
Access Modes:    RWO
VolumeMode:      Filesystem
Capacity:        30Gi
Node Affinity:   <none>
Message:
Source:
    Type:      NFS (an NFS mount that lasts the lifetime of a pod)
    Server:    xx.x.x.xxx
    Path:      /data/promd/default-test-pvc-7f99e4e3-6f0b-11e9-ac77-525400f209ec
    ReadOnly:  false
Events:        <none>
```

## pvc和pv绑定

```bash
kubectl describe pvc test

Name:          test
Namespace:     default
StorageClass:  nfs-client
Status:        Bound
Volume:        pvc-7f99e4e3-6f0b-11e9-ac77-525400f209ec
Labels:        <none>
Annotations:   kubectl.kubernetes.io/last-applied-configuration:
                 {"apiVersion":"v1","kind":"PersistentVolumeClaim","metadata":{"annotations":{},"name":"test","namespace":"default"},"spec":{"accessModes":...
               pv.kubernetes.io/bind-completed: yes
               pv.kubernetes.io/bound-by-controller: yes
               volume.beta.kubernetes.io/storage-provisioner: cluster.local/nfs-nfs-client-provisioner
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:      30Gi
Access Modes:  RWO
VolumeMode:    Filesystem
Events:
  Type       Reason                 Age                From                                                                                                                       Message
  ----       ------                 ----               ----                                                                                                                       -------
  Normal     ExternalProvisioning   12m (x2 over 12m)  persistentvolume-controller                                                                                                waiting for a volume to be created, either by external provisioner "cluster.local/nfs-nfs-client-provisioner" or manually created by system administrator
  Normal     Provisioning           12m                cluster.local/nfs-nfs-client-provisioner_nfs-nfs-client-provisioner-669946879f-9992k_d428a097-6efe-11e9-b1c8-e29fe073b21b  External provisioner is provisioning volume for claim "default/test"
  Normal     ProvisioningSucceeded  12m                cluster.local/nfs-nfs-client-provisioner_nfs-nfs-client-provisioner-669946879f-9992k_d428a097-6efe-11e9-b1c8-e29fe073b21b  Successfully provisioned volume pvc-7f99e4e3-6f0b-11e9-ac77-525400f209ec
Mounted By:  <none>
```

## export目内文件夹创建

```bash
ll
/data/promd/default-test-pvc-7f99e4e3-6f0b-11e9-ac77-525400f209ec
```

## 删除pvc以后

export就会变成

```bash
archived-default-test-pvc-7f99e4e3-6f0b-11e9-ac77-525400f209ec
```


