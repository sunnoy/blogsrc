---
title: cobra使用
date: 2020-01-14 12:12:28
tags:
- go
---

# cobra介绍

cobra是一个go的命令行工具库，k8s就是用的这个库

<!--more-->

# 安装

```bash
go get -u github.com/spf13/cobra/cobra
cd $GOPATH/bin
./cobra init --pkg-name cobra ../src/cobra
```

