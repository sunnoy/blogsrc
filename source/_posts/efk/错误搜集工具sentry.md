---
title: 错误搜集工具sentry
date: 2019-08-14 08:12:28
tags:
- efk
---

# sentry概述

sentry专门用来错误事件，同样sentry也是一个综合性的平台，用于日志的搜集展示和搜索。他的主要特性是 real-time。
是一个轻量级日志+追踪工具，多用于前端

<!--more-->

# 服务端安装

## 安装

需要安装docker 和 docker-compose

```bash
git clone https://github.com/getsentry/onpremise.git
cd onpremise && ./install.sh

#过程中需要创建账户和密码
```

## 获取DSN

访问 http://ip:9000
通过界面获取

# kubernets agent安装

## 项目地址

```bash
https://github.com/getsentry/sentry-kubernetes
```

## 创建sa

```bash
#创建一个sa
kubectl create serviceaccount sentry

#进行sa的集群角色绑定
kubectl create clusterrolebinding sentry --clusterrole cluster-admin --serviceaccount=defalut:sentry

```

## 部署agent

```bash
kubectl run sentry-kubernetes \
  --image getsentry/sentry-kubernetes \
  --serviceaccount=sentry \
  --env="DSN=http://1b05e85f10ff43f791f6608e1d032962:62a663e02e214e4aa4ac5b195bbd4d47@x.x.x.x:9000/1"

```

