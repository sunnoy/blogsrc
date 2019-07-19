---
title: kubernetes-存储介绍
date: 2019-7-18 12:12:28
tags:
- kubernetes
---

# kubernetes存储概览

kubernetes对存储做了一层封装，核心实现下面三个功能

- 对存储资源做生命周期管理
- 对存储资源使用者和提供者做解偶分离
- 对存储资源做存储类型的区分

<!--more-->

相应的kubernetes中关于存储的组件也有一些

- Volume Plugins 为各种存储提供接口
- Volume Manager 运行在node上
- PV/PVC Controller 做存储资源的生命周期管理 运行中master上
- Attach/Detach 做使用者和提供者的解偶 运行在master上

![architecture-v1](https://qiniu.li-rui.top/architecture-v1.jpg)

对与操作attach/detach，kubernetes提供了两种实现方式

- 在master通过服务调用实现，在master上主要是通过监听API Server来获取资源变化，从而触发卷的增删改查
- 在node上通过Volume Manager实现，有pod调度到这个node上就会触发kubelet（具体是kubelet里的pod manager）

## 存储中的六种操作

- provision/delete 创建存储资源
- attach/detach 将存储资源放入集群
- mount/unmount 将存储资源和容器绑定


# kubernetes存储插件类型

- In-tree 相关代码在核心组件里面 比如nfs，aws，gcp等
- Out-of-tree 相关代码和核心组件解偶，这个是趋势 如 csi FlexVolume

# Out-of-tree理解

服务中的一个容器想要使用存储，直接调用一个接口，然后就会自动在云提供商后台创建磁盘，自动挂载到你的机器上，最后自动挂载到容器里面。

## 两类Out-of-tree

- FlexVolume 部署麻烦，结构复杂
- csi 组件化部署，未来趋势。容器屁平台和csi插件接口连接通过grpc进行

无论是FlexVolume还是csi，他们做的主要是对存储资源声明周期的管理


# csi架构

csi具有承上启下的作用

![csi-art](https://qiniu.li-rui.top/csi-art.png)

csi接口部署在集群中，存储服务端是服务提供商(比如阿里云)的接口

## kubernetes中部署架构

![deploy-art](https://qiniu.li-rui.top/deploy-art.png)

- 灰色组件表明 K8S 核心组件
- 深蓝色部分指的是 K8S 与 CSI 的对接组件
- 绿色部分表明青云开发的 CSI 存储插件

### csi组件

csi组件分为controller和node两种服务

- controller服务是csi的管理端，包含provisioner和attacher，用来和api server交互获取一些存储信息，一般StatefulSet部署
- node服务包含driver registrar，用来进行一些node本地的操作

### 调用过程

- 用户需要创建为容器创建存储，比如创建了一个pvc
- 容器平台开始调用csi接口
- csi接口调用提供商sdk创建磁盘
- 提供商sdk将创建磁盘的信息通过csi返回给容器管理平台
- 容器平台将容器调度到某一个node上
- 容器平台调用csi插件将创建的磁盘挂载到运行容器的node上
- 插件对挂载好的磁盘进行格式化
- 插件将格式化好的存储挂载到容器里面

# FlexVolume架构

和csi相差不大，具体体现在attach/detach操作上。csi的driver调用方式为grpc，flexvolume为在node上使用命令行

# 参考文档

```bash
https://kubernetes-csi.github.io/docs/introduction.html
http://newto.me/k8s-storage-architecture/
https://zhuanlan.zhihu.com/p/51757577
https://github.com/container-storage-interface/spec/blob/master/spec.md
https://github.com/kubernetes/community/blob/master/contributors/devel/sig-storage/flexvolume.md
```




