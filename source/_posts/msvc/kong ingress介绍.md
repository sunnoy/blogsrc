---
title: kong ingress介绍
date: 2019-05-31 12:12:28
tags:
- mesh
---

# kong ingress

![dbless-deployment](https://qiniu.li-rui.top/dbless-deployment.png)

<!--more-->

# 没有数据库安装

```bash
helm install ms/kong --name kong \
--set ingressController.enabled=true \
--set postgresql.enabled=false \
--set env.database=off \
--set admin.nodePort=33101 \
--set proxy.http.nodePort=21520 \
--set proxy.tls.nodePort=21521 
```

# Kong CRD

kong通过创建一些CRD来实现他的一些抽象封装，仅仅通过几种crd资源的创建就可以进行使用kong，而不必使用kong admin接口来配置kong

## KongPlugin
用来选择和配置插件

## KongIngress

对ingress或者service对象的增强

## KongConsumer

kong中的消费者

## KongCredential

kong中关于认证的凭据

# 注释

kong ingress依靠一些注释来完成kong组件和kong ingress以及困ernetes中的资源进行结合

## kubernetes.io/ingress.class

在ingress对象中指定使用的ingress controller

```yaml
metadata:
  name: foo
  annotations:
    kubernetes.io/ingress.class: "kong"
```

## plugins.konghq.com

plugins.konghq.com 注释可以用于ingress以及service资源上面

- 用在service对象上面，就是和kong中的service相对应
- 用在ingress对象上面，就代表该ingress对象使用的插件

插件通过kong CRD kongplugins.configuration.konghq.com 来定义

## configuration.konghq.com

configuration.konghq.com 主要指定kong CRD kongingresses.configuration.konghq.com作用的ingress或者service对象上面

不管在ingress或者service对象上使用，都应该使用一个configuration.konghq.com注释

# 插件使用

## 创建插件资源

```yaml
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: url-rw
plugin: request-transformer
```

## 使用插件

### 在ingress

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
   name: myapp-ingress
   annotations:
      plugins.konghq.com: url-rw
```

### 在service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
  labels:
     app: myapp-service
  annotations:
     plugins.konghq.com: url-rw

```

## 检查插件使用状态

通过kong admin API来看

```bash
curl https://admin:port/plugins
```

# KongIngress使用

KongIngress用来增强Ingres对象的功能，主要在Ingres对象配置的基础上进行增强，主要是使用kong内部的资源对象upstream, proxy or route的强化配置

## 创建

```yaml
apiVersion: configuration.konghq.com/v1
kind: KongIngress
metadata:
  name: configuration-demo
route:
  methods:
  - POST
  - PUT
  - DELETE
  - GET
  strip_path: false
  preserve_host: true
  protocols:
  - http
  - https
```

## 使用kongingress

可以在ingress或者service对象上使用，通过注释的方式

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    configuration.konghq.com: configuration-demo
```



