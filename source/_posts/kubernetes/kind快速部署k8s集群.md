---
title: kind快速部署k8s集群
date: 2019-7-22 12:12:28
tags:
- kubernetes
---

# 介绍

使用[kind](https://kind.sigs.k8s.io/docs/user/quick-start/)工具以容器的方式部署集群

<!--more-->


# 组件安装

## docker安装

```bash
curl -Lo https://raw.githubusercontent.com/sunnoy/k8s-play/master/docker-install.sh && bash docker-install.sh
```

## kind安装

```bash
curl -Lo https://github.com/sunnoy/k8s-play/raw/master/kind-linux-amd64
chmod +x ./kind-darwin-amd64
mv ./kind-darwin-amd64 /usr/local/bin/kind
```

## kubectl安装

```bash
curl -LO https://github.com/sunnoy/k8s-play/raw/master/kubectl
chmod +x ./kubectl
mv ./kubectl /usr/local/bin/kubectl
cd
echo 'source <(kubectl completion bash)' >>~/.bashrc
kubectl completion bash >/etc/bash_completion.d/kubectl
source .bashrc
```

# 创建集群

## 单个node

```bash
kind create cluster --name my-cluster
```

## 多个node

### 创建配置文件

一个master 三个node

control-plane需要是奇数，另外注意集群node数量不要过多，宿主机配置低了就开不出来集群了

```yaml
kind: Cluster
apiVersion: kind.sigs.k8s.io/v1alpha3
kubeadmConfigPatches:
- |
  apiVersion: kubeadm.k8s.io/v1beta2
  kind: ClusterConfiguration
  metadata:
    name: config
  networking:
    serviceSubnet: 10.0.0.0/16
kubeadmConfigPatchesJson6902:
- group: kubeadm.k8s.io
  version: v1beta2
  kind: ClusterConfiguration
  patch: |
    - op: add
      path: /apiServer/certSANs/-
      value: my-hostname
nodes:
- role: control-plane
- role: control-plane
- role: worker
- role: worker
- role: worker
```

### 创建集群

```bash
kind create cluster --config ha.yaml --name ha
```

# 应用部署

## 部署应用

```bash
kubectl run --image=nginx nginx
```

## 访问应用

node容器因此常用的nodeport不可以使用，需要用到port-forward

```bash
#底层使用了socat，需要安装依赖
yum install socat -y

#[宿主机port]:[容器port]
#pod
kubectl port-forward nginx-7bb7cd8db5-cglk8 --address=0.0.0.0 809:80
#deployment
kubectl port-forward deployment/nginx --address=0.0.0.0 809:80
#service
kubectl port-forward service/nginx --address=0.0.0.0 809:80
```

# 删除集群

```bash
kind delete cluster --name my-cluster
```

