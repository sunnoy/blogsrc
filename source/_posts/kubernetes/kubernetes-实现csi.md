---
title: kubernetes-实现csi
date: 2019-7-18 12:12:28
tags:
- kubernetes
---

# csi介绍

Container Storage Interface(csi)是 Kubernetes v1.8+ 支持的一种存储插件扩展方式。

本文以阿里云为例

<!--more-->

# csi部署机制

csi的部署相对来说比较复杂，不过kubernetes官方提供一些驱动和对象，主要有

- Kubernetes CSI Sidecar Containers
- Kubernetes CSI objects

## Sidecar Containers

Sidecar Containers的主要目的是为了简化部署和开发流程，本质上是API server和csi驱动的的通讯桥梁，这些开发者直接调用就行了，不必在写相关的代码。这些容器有Kubernetes Storage community来维护，这些容器主要包含

- external-provisioner 通过api server监视PersistentVolumeClaim对象，对csi调用CreateVolume创建volume，然后创建pv
- external-attacher 通过API server监视VolumeAttachment对象，触发Controller[Publish|Unpublish]Volume操作对csi驱动，主要完成Kubernetes volume attach/detach的操作，也即是将存储放入集群内
- external-snapshotter 通过API server监视VolumeSnapshot和VolumeSnapshotContent对象。然后调用csi执行快照操作
- external-resizer 通过API server监视PersistentVolumeClaim对象，触发csi中的ControllerExpandVolume操作 

- node-driver-registrar 通过csi的NodeGetInfo方法来获取node信息，通过kubelet的插件机制注册到kubelet上面
- cluster-driver-registrar 向api server注册csi驱动，主要是整一些kubernetes和csi的交互方式，通常会创建CSIDriver Object 
- livenessprobe csi驱动的监控模块

## CSI objects

有两个csi对象专门对csi使用

- CSIDriver 主要完成两个目的
  - 简化csi服务发现
  - 定制化kubernetes和csi交互行为，比如只对特定的pod进行挂载
- CSINodeInfo 
  - node对象一致性，通过该对象kubernetes做操作将会调取CSINodeInfo中的node信息，而不是原本node对象中的信息
  - 一个标志位，当kubelet和kube-controller-manager and kubernetes scheduler通信的时候进行信息通报
  - 为kubernetes创建存储拓扑元数据

# 阿里云csi准备操作

本文仅仅使用云盘操作

[仓库地址](https://github.com/AliyunContainerService/csi-plugin)

## 参数依赖配置

### kubelet参数

FlexVolume使用的是node侧attach/detach，因此需要将kubelet中配置参数

```bash
vim /etc/systemd/system/kubelet.service
#添加选项
--allow-privileged=true

#重启服务
systemctl daemon-reload
systemctl restart kubelet
```

### kube-apiserver参数

```bash
vim /etc/systemd/system/kube-apiserver.service

#添加选项
--allow-privileged=true
--feature-gates=VolumeSnapshotDataSource=true,CSINodeInfo=true,CSIDriverRegistry=true

#重启服务
systemctl daemon-reload
systemctl restart kube-apiserver
```

## CRD创建

```bash
#csidriver、csinodeinfo
kubectl create -f https://raw.githubusercontent.com/kubernetes/csi-api/ab0df28581235f5350f27ce9c27485850a3b2802/pkg/crd/testdata/csidriver.yaml --validate=false 
kubectl create -f https://raw.githubusercontent.com/kubernetes/csi-api/ab0df28581235f5350f27ce9c27485850a3b2802/pkg/crd/testdata/csinodeinfo.yaml --validate=false 
```

## 创建RBAC

```bash
kubectl create -f ./deploy/rbac.yaml

#会创建
serviceaccount/alicloud-csi-plugin 
clusterrole.rbac.authorization.k8s.io/alicloud-csi-plugin 
clusterrolebinding.rbac.authorization.k8s.io/alicloud-csi-plugin 
```

# 使用csi disk

disk目前支持的csi特性为

- Disk Snapshot
- Block Volumes
- Shared Disk
- Volume Attach Limits

## 部署csi插件相关

主要部署下面几个插件

- disk-plugin
- disk-attacher
- disk-provisioner

### disk-plugin

master也要部署，pod包含两个容器

- csi-node-driver-registrar 向该node上的kubelet注册node信息
- csi-diskplugin 接收external-provisioner、external-attacher、external-snapshotter、external-resizer的请求，和阿里云sdk进行交互操作，通过/var/lib/kubelet/plugins/diskplugin.csi.alibabacloud.com/csi.sock暴漏endpointe

需要填入阿里云认证相关

- ACCESS_KEY_ID
- ACCESS_KEY_SECRET

```yaml
#deploy/disk/disk-plugin.yaml
kind: DaemonSet
apiVersion: apps/v1beta2
metadata:
  name: csi-disk-plugin
spec:
  selector:
    matchLabels:
      app: csi-disk-plugin
  template:
    metadata:
      labels:
        app: csi-disk-plugin
    spec:
      serviceAccount: alicloud-csi-plugin
      hostNetwork: true
      hostPID: true
      containers:
        - name: driver-registrar
          image: registry.cn-hangzhou.aliyuncs.com/plugins/csi-node-driver-registrar:v1.0.1
          imagePullPolicy: Always
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "rm -rf /registration/diskplugin.csi.alibabacloud.com /registration/diskplugin.csi.alibabacloud.com-reg.sock"]
          args:
            - "--v=5"
            - "--csi-address=/csi/csi.sock"
            - "--kubelet-registration-path=/var/lib/kubelet/plugins/diskplugin.csi.alibabacloud.com/csi.sock"
          env:
            - name: KUBE_NODE_NAME
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: spec.nodeName
          volumeMounts:
            - name: plugin-dir
              mountPath: /csi
            - name: registration-dir
              mountPath: /registration

        - name: csi-diskplugin
          securityContext:
            privileged: true
            capabilities:
              add: ["SYS_ADMIN"]
            allowPrivilegeEscalation: true
          image: registry.cn-hangzhou.aliyuncs.com/plugins/csi-diskplugin:v1.13.2-43e62a8e
          imagePullPolicy: "Always"
          args :
            - "--endpoint=$(CSI_ENDPOINT)"
            - "--v=5"
            - "--driver=diskplugin.csi.alibabacloud.com"
          env:
            - name: CSI_ENDPOINT
              value: unix://var/lib/kubelet/plugins/diskplugin.csi.alibabacloud.com/csi.sock
            - name: ACCESS_KEY_ID
              value: ""
            - name: ACCESS_KEY_SECRET
              value: ""
            - name: MAX_VOLUMES_PERNODE
              value: "5"
          volumeMounts:
            - name: pods-mount-dir
              mountPath: /var/lib/kubelet
              mountPropagation: "Bidirectional"
            - mountPath: /dev
              mountPropagation: "HostToContainer"
              name: host-dev
            - mountPath: /var/log/alicloud
              name: host-log
      volumes:
        - name: plugin-dir
          hostPath:
            path: /var/lib/kubelet/plugins/diskplugin.csi.alibabacloud.com
            type: DirectoryOrCreate
        - name: registration-dir
          hostPath:
            path: /var/lib/kubelet/plugins_registry
            type: DirectoryOrCreate
        - name: pods-mount-dir
          hostPath:
            path: /var/lib/kubelet
            type: Directory
        - name: host-dev
          hostPath:
            path: /dev
        - name: host-log
          hostPath:
            path: /var/log/alicloud/
  updateStrategy:
    type: RollingUpdate
```

### disk-attacher

- 通过API server监视VolumeAttachment对象，触发Controller[Publish|Unpublish]Volume操作对csi驱动，主要完成Kubernetes volume attach/detach的操作，也即是将存储放入集群内
- 通过nodeSelector master node部署

```yaml
kind: Service
apiVersion: v1
metadata:
  name: csi-disk-attacher
  labels:
    app: csi-disk-attacher
spec:
  selector:
    app: csi-disk-attacher
  ports:
    - name: dummy
      port: 12345

---
kind: StatefulSet
apiVersion: apps/v1beta1
metadata:
  name: csi-disk-attacher
spec:
  serviceName: "csi-disk-attacher"
  replicas: 1
  template:
    metadata:
      labels:
        app: csi-disk-attacher
    spec:
      nodeSelector:
         kubernetes.io/role: "master"
      serviceAccount: alicloud-csi-plugin
      containers:
        - name: csi-attacher
          image: registry.cn-hangzhou.aliyuncs.com/plugins/csi-attacher:v1.0.0
          args:
            - "--v=5"
            - "--csi-address=$(ADDRESS)"
          env:
            - name: ADDRESS
              value: /var/lib/kubelet/plugins/diskplugin.csi.alibabacloud.com/csi.sock
          imagePullPolicy: "Always"
          volumeMounts:
            - name: socket-dir
              mountPath: /var/lib/kubelet/plugins/diskplugin.csi.alibabacloud.com
      volumes:
        - name: socket-dir
          hostPath:
            path: /var/lib/kubelet/plugins/diskplugin.csi.alibabacloud.com
            type: DirectoryOrCreate
```

### disk-provisioner

- external-provisioner 通过api server监视PersistentVolumeClaim对象，对csi调用CreateVolume创建volume，然后创建pv
- master 部署

需要填入阿里云认证相关

- ACCESS_KEY_ID
- ACCESS_KEY_SECRET

```yaml
kind: Service
apiVersion: v1
metadata:
  name: csi-disk-provisioner
  labels:
    app: csi-disk-provisioner
spec:
  selector:
    app: csi-disk-provisioner
  ports:
    - name: dummy
      port: 12345

---
kind: StatefulSet
apiVersion: apps/v1beta1
metadata:
  name: csi-disk-provisioner
spec:
  serviceName: "csi-disk-provisioner"
  replicas: 1
  template:
    metadata:
      labels:
        app: csi-disk-provisioner
    spec:
      nodeSelector:
         kubernetes.io/role: "master"
      serviceAccount: alicloud-csi-plugin
      hostNetwork: true
      containers:
        - name: csi-provisioner
          image: registry.cn-hangzhou.aliyuncs.com/plugins/csi-provisioner:v1.0.1
          args:
            - "--provisioner=diskplugin.csi.alibabacloud.com"
            - "--csi-address=$(ADDRESS)"
            - "--v=5"
          env:
            - name: ADDRESS
              value: /socketDir/csi.sock
          imagePullPolicy: "Always"
          volumeMounts:
            - name: disk-provisioner-dir
              mountPath: /socketDir

        - name: csi-diskplugin
          securityContext:
            privileged: true
            capabilities:
              add: ["SYS_ADMIN"]
            allowPrivilegeEscalation: true
          image: registry.cn-hangzhou.aliyuncs.com/plugins/csi-diskplugin:v1.13.2-43e62a8e
          imagePullPolicy: "Always"
          args :
            - "--endpoint=$(CSI_ENDPOINT)"
            - "--v=5"
            - "--driver=diskplugin.csi.alibabacloud.com"
          env:
            - name: CSI_ENDPOINT
              value: unix://socketDir/csi.sock
            - name: ACCESS_KEY_ID
              value: ""
            - name: ACCESS_KEY_SECRET
              value: ""
            - name: MAX_VOLUMES_PERNODE
              value: "5"
          volumeMounts:
            - mountPath: /var/log/alicloud
              name: host-log
            - mountPath: /socketDir/
              name: disk-provisioner-dir
      volumes:
        - name: disk-provisioner-dir
          emptyDir: {}
        - name: host-log
          hostPath:
            path: /var/log/alicloud/
```

## 创建StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: csi-disk
parameters:
  fsType: ext4
  readOnly: "false"
  regionId: cn-beijing
  type: available
  zoneId: cn-beijing-c
provisioner: diskplugin.csi.alibabacloud.com
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

相关参数，和FlexVolume中创建类是一样的

- provisioner：配置为 alicloud/disk，标识StorageClass使用阿里云云盘 provisioner 插件创建。
- type：标识云盘类型，支持 cloud、cloud_efficiency、cloud_ssd、available 四种类型；其中 available 会对高效、SSD、普通云盘依次尝试创建，直到创建成功。
- regionid：期望创建云盘的区域。
- reclaimPolicy: 云盘的回收策略，默认为Delete，支持Retain。如果数据安全性要求高，推荐使用Retain方式以免误删；
- zoneid：期望创建云盘的可用区。cn-hangzhou-a,cn-hangzhou-b,cn-hangzhou-c，可以有多个
- encrypted：（可选）创建的云盘是否加密，默认情况是false，创建的云盘不加密。

# 使用测试

## 部署

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: disk-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: csi-disk
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-disk
  labels:
    app: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
        volumeMounts:
          - name: disk-pvc
            mountPath: "/data"
      volumes:
        - name: disk-pvc
          persistentVolumeClaim:
            claimName: disk-pvc
```

## 查看

获取磁盘id和名称

```bash
#pvc-c980d884-a9f8-11e9-bec1-00163e0eedd5 为磁盘名称
kubectl get pvc disk-pvc
NAME       STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
disk-pvc   Bound    pvc-c980d884-a9f8-11e9-bec1-00163e0eedd5   20Gi       RWO            csi-disk       3m20s

# VolumeHandle 为磁盘id
kubectl describe pv pvc-c980d884-a9f8-11e9-bec1-00163e0eedd5  | grep VolumeHandle
    VolumeHandle:      d-2zehzido76fpvmpsg0ra
```

## 其他功能

[disk-snapshot](https://github.com/AliyunContainerService/csi-plugin/blob/master/docs/disk-snapshot.md)

[disk-block](https://github.com/AliyunContainerService/csi-plugin/blob/master/docs/disk-block.md)

[disk-shared](https://github.com/AliyunContainerService/csi-plugin/blob/master/docs/disk-shared.md)

[volume-attach-limits](https://github.com/AliyunContainerService/csi-plugin/blob/master/docs/disk-volume-limits.md)

