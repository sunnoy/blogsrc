---
title: kubectl插件krew
date: 2019-12-01 12:12:28
tags:
- kubernetes
---

# kubectl插件机制

kubectl提供了一个插件机制，是社区拓展kubectl的一个方式。

<!--more-->

# krew

krew就是一个kubectl的插件平台，用来下载管理插件

[地址](https://github.com/kubernetes-sigs/krew)

# 安装

```bash
# linux macos

(
  set -x; cd "$(mktemp -d)" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/download/v0.3.2/krew.{tar.gz,yaml}" &&
  tar zxvf krew.tar.gz &&
  ./krew-"$(uname | tr '[:upper:]' '[:lower:]')_amd64" install \
    --manifest=krew.yaml --archive=krew.tar.gz
)

# 添加环境变量
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
```

# 插件仓库

krew仓库已经搜集到的[插件列表](https://github.com/kubernetes-sigs/krew-index/blob/master/plugins.md)

# 常用命令

官方给出了一些[常用命令](https://github.com/kubernetes-sigs/krew/blob/master/docs/USER_GUIDE.md)

## 搜索插件

```bash
# 首先进行初始化
kubectl krew update

# 然后进行搜索
kubectl krew search

```

# sniff插件使用

sniff 是一个抓包插件，抓包完成后会立即打开Wireshark

[地址](https://github.com/eldadru/ksniff)

确保本地安装了Wireshark

## 安装

```bash
kubectl krew install sniff
```

## 使用

```bash
kubectl >= 1.12:
kubectl sniff <POD_NAME> [-n <NAMESPACE_NAME>] [-c <CONTAINER_NAME>] [-i <INTERFACE_NAME>] [-f <CAPTURE_FILTER>] [-o OUTPUT_FILE] [-l LOCAL_TCPDUMP_FILE] [-r REMOTE_TCPDUMP_FILE]

POD_NAME: Required. the name of the kubernetes pod to start capture it's traffic.
NAMESPACE_NAME: Optional. Namespace name. used to specify the target namespace to operate on.
CONTAINER_NAME: Optional. If omitted, the first container in the pod will be chosen.
INTERFACE_NAME: Optional. Pod Interface to capture from. If omited, all Pod interfaces will be captured.
CAPTURE_FILTER: Optional. specify a specific tcpdump capture filter. If omitted no filter will be used.
OUTPUT_FILE: Optional. if specified, ksniff will redirect tcpdump output to local file instead of wireshark.
LOCAL_TCPDUMP_FILE: Optional. if specified, ksniff will use this path as the local path of the static tcpdump binary.
REMOTE_TCPDUMP_FILE: Optional. if specified, ksniff will use the specified path as the remote path to upload static tcpdump to.
```