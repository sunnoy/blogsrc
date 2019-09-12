---
title: 集群本地调试工具Telepresence
date: 2019-09-12 12:12:28
tags:
- kubernetes
---

# Telepresence使用场景

当在集群中使用service对象的时候，服务之间连接是通过service name来的，那么在集群外应用如何通过集群内的service name连接到集群内呢？

Telepresence就是为满足这样的场景使用的，主要用于测试环境。Telepresencey已经是cncf的沙箱项目

<!--more-->

# 安装

- [项目地址](https://github.com/telepresenceio/telepresence)

- [文档地址](https://www.telepresence.io/discussion/overview)

mac安装

```bash
brew cask install osxfuse
brew install datawire/blackbird/telepresence
```

# 使用

Telepresence本质上是提供了双向代理

- 在集群内访问一个service name的时候，流量就会转发到本地启动的服务上
- 本地访问远程集群内的service name的时候就会通过远程的dns解析出service IP然后通过代理转发到集群里面

Telepresence通过在集群中创建deployment和service对象的方式来完成双向代理的操作

## 集群连接

既然是在集群内创建deployment等对象，就要可以访问到集群，因此需要认证和授权过程，Telepresence使用的是kubectl的认证配置方式，因此想要使用需要配置kubectl所需要的认证方式。比如kubeconfg文件，其中可以通过下面的选项指定集群和namespace

```bash
telepresence --context mycluster --namespace kube-system
```

## 新创建deployment

通过创建一个deployment在集群中，本地就可以访问集群的服务了

```bash
telepresence --new-deployment myserver --run-shell
#仅仅一条命令 也可以
#该命令会创建名称telepresence开头的deploymnet对象
telepresence
```

## 在集群中创建服务

下面会集群内创建name为myserver的deployment和service，service的端口为80，集群内访问myserver:80就会将流量转发到本地的808端口的服务上

```bash
telepresence --new-deployment myserver --expose 808:80 \
--run python3 -m http.server 808

T: Starting proxy with method 'vpn-tcp', which has the following limitations: All processes are affected, only one telepresence can run per machine, and you can't use other VPNs. You
T: may need to add cloud hosts and headless services with --also-proxy. For a full list of method limitations see https://telepresence.io/reference/methods.html
T: Volumes are rooted at $TELEPRESENCE_ROOT. See https://telepresence.io/howto/volumes.html for details.
T: Starting network proxy to cluster using new Deployment myserver
T: Forwarding remote port 80 to local port 808.

# 集群内的服务
myserver     ClusterIP   10.68.181.94   <none>        80/TCP    53s
```

## 替换集群内的deployment

如果集群中已经存在deployment了，本地需要调试该deployment，此时需要进行交换操作

前提是集群中存在deployment app和service app ，app的port为89，本地存在app的运行实例监听端口在900

```bash
telepresence --swap-deployment app --expose 900:80 \
--run python3 -m http.server 900
```

# 代理模式

telepresence提供了三种代理模式

- inject-tcp
- vpn-tcp
- container

几种方式的机制和限制详见[文档](https://www.telepresence.io/reference/methods)

# 类似工具

```bash
https://github.com/alibaba/kt-connect
```

