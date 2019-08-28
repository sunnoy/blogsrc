---
title: 集群备份工具velero使用
date: 2019-08-27 12:12:28
tags:
- kubernetes
---

# velero介绍

[Velero](https://velero.io/)是一个k8s资源的备份和迁移工具，其最大的特点在于可以备份pvc

<!--more-->

# 和etcd备份对比

直接备份etcd是将集群的全部资源备份起来，那么velero就是对集群内对象级别的备份，除了对集群进行整体备份外，还可以通过对type, namespace,label等进行对象分类备份或者恢复，需要注意的是正在备份过程中创建的对象是不会被进入备份的

# velero架构

velero在集群中创建了很多crd以及相关的控制器，进行备份恢复等操作实质上是对相关crd的操作

所创建的crd

```bash
kubectl -n velero get crds -l component=velero

NAME                                CREATED AT
backups.velero.io                   2019-08-28T03:19:56Z
backupstoragelocations.velero.io    2019-08-28T03:19:56Z
deletebackuprequests.velero.io      2019-08-28T03:19:56Z
downloadrequests.velero.io          2019-08-28T03:19:56Z
podvolumebackups.velero.io          2019-08-28T03:19:56Z
podvolumerestores.velero.io         2019-08-28T03:19:56Z
resticrepositories.velero.io        2019-08-28T03:19:56Z
restores.velero.io                  2019-08-28T03:19:56Z
schedules.velero.io                 2019-08-28T03:19:56Z
serverstatusrequests.velero.io      2019-08-28T03:19:56Z
volumesnapshotlocations.velero.io   2019-08-28T03:19:56Z
```

## 备份过程

![velero](https://qiniu.li-rui.top/velero.png)

- 本地velero(其实是一个http客户端)客户端发送备份指令
- 集群内就会创建一个 Backup 对象
- BackupController 监测 Backup 对象开始备份过程
- BackupController会向API server查询相关数据
- BackupController将查询到的数据备份到远端的对象存储

## 数据一致性

对象存储的数据是唯一的数据源，也就是说集群内的控制器会检查远程的oss存储，发现有备份就会在集群内创建相关crd。如果发现远端存储没有当前集群内的crd所关联的存储数据，那么就会删除当前集群内的crd

# 后端存储

velero有两种关于后端存储的crd

- [BackupStorageLocation](https://velero.io/docs/v1.1.0/api-types/backupstoragelocation/)
- [VolumeSnapshotLocation](https://velero.io/docs/v1.1.0/api-types/volumesnapshotlocation/)

[支持的平台](https://velero.io/docs/v1.1.0/support-matrix/)

## BackupStorageLocation

主要用来集群资源的数据存放位置，也就是集群对象数据，不是pvc的数据

主要支持的后端存储是s3兼容的存储，比如mimo和阿里云oss


```yaml
apiVersion: velero.io/v1
kind: BackupStorageLocation
metadata:
  name: default
  namespace: velero
spec:
# 只有 aws gcp azure
  provider: aws
  #存储主要配置
  objectStorage:
  # bucket的名称
    bucket: myBucket
    # bucket内的
    prefix: backup
# 不同的 provider 不同的配置
  config:
    #bucket地区
    region: us-west-2
    # s3认证信息
    profile: "default"
    # 使用Minio的时候加上，默认为false
    # aws的s3可以支持两种url bucket URL
    # 1 Path style URL： http://s3endpoint/BUCKET
    # 2 Virtual-hosted style URL： http://oss-cn-beijing.s3endpoint 将bucker name放到了host header中
    # 3 阿里云仅仅支持virtual hosted 如果下面写上true阿里云oss会报错403
    s3ForcePathStyle: "false"
    # s3的地址，格式为 http://minio:9000
    s3Url: http://minio:9000

```

使用阿里云的oss

```yaml
apiVersion: velero.io/v1
kind: BackupStorageLocation
metadata:
  labels:
    component: velero
  name: default
  namespace: velero
spec:
  config:
    region: oss-cn-beijing
    s3Url: http://oss-cn-beijing.aliyuncs.com
    s3ForcePathStyle: "false"
  objectStorage:
    bucket: build-jenkins
    prefix: ""
  provider: aws
```



## VolumeSnapshotLocation

主要用来给pv做快照，需要云提供商提供插件，阿里云已经提供了[插件](https://github.com/AliyunContainerService/velero-plugin)，这个需要使用csi等存储机制

也可以使用专门的备份工具[restic](https://velero.io/docs/v1.1.0/restic/)，把pv数据备份到oss中去。安装的时候需要添加选项

```bash
--use-restic
#存储pv使用的是oss也就是BackupStorageLocation 因此不用创建VolumeSnapshotLocation对象
--use-volume-snapshots=false
```

本文使用restic来使用pv备份，现阶段有一些限制

- hostPath不支持备份
- 备份数据标志通过pod来识别
- 单线程操作大量文件比较慢 

## Location创建

```bash
velero backup-location create default \
    --provider aws \
    --bucket velero-backups \
    --config region=us-east-1

velero snapshot-location create ebs-us-east-1 \
    --provider aws \
    --config region=us-east-1
```

# 安装

velero提供了一个命令行和用来初始化服务端和常用备份恢复操作。该命令行和集群交互通过和kubectl的方式是一样的，通过寻找kubeconfig的相关配置来访问集群，主要是 KUBECONFIG环境变量和 ~/.kube/config 文件以及选项 --kubeconfig

本次安装将会使用阿里云的oss和restic来作为后端存储

## 安装准备

- 开通阿里云的oss并获取相关认证信息
- restic需要docker进程开通mount传播，需要在docker启动的systemd文件内加入MountFlags=shared

>MountFlags：服务的 Mount Namespace 配置，会影响进程上下文中挂载点的信息，即服务是否会继承主机上已有挂载点，以及如果服务运行执行了挂载或卸载设备的操作，是否会真实地在主机上产生效果。可选值为 shared、slaved 或 private
>shared：服务与主机共用一个 Mount Namespace，继承主机挂载点，且服务挂载或卸载设备会真实地反映到主机上
>slave：服务使用独立的 Mount Namespace，它会继承主机挂载点，但服务对挂载点的操作只有在自己的 Namespace 内生效，不会反映到主机上
>private：服务使用独立的 Mount Namespace，它在启动时没有任何任何挂载点，服务对挂载点的操作也不会反映到主机上
>[详情参考](https://blog.mallux.me/2017/02/13/systemd/)

## 开始安装

下载最新版[velero](https://github.com/heptio/velero/releases/)，并解压

```bash
#可以做一下自动完成
velero completion bash
```

### 准备credentials-velero

credentials-velero为阿里云oss的认证信息，会在集群中创建密钥

```bash
# default和BackupStorageLocation对象中profile字段的值要对应
[default]
aws_access_key_id = xxx
aws_secret_access_key = xxx
```

### 安装命令

```bash
./velero install \
    --image gcr.azk8s.cn/heptio-images/velero:v1.1.0 \
    --provider aws \
    --bucket xxx \
    --prefix xxx \
    --namespace velero \
    --secret-file ./credentials-velero \
    --velero-pod-cpu-request 200m \
    --velero-pod-mem-request 200Mi \
    --velero-pod-cpu-limit 200m \
    --velero-pod-mem-limit 200Mi \
    --use-volume-snapshots=false \
    --use-restic \
    --restic-pod-cpu-request 200m \
    --restic-pod-mem-request 200Mi \
    --restic-pod-cpu-limit 200m \
    --restic-pod-mem-limit 200Mi \
    --backup-location-config region=oss-cn-beijing,s3ForcePathStyle="false",s3Url=http://oss-cn-beijing.aliyuncs.com
```

# 带有pv的备份

## 加注释

restic使用需要为带有pvc的pod加上注释

```bash
kubectl -n YOUR_POD_NAMESPACE annotate pod/YOUR_POD_NAME backup.velero.io/backup-volumes=YOUR_VOLUME_NAME_1,YOUR_VOLUME_NAME_2,...

# 这里使用elasticsearchpod
kubectl -n elasticsearch annotate pod elasticsearch-master-0 backup.velero.io/backup-volumes=elasticsearch-master

kubectl get pod -n elasticsearch elasticsearch-master-0 -o jsonpath='{.metadata.annotations}'

map[backup.velero.io/backup-volumes:elasticsearch-master]
```

## 创建备份

基本命令

```bash
velero create backup NAME [flags]

# 剔除namespace
--exclude-namespaces stringArray                  namespaces to exclude from the backup
#剔除资源类型
--exclude-resources stringArray                   resources to exclude from the backup, formatted as resource.group, such as storageclasses.storage.k8s.io
#包含 集群 资源类型 
--include-cluster-resources optionalBool[=true]   include cluster-scoped resources in the backup
#包含namespace
--include-namespaces stringArray                  namespaces to include in the backup (use '*' for all namespaces) (default *)
#包含namespace资源类型
--include-resources stringArray                   resources to include in the backup, formatted as resource.group, such as storageclasses.storage.k8s.io (use '*' for all resources)
#给这个备份加上标签
--labels mapStringString                          labels to apply to the backup
-o, --output string                                   Output display format. For create commands, display the object but do not send it to the server. Valid formats are 'table', 'json', and 'yaml'. 'table' is not valid for the install command.
#对指定标签的资源进行备份
-l, --selector labelSelector                          only back up resources matching this label selector (default <none>)
#对pv创建快照
--snapshot-volumes optionalBool[=true]            take snapshots of PersistentVolumes as part of the backup
#指定备份的位置
--storage-location string                         location in which to store the backup
#备份数据多久删掉
--ttl duration                                    how long before the backup can be garbage collected (default 720h0m0s)
#指定快照的位置，也就是哪一个公有云驱动
--volume-snapshot-locations strings               list of locations (at most one per provider) where volume snapshots should be stored
```

创建备份

```bash
velero create backup es --include-namespaces=elasticsearch 
```

**restic会使用Path style，而阿里云会禁止Path style需要使用Virtual-hosted，暂时备份没有办法备份pv到oss**

备份创建后会创建一个crd backups.velero.io

## 创建恢复

命令参考

```bash
velero restore create [RESTORE_NAME] [--from-backup BACKUP_NAME | --from-schedule SCHEDULE_NAME] [flags]

      --exclude-namespaces stringArray                  namespaces to exclude from the restore
      --exclude-resources stringArray                   resources to exclude from the restore, formatted as resource.group, such as storageclasses.storage.k8s.io
      --from-backup string                              backup to restore from
      --from-schedule string                            schedule to restore from
  -h, --help                                            help for create
      --include-cluster-resources optionalBool[=true]   include cluster-scoped resources in the restore
      --include-namespaces stringArray                  namespaces to include in the restore (use '*' for all namespaces) (default *)
      --include-resources stringArray                   resources to include in the restore, formatted as resource.group, such as storageclasses.storage.k8s.io (use '*' for all resources)
      --label-columns stringArray                       a comma-separated list of labels to be displayed as columns
      --labels mapStringString                          labels to apply to the restore
      --namespace-mappings mapStringString              namespace mappings from name in the backup to desired restored name in the form src1:dst1,src2:dst2,...
  -o, --output string                                   Output display format. For create commands, display the object but do not send it to the server. Valid formats are 'table', 'json', and 'yaml'. 'table' is not valid for the install command.
      --restore-volumes optionalBool[=true]             whether to restore volumes from snapshots
  -l, --selector labelSelector                          only restore resources matching this label selector (default <none>)
      --show-labels                                     show labels in the last column
  -w, --wait
```

创建恢复

```bash
velero restore create back --from-backup test
```
这也会创建一个restores.velero.io对象












