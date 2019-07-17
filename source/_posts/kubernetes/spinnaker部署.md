---
title: spinnaker部署
date: 2019-07-16 12:12:28
tags:
- kubernetes
---

# halyard

haryard是一个官方的spinnaker部署工具，注意halyard执行的环境需要有代理

<!--more-->

# 部署halyard

## docker部署

```bash
docker run -p 8084:8084 -p 9000:9000 \
    --name halyard --rm \
    -v ~/.hal:/home/spinnaker/.hal \
    -it \
    gcr.azk8s.cn/spinnaker-marketplace/halyard:1.20.1

#进入容器后可以进行bash自动完成配置
source <(hal --print-bash-completion)

#该容器内建kubectl命令，用于对集群进行管理
kubectl version
Client Version: version.Info{Major:"1", Minor:"12", GitVersion:"v1.12.7",
```

# Cloud Providers配置

Cloud Providers是运行应用的载体，里面跑的可以是虚拟机也可以pod容器等


## 开启k8s的插件

```bash
#启用插件，应该会看到没有集群认证的提示
hal config provider kubernetes enable

#- WARNING Provider kubernetes is enabled, but no accounts have been configured.
```

## k8s配置sa

- 这里配置一个具有 clusterrolebinding role类型 cluster-admin 权限的 service account
- 然后配置kubectl命令认证

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

#通token来配置kubectl来认证
kubectl config set-credentials spinnaker-token-user --token $TOKEN

kubectl config set-cluster k8s --server=https://172.24.41.238:6443  --insecure-skip-tls-verify=true

kubectl config set-context k8s --cluster=k8s --user=spinnaker-token-user

kubectl config use-context k8s
```

## hal配置集群

这里只是配置一个账号，实际上这个账号对应那个集群是有kubectl来决定的

### 通过sa

```bash
hal config provider kubernetes account add spinnaker-account \
    --provider-version v2 \
    --service-account 
```

### 通过context

如果挂载了kubeconfig目录的话就可以不用创建集群sa

```bash
#要确定当前有连接集群的配置
kubectl config current-context

#然后才可以使用创建账号
hal config provider kubernetes account add spinnaker-account \
    --provider-version v2 \
    --context $(kubectl config current-context)
```

### 通过configfile

```bash
hal config provider kubernetes account add spinnaker-account \
    --provider-version v2 \
    --kubeconfig-file ~/.kube/config
```

## 指定spinnaker方式

spinnaker部署方式有两种

- distributed 和halyard不在一个地方
- local 和halyard在一个地方

这里我们选择distributed方式

```bash
hal config deploy edit --type distributed --account-name spinnaker-account
```

## features配置

```bash
hal config features edit --artifacts true
```

存在的一些features

```bash
auth=false, fiat=false, chaos=false, entityTags=false, jobs=true, pipelineTemplates=null, artifacts=true, mineCanary=null, appengineContainerImageUrlDeployments=null, infrastructureStages=null, travis=null, wercker=null, managedPipelineTemplatesV2UI=null, gremlin=null
```

# spinnaker存储配置

这里使用s3。工具采用 minon

```bash
#添加s3存储
echo spinnakeradmin | hal config storage s3 edit \
    --endpoint http://sp-minio:9000 \
    --access-key-id spinnakeradmin \
    --secret-access-key --bucket spinnaker \
    --path-style-access true

#指定s3存储
hal config storage edit --type s3
```

# 部署spinnaker

## 连接url(可选)

```bash
# api
hal config security api edit --override-base-url https://spinnaker-api.example.com

# ui
hal config security ui edit --override-base-url https://spinnaker.example.com
```

## 查看可用版本

```bash
hal version list
```

## 指定部署版本

```bash
hal config version edit --version 1.13.5
```

## 开始部署

```bash
hal deploy apply

#查看部署
kubectl get pods -n spinnaker -w
```

# 升级 spinnker

## 获取版本

```bash
hal version list
```

## 指定版本和部署

```bash
hal config version edit --version $VERSION

hal deploy apply

hal deploy connect
```

# 删除 spinnker

```bash
hal deploy clean
```

# helm部署

## 部署命令

```bash
helm install --name sp stable/spinnaker --timeout 600
```

## 会创建一个job对象

该job使用的pod里面有3个脚本，[模板在此](https://sourcegraph.com/github.com/helm/charts/-/blob/stable/spinnaker/templates/configmap/halyard-config.yaml?utm_source=share#L17)

该对象会执行一个脚本 /opt/halyard/scripts/install.sh

### install.sh

```bash
#!/bin/bash

# Wait for the Hal daemon to be ready
export DAEMON_ENDPOINT=http://sp-spinnaker-halyard:8064
export HAL_COMMAND="hal --daemon-endpoint $DAEMON_ENDPOINT"
until $HAL_COMMAND --ready; do sleep 10 ; done

bash -xe /opt/halyard/scripts/config.sh

$HAL_COMMAND deploy apply
```

### config.sh

该脚本主要做一些部署spinnaker要用的配置

```bash
# 确定部署版本
$HAL_COMMAND config version edit --version 1.12.5

# Storage配置
echo spinnakeradmin | $HAL_COMMAND config storage s3 edit \
    --endpoint http://sp-minio:9000 \
    --access-key-id spinnakeradmin \
    --secret-access-key --bucket spinnaker \
    --path-style-access true
$HAL_COMMAND config storage edit --type s3


# Docker Registry 配置
$HAL_COMMAND config provider docker-registry enable

if $HAL_COMMAND config provider docker-registry account get dockerhub; then
  PROVIDER_COMMAND='edit'
else
  PROVIDER_COMMAND='add'
fi

CREDS=""


$HAL_COMMAND config provider docker-registry account $PROVIDER_COMMAND dockerhub --address index.docker.io \
    ${CREDS}  --repositories library/alpine,library/ubuntu,library/centos,library/nginx


# kubernetes配置
$HAL_COMMAND config provider kubernetes enable

if $HAL_COMMAND config provider kubernetes account get default; then
  PROVIDER_COMMAND='edit'
else
  PROVIDER_COMMAND='add'
fi

$HAL_COMMAND config provider kubernetes account $PROVIDER_COMMAND default --docker-registries dockerhub \
            --context default --service-account true \
            #这里忽略一些namespace
             --omit-namespaces=kube-system,kube-public --provider-version v2
$HAL_COMMAND config deploy edit --account-name default --type distributed \
                       --location default
# Use Deck to route to Gate
$HAL_COMMAND config security api edit --no-validate --override-base-url /gate
$HAL_COMMAND config features edit --artifacts true
$HAL_COMMAND config features edit --jobs true
```

