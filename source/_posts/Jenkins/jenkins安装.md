---
title: jenkins安装
date: 2018-11-04 12:12:28
tags:
- jenkins
---

## jenkins安装

### 1. 环境安装

- docker环境

### 2. 镜像拉取

```bash
docker pull jenkins/jenkins
```
<!--more-->

### 3. 启动容器

- 存储持久化

```bash
mkdir /opt/jkins

```

- 启动镜像

在插件安装界面安装少量插件即可，后面进入Jenkins要设置插件源

```bash
docker run -p 8002:8080 -p 50000:50000 -d --restart=always --name jenkins -u root -v /opt/jkins:/var/jenkins_home jenkins/jenkins
```

### 使用国内插件源镜像

镜像地址

```bash
https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json
```
找到插件配置

![插件配置](https://qiniu.li-rui.top/插件配置.png)

配置插件地址

![配置](https://qiniu.li-rui.top/配置.png)

# helm安装

```yaml
master:
  imagePullPolicy: IfNotPresent
  adminPassword: xylink
  serviceType: NodePort
  nodePort: 32000
  slaveKubernetesNamespace: kubernetes-plugin
  javaOpts: -Dhudson.slaves.NodeProvisioner.initialDelay -Dhudson.slaves.NodeProvisioner.MARGIN=50 -Dhudson.slaves.NodeProvisioner.MARGIN0=0.85
  tolerations:
    - key: "kubernetes.io/master"
      operator: "Equal"
      value: "value"
      effect: "NoSchedule"
  nodeSelector:
    kubernetes.io/role: master
  persistence:
    storageClass: nfs-client
    size: 5Gi
```
