---
title: kubectl中context使用
date: 2019-08-09 12:12:28
tags:
- kubernetes
---

# 什么是context

我们知道kubectl是官方的集群管理工具，当我们去管理一个开启认证功能的集群的时候一定是要带上相关认证凭据的。

一般来说认证凭据包含下面几个：
- 集群地址
- 集群证书ca
- 集群客户端认证凭据
- namespace
- 集群用户

如果每次管理集群都携带这么多参数未免过于繁琐，于是context出现了。

context就是将上面的认证信息汇总在一块儿的东西

<!--more-->

# 集群客户端认证凭据

集群认证有好多方式，常见有两种客户端证书和token

## token认证

这里为spinnaker工具创建一个具有管理员权限的用户

```bash
#创建一个专属的namespace
kubectl create ns spinnaker

#创建一个sa
kubectl create serviceaccount spinnaker-service-account -n spinnaker

#进行sa的集群角色绑定
kubectl create clusterrolebinding spinnaker-service-account --clusterrole cluster-admin --serviceaccount=spinnaker:spinnaker-service-account

#获取sa的token-secret
TOKEN_SECRET=$(kubectl get serviceaccount -n spinnaker spinnaker-service-account -o jsonpath='{.secrets[0].name}')

#获取token
TOKEN=$(kubectl get secret -n spinnaker $TOKEN_SECRET -o jsonpath='{.data.token}' | base64 --decode)
```

## 客户端认证

客户端认证就是部署集群的时候创建的客户端证书，也就是下面apiserver参数中的ca所签发的证书

```bash
--client-ca-file=/etc/kubernetes/ssl/ca.pem
```

## 加入context

```bash
#添加user，这里是token，还可以是客户端证书
kubectl config set-credentials spinnaker-token-user --token $TOKEN

#添加用户使用客户端证书
kubectl config set-credentials clinet --client-certificate=path/to/certfile --client-key=path/to/keyfile

```
# 集群添加

```bash
#添加集群
kubectl config set-cluster k8s --server=https://1.2.3.4:6443 \
                              # ca证书
                              --certificate-authority=path/to/certificate/authority \
                              --insecure-skip-tls-verify=true
                             
```
# 创建一个context

## 直接创建

其实就是将用户和集群结合起来，然后在`${HOME}/.kube/config`创建一个config文件

```bash
# 格式
kubectl config set-context [NAME | --current] [--cluster=cluster_nickname]
[--user=user_nickname] [--namespace=namespace] [options]

# 示例，namespace可以选择性添加，默认为default
kubectl config set-context k8s --cluster=k8s --user=spinnaker-token-user --namespace=namespace
```

## 更改kubetcl默认namespace

可以对已有的context操作

```bash
kubectl config set-context context-cluster1-admin --namespace=res
```

## 查看当前的config文件

```yaml
#kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://xxxxxxx:6443
  name: cluster1
contexts:
- context:
    cluster: cluster1
    namespace: res
    user: admin
  name: context-cluster1-admin
current-context: context-cluster1-admin
kind: Config
preferences: {}
users:
- name: admin
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
```

# context管理

```bash
#查看当前kubeconfig中的context
kubectl config get-contexts

#指定某一个context为当前在用的context
kubectl config use-context k8s

# 显示当前
kubectl config current-context
```

# config的帮助

```bash
Modify kubeconfig files using subcommands like "kubectl config set current-context my-context"

 The loading order follows these rules:

  1.  If the --kubeconfig flag is set, then only that file is loaded. The flag may only be set once and no merging takes
place.
  2.  If $KUBECONFIG environment variable is set, then it is used as a list of paths (normal path delimiting rules for
your system). These paths are merged. When a value is modified, it is modified in the file that defines the stanza. When
a value is created, it is created in the first file that exists. If no files in the chain exist, then it creates the
last file in the list.
  3.  Otherwise, ${HOME}/.kube/config is used and no merging takes place.

Available Commands:
  current-context Displays the current-context
  delete-cluster  Delete the specified cluster from the kubeconfig
  delete-context  Delete the specified context from the kubeconfig
  get-clusters    Display clusters defined in the kubeconfig
  get-contexts    Describe one or many contexts
  rename-context  Renames a context from the kubeconfig file.
  set             Sets an individual value in a kubeconfig file
  set-cluster     Sets a cluster entry in kubeconfig
  set-context     Sets a context entry in kubeconfig
  set-credentials Sets a user entry in kubeconfig
  unset           Unsets an individual value in a kubeconfig file
  use-context     Sets the current-context in a kubeconfig file
  view            Display merged kubeconfig settings or a specified kubeconfig file

Usage:
  kubectl config SUBCOMMAND [options]
```