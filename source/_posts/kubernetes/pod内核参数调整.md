---
title: pod内核参数调整
date: 2020-01-14 12:12:28
tags:
- kubernetes
---

# 方式

有多中方式，这介绍使用 init 容器调整

<!--more-->

# 示例

```yaml
kubectl create -f- <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-sysctl-init
  namespace: default
spec:
  securityContext:
    privileged: true
  containers:
  - image: nginx
    imagePullPolicy: Always
    name: nginx
    ports:
    - containerPort: 80
      protocol: TCP
  initContainers:
  - image: busybox
    command:
    - sh
    - -c
    - echo 10000 > /proc/sys/net/core/somaxconn
    imagePullPolicy: Always
    name: setsysctl
    securityContext:
      privileged: true
EOF
```