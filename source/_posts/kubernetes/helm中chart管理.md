---
title: helm中chart管理
date: 2019-4-11 12:12:28
tags:
- kubernetes
---

# 创建chart骨架

```bash
helm create test

#打包chart
helm package mychart

#检查chart
helm lint mychart

#试运行
helm install --dry-run --debug

# USER-SUPPLIED VALUES --set ss=ss 的变量
helm install --dry-run --debug --set ss=ss

#获取k8s manifest文件
helm get manifest

```
<!--more-->