---
title: kubebuilder编写控制器之四认识控制器
date: 2019-10-10 12:22:28
tags:
- kubernetes
- node
---

# node资源

node资源不是集群的内建资源，集群中的node对象更像是实际node的一个“代理”。因此心跳机制就显得尤为重要

<!--more-->

# 心跳与租约

从1.4开始引入了Node lease对象，这个对象属于leases.coordination.k8s.io API组，并且在namespace kube-node-lease中。

lease对象是心跳的一个“代理”，为了更好的减少由于node心跳产生的master负载。

lease对象有controller manger维护，status则是有kubelet来维护。他们两个有不通的运行周期

[具体参见](https://kubernetes.io/docs/concepts/architecture/nodes/)

[设计文档](https://github.com/kubernetes/enhancements/blob/master/keps/sig-node/0009-node-heartbeat.md#graduation-criteria)

# kubelet的配置

kubelet后续会采用动态配置，[详细的配置字段](https://github.com/kubernetes/kubernetes/blob/master/staging/src/k8s.io/kubelet/config/v1beta1/types.go)


todo 需要更多说明，字段介绍以及分类和相关机制