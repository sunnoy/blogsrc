---
title: fork官方项目
date: 2020-01-16 12:12:28
tags:
- kubernetes
---

# 基础准备

在github上面fork官方项目 https://github.com/kubernetes/kubernetes

<!--more-->

```bash
mkdir -p $GOPATH/src/k8s.io
cd $GOPATH/src/k8s.io
git clone git@github.com:sunnoy/kubernetes.git
```

# 添加官方项目的分支

```bash
git remote add upstream https://github.com/kubernetes/kubernetes
```

# 同步官方分支

```bash
git fetch upstream
git checkout master
git merge upstream/master
```

