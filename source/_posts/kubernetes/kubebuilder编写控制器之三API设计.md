---
title: kubebuilder编写控制器之三API设计
date: 2019-09-28 12:12:28
tags:
- kubernetes
- crd
---

# API设计原则

kubernetes的API中资源对象的信息是通过json来处理的。因此就涉及到序列化和反序列化

- 从对象信息到json就是序列化
- 反之为反序列化

另外在序列化的过程中会通过json的tag来控制序列化的过程，比如空值处理

<!--more-->

# spec字段设计

# 未完待续








