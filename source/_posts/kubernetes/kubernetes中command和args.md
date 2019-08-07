---
title: kubernetes中command和args
date: 2019-8-7 12:12:28
tags:
- kubernetes
---

# 容器运行时加入命令

在docker中运行容器内进程有两个参数Entrypoint和Cmd

<!--more-->

# kubernetes中加入命令

在k8s中有着下面的对应关系

- Entrypoint <-> command
- cmd <-> args

## 多种写法

```yaml
spec:    
  containers:    
  - name: mycentos  
    image: centos  
    imagePullPolicy: IfNotPresent  
    command: ["/bin/sh","-c","while true;do echo hello;sleep 1;done"]

spec:  
  containers:  
  - name: mycentos
    image: centos
    imagePullPolicy: IfNotPresent
    command: ["/bin/sh"]
    args: ["-c","while true;do echo hello;sleep 1;done"]
```

## 容器运行方式

- command和args都没有，就是容器内的默认的运行命令
- 只有 command 忽略容器内的，执行command的
- 只有 args 容器内的执行命令加上args的参数执行
- command和args都有，忽略容器内的，直接执行command加args的情况