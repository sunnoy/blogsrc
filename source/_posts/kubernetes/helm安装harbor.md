---
title: helm安装harbor
date: 2019-11-04 12:12:28
tags:
- kubernetes
---

# harbor介绍

harbor是一个私有镜像源

![屏幕快照 2019-03-19 10.31.01](https://qiniu.li-rui.top/屏幕快照%202019-03-19%2010.31.01.png)

<!--more-->

# harbor安装

现在harbor安装使用docker-compose安装多一些，使用helm安装少些，这里使用helm来安装

## helm安装

这里假设helm已经安装到k8s集群中

## 拉取harbor chart

这里需要拉取特定分支的代码，不要拉取主分支，不稳定，这里选择1.0.0分支

```bash
git clone -b 1.0.0 https://github.com/goharbor/harbor-helm.git
```

# 相关设置

修改文件values.yaml相关字段，设置主要为两部分

- 服务暴露方式
- 持久化存储配置

## 服务暴露方式

可选的有

- ingress
- clusterIP
- nodePort

这里选择的是nodePort

## 数据持久化

数据持久化配置稍微复杂一些

```yaml
  persistentVolumeClaim:
    registry:
      # Use the existing PVC which must be created manually before bound
      existingClaim: ""
      # Specify the "storageClass" used to provision the volume. Or the default
      # StorageClass will be used(the default).
      # Set it to "-" to disable dynamic provisioning
      storageClass: ""
      subPath: ""
      accessMode: ReadWriteOnce
      size: 5Gi
```
首先集群中需要有pv或者storageClass。下面又有几种方式

### 每个服务对应一个pvc

因为pv和pvc是一对一的映射关系，因此需要在集群中建立相应数量的pv，该chart中会自动创建pvc。

持久化配置中，下面字段除storageClass无需配置

```yaml
      existingClaim: ""
      #禁止storageclass
      storageClass: "-"
      subPath: ""
```

这里使用nfs作为后端存储

nfs服务端
```bash
cat /etc/exports
/data/harbor-data    10.9.1.0/24(rw,sync,no_root_squash,no_all_squash)

#为每个pv简历文件夹
for i in {1..5}; do
mkdir nfs${i}
done
```


建立相等数量的pv

```bash
for i in {1..5}; do
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv00${i}
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    path: /data/harbor-data/nfs${i}
    server: 10.9.1.146
EOF
done
```

这里我们使用这个方案

### 多个服务对应一个pvc

也可以在集群中创建一个pvc然后通过字段existingClaim(这里假设为abc）和subPath来为一个pv中为不同服务配置不同的subPth

```yaml
      existingClaim: "abc"
      #禁止storageclass
      storageClass: "-"
      subPath: "abc-1"
```

### 多个服务对应一个动态pv

在集群中创建storageClass(这里假设为abc）然后通过下面示例配置来使用动态pv

```yaml
      existingClaim: ""
      #禁止storageclass
      storageClass: "abc"
      subPath: "abc-1"
```

# 安装harbor

直接通过helm命令即可安装

```bash
cd harbor-helm/
#起个名字为dd
helm install --name dd .

##查看安装状态
helm status dd
```
## 集群状态查看

```bash
#查看pod
[root@dev-k8s-node1 ~]# kubectl get pod
NAME                                       READY   STATUS    RESTARTS   AGE
dd-harbor-adminserver-5b4f7fb97b-c8vxs     1/1     Running   1          78m
dd-harbor-chartmuseum-5cbc8c9684-57vls     1/1     Running   0          78m
dd-harbor-clair-65979f499-v2c9r            1/1     Running   1          78m
dd-harbor-core-567b75f9bf-jw9sj            1/1     Running   0          78m
dd-harbor-database-0                       1/1     Running   0          78m
dd-harbor-jobservice-7d58c6c7bb-ms2nf      1/1     Running   0          78m
dd-harbor-nginx-669759799f-ggv9h           1/1     Running   0          78m
dd-harbor-notary-server-54f8c678d5-trdmf   1/1     Running   0          78m
dd-harbor-notary-signer-5bbf77798b-szpt4   1/1     Running   0          78m
dd-harbor-portal-6dfbd9c596-b7wb8          1/1     Running   0          78m
dd-harbor-redis-0                          1/1     Running   0          78m
dd-harbor-registry-5b547746b-pc9mz         2/2     Running   0          78m


#查看pv
[root@dev-k8s-node1 ~]# kubectl get pv
NAME    CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                        STORAGECLASS   REASON   AGE
pv001   10Gi       RWO            Recycle          Bound    default/database-data-dd-harbor-database-0                           15h
pv002   10Gi       RWO            Recycle          Bound    default/data-dd-harbor-redis-0                                       15h
pv003   10Gi       RWO            Recycle          Bound    default/dd-harbor-chartmuseum                                        15h
pv004   10Gi       RWO            Recycle          Bound    default/dd-harbor-jobservice                                         15h
pv005   10Gi       RWO            Recycle          Bound    default/dd-harbor-registry                                           15h


#查看pvc
[root@dev-k8s-node1 ~]# kubectl get pvc
NAME                                 STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-dd-harbor-redis-0               Bound    pv002    10Gi       RWO                           78m
database-data-dd-harbor-database-0   Bound    pv001    10Gi       RWO                           78m
dd-harbor-chartmuseum                Bound    pv003    10Gi       RWO                           78m
dd-harbor-jobservice                 Bound    pv004    10Gi       RWO                           78m
dd-harbor-registry                   Bound    pv005    10Gi       RWO                           78m


```


# 删除安装

```bash
helm delete --purge dd
```

# 访问harbor

通过nodePort访问，密码为文件values.yaml

```yaml
harborAdminPassword: "Harbor12345"
```

# docker访问

如果没有ssl证书需要添加配置，文件/etc/docker/daemon.json

```bash
{
  "insecure-registries" : ["docker.li-rui.top"]
}

#重启docker

#登录
docker login -u=admin -p=Harbor12345 docker.li-rui.top
#登出
docker logout docker.li-rui.top
```







