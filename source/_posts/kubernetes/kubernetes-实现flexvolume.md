---
title: kubernetes-实现flexvolume
date: 2019-7-18 12:12:28
tags:
- kubernetes
---

# flexvolume介绍

FlexVolume 是 Kubernetes v1.8+ 支持的一种存储插件扩展方式。它需要外部插件将二进制文件放到预先配置的路径中（如 /usr/libexec/kubernetes/kubelet-plugins/volume/exec/alicloud~disk/disk），并需要在系统中安装好所有需要的依赖。驱动会在特定的目录执行命令，进行存储的相关操作

本文以阿里云为例

<!--more-->

# kubelet参数

FlexVolume使用的是node侧attach/detach，因此需要将kubelet中配置参数

```bash
vim /etc/systemd/system/kubelet.service
#添加选项
--enable-controller-attach-detach=false

#重启服务
systemctl daemon-reload
systemctl restart kubelet
```

# 插件安装

[阿里云官方文档](https://www.alibabacloud.com/help/zh/doc-detail/86785.htm)

## FlexVolume插件

FlexVolume是阿里云开发的插件

有前面的介绍可知，FlexVolume需要每个node都需要安装，并且开启选项
- hostPID: true
- hostNetwork: true

```yaml
apiVersion: apps/v1 
kind: DaemonSet
metadata:
  name: flexvolume
  namespace: kube-system
  labels:
    k8s-volume: flexvolume
spec:
  selector:
    matchLabels:
      name: acs-flexvolume
  template:
    metadata:
      labels:
        name: acs-flexvolume
    spec:
      hostPID: true
      hostNetwork: true
      tolerations:
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      containers:
      - name: acs-flexvolume
        image: registry.cn-hangzhou.aliyuncs.com/acs/flexvolume:v1.11.2.32-af2d48c-aliyun
        imagePullPolicy: Always
        securityContext:
          privileged: true
        env:
        - name: ACS_DISK
          value: "true"
        - name: ACS_NAS
          value: "true"
        - name: ACS_OSS
          value: "true"
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: usrdir
          mountPath: /host/usr/
        - name: etcdir
          mountPath: /host/etc/
        - name: logdir
          mountPath: /var/log/alicloud/
      volumes:
      - name: usrdir
        hostPath:
          path: /usr/
      - name: etcdir
        hostPath:
          path: /etc/
      - name: logdir
        hostPath:
          path: /var/log/alicloud/
```

## Disk Provisioner 安装

Disk Provisioner是storageclass的控制器，通过监听api server来完成创建pv以及和pvc绑定以及创建和删除的操作。

和阿里云sdk对接创建磁盘资源

可以通过ClusterRole中的权限看出来，通过nodeSelector将pod部署在master节点上

出现错误也是查看日志的地方

[仓库地址](https://github.com/kubernetes/cloud-provider-alibaba-cloud)

```yaml
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: alicloud-disk-controller-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["list", "watch", "create", "update", "patch"]
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: alicloud-disk-controller
  namespace: kube-system
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: run-alicloud-disk-controller
subjects:
  - kind: ServiceAccount
    name: alicloud-disk-controller
    namespace: kube-system
roleRef:
  kind: ClusterRole
  name: alicloud-disk-controller-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: alicloud-disk-controller
  namespace: kube-system
spec:
  replicas: 1
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: alicloud-disk-controller
    spec:
      nodeSelector:
         kubernetes.io/role: "master"
      serviceAccount: alicloud-disk-controller
      containers:
        - name: alicloud-disk-controller
          image: registry.cn-hangzhou.aliyuncs.com/acs/alicloud-disk-controller:v1.12.6.16-1f4c6cb-aliyun
          volumeMounts:
            - name: cloud-config
              mountPath: /etc/kubernetes/
            - name: logdir
              mountPath: /var/log/alicloud/
      volumes:
        - name: cloud-config
          hostPath:
            path: /etc/kubernetes/
        - name: logdir
          hostPath:
            path: /var/log/alicloud/
```

## storageclass创建

### 创建

其中注释 `storageclass.kubernetes.io/is-default-class: "true"` 来是否指定为默认的storageclass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
  name: alicloud-disk-beijing-c
parameters:
  regionid: cn-beijing
  type: available
  zoneid: cn-beijing-c
provisioner: alicloud/disk
reclaimPolicy: Retain
volumeBindingMode: Immediate
```

### 参数说明

[详见官方文档](https://www.alibabacloud.com/help/zh/doc-detail/86612.htm?spm=a2c63.p38356.879954.8.519d383dqTDRVo#section-t55-yvs-vdb)

- provisioner：配置为 alicloud/disk，标识StorageClass使用阿里云云盘 provisioner 插件创建。
- type：标识云盘类型，支持 cloud、cloud_efficiency、cloud_ssd、available 四种类型；其中 available 会对高效、SSD、普通云盘依次尝试创建，直到创建成功。
- regionid：期望创建云盘的区域。
- reclaimPolicy: 云盘的回收策略，默认为Delete，支持Retain。如果数据安全性要求高，推荐使用Retain方式以免误删；
- zoneid：期望创建云盘的可用区。cn-hangzhou-a,cn-hangzhou-b,cn-hangzhou-c，可以有多个
- encrypted：（可选）创建的云盘是否加密，默认情况是false，创建的云盘不加密。

### 磁盘类型与大小

普通云盘：最小5Gi
高效云盘：最小20Gi
SSD云盘：最小20Gi

## pvc创建测试

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: alicloud-disk-beijing-c
  resources:
    requests:
      storage: 20Gi
```

## 查看创建

```bash
#发现为pending状态
NAME   STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS              AGE
pvc    Pending                                      alicloud-disk-beijing-c   3s

#查看原因
kubectl describe pvc pvc
  Warning    ProvisioningFailed    13s (x3 over 35s)  alicloud/disk alicloud-disk-controller-67cdd5dbd6-qndq5 d693bc8c-a932-11e9-bf07-be2f53d56dff  Failed to provision volume with StorageClass "alicloud-disk-beijing-c": AccessKeyId cannot be empty!
```
看来是认证没过去，首先是翻边全网也没有找到AccessKeyId在哪里配置，询问客服才知道需要配置相应的ram role

## ram role配置

### 创建role

登陆阿里云控制台-访问控制-角色管理[左侧]-添加角色

服务角色类型-ecs云服务器

角色的授权策略

```json
{
  "Version": "1",
  "Statement": [
    {
      "Action": [
        "ecs:AttachDisk",
        "ecs:DetachDisk",
        "ecs:DescribeDisks",
        "ecs:CreateDisk",
        "ecs:CreateSnapshot",
        "ecs:DeleteDisk",
        "ecs:CreateNetworkInterface",
        "ecs:DescribeNetworkInterfaces",
        "ecs:AttachNetworkInterface",
        "ecs:DetachNetworkInterface",
        "ecs:DeleteNetworkInterface",
        "ecs:DescribeInstanceAttribute",
        "ess:Describe*",
        "ess:CreateScalingRule",
        "ess:ModifyScalingGroup",
        "ess:RemoveInstances",
        "ess:ExecuteScalingRule",
        "ess:ModifyScalingRule",
        "ess:DeleteScalingRule",
        "ecs:DescribeInstanceTypes",
        "ess:DetachInstances"
      ],
      "Resource": [
        "*"
      ],
      "Effect": "Allow"
    },
    {
      "Action": [
        "cr:Get*",
        "cr:List*",
        "cr:PullRepository"
      ],
      "Resource": [
        "*"
      ],
      "Effect": "Allow"
    },
    {
      "Action": [
        "eci:CreateContainerGroup",
        "eci:DeleteContainerGroup",
        "eci:DescribeContainerGroups",
        "eci:DescribeContainerLog"
      ],
      "Resource": [
        "*"
      ],
      "Effect": "Allow"
    },
    {
      "Action": [
        "log:*"
      ],
      "Resource": [
        "*"
      ],
      "Effect": "Allow"
    },
    {
      "Action": [
        "cms:*"
      ],
      "Resource": [
        "*"
      ],
      "Effect": "Allow"
    },
    {
      "Action": [
        "vpc:*"
      ],
      "Resource": [
        "*"
      ],
      "Effect": "Allow"
    }
  ]
}
```

### 分配role给ecs

这里为使用云盘的ecs绑定role

![ecs-ram](https://qiniu.li-rui.top/ecs-ram.jpg)

## 重建pvc

```bash
# d-2zeh41naj8qhdsxcrncr 为云盘ID
NAME   STATUS   VOLUME                   CAPACITY   ACCESS MODES   STORAGECLASS              AGE
pvc    Bound    d-2zeh41naj8qhdsxcrncr   20Gi       RWO            alicloud-disk-beijing-c   8s
```








