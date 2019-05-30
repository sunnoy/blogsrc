---
title: harbor helm以及ingress配置
date: 2019-05-30 12:12:28
tags:
- mesh
---

# harbor介绍

harbor是docker镜像的开源仓库托管

<!--more-->

# 安装

## 安装命令

- externalURL是外部访问的url，后面端口是ingress的http端口
- 这里禁用了tls，因为是在内网中用
- 因为是测试没有使用持久化存储，如果有动态存储就去掉--set persistence.enabled=false者一行

```bash
helm repo add harbor https://helm.goharbor.io
helm install --name harbor harbor/harbor \
--set expose.type=ingress \
--set expose.ingress.hosts.core=ddd.com \
--set expose.ingress.hosts.notary=aaa.com \
--set externalURL=http://ddd.com:21520 \
#--set persistence.enabled=false \
--set expose.tls.enabled=false \
--set harborAdminPassword=admin 
```

## 安装检查

```bash
#pod
harbor-harbor-chartmuseum-78c8d78b-48gxq       1/1     Running   0          3m5s
harbor-harbor-clair-7c795c87c4-fmdv5           1/1     Running   3          3m5s
harbor-harbor-core-7698bd46-7lg98              1/1     Running   1          3m5s
harbor-harbor-database-0                       1/1     Running   0          3m5s
harbor-harbor-jobservice-5f5485bc69-9m9vw      1/1     Running   2          3m5s
harbor-harbor-notary-server-7b56c7f8df-6lhcq   1/1     Running   1          3m5s
harbor-harbor-notary-signer-7d8666f875-f894g   1/1     Running   1          3m5s
harbor-harbor-portal-74dff88588-mmnc4          1/1     Running   0          3m5s
harbor-harbor-redis-0                          1/1     Running   0          3m5s
harbor-harbor-registry-58c97bd88d-kp9tw        2/2     Running   0          3m5s

# pv
pvc-2af446e9-82d1-11e9-8307-525400d86b78   5Gi        RWO            Delete           Bound    default/harbor-harbor-chartmuseum                nfs-client              3m14s
pvc-2af480f2-82d1-11e9-8307-525400d86b78   1Gi        RWO            Delete           Bound    default/harbor-harbor-jobservice                 nfs-client              3m25s
pvc-2af4cbaa-82d1-11e9-8307-525400d86b78   5Gi        RWO            Delete           Bound    default/harbor-harbor-registry                   nfs-client              3m25s
pvc-2b0fa96c-82d1-11e9-8307-525400d86b78   1Gi        RWO            Delete           Bound    default/database-data-harbor-harbor-database-0   nfs-client              3m14s
pvc-2b12fe19-82d1-11e9-8307-525400d86b78   1Gi        RWO            Delete           Bound    default/data-harbor-harbor-redis-0               nfs-client              3m14s
```

# ingress配置

默认的ingress中的注释是这样的

```yaml
    annotations:
      ingress.kubernetes.io/ssl-redirect: "true"
      ingress.kubernetes.io/proxy-body-size: "0"
      nginx.ingress.kubernetes.io/ssl-redirect: "true"
      nginx.ingress.kubernetes.io/proxy-body-size: "0"
```

因为禁用了tls，上面的所以会造成错误，需要删掉两条

```yaml
    annotations:
      ingress.kubernetes.io/proxy-body-size: "0"
      nginx.ingress.kubernetes.io/proxy-body-size: "0"
```

# docker配置

## hosts配置

```bash
10.9.90.161 ddd.com
10.9.90.161 aaa.com
```

## docker进程配置

注意需要加上端口

```json
// 文件 /etc/docker/daemon.json
{
  "insecure-registries" : ["ddd.com:21520"]
}
```

## push镜像

```bash
docker login -uadmin -padmin ddd.com:21520
docker image tag nginx:1.14.2 ddd.com:21520/library/dong:1.0
docker push ddd.com:21520/library/dong:1.0
```