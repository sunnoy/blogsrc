---
title: kubernetes-client-go
date: 2019-11-09 12:22:28
tags:
- kubernetes
---

# client-go介绍

client go是kubernetes的go语言客户端库

- 地址 https://github.com/kubernetes/client-go

<!--more-->

# 通过go mod 安装

```bash
export GO111MODULE=on
go mod init
# 需要指定集群的版本
go get k8s.io/client-go@kubernetes-1.15.0
```

# 使用

to do 增加相关测试代码