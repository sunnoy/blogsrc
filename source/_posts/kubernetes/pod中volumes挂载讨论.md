---
title: pod中volumes挂载讨论
date: 2019-07-31 12:12:28
tags:
- kubernetes
---

# k8s中卷的设计

pod使用卷是需要首先创建volumes然后在容器中进行引用，整体策略和docker compose是一样的，让存储卷和容器进行解耦。在k8s中解耦是个很常见的策略，比如配置信息也是通过解耦的策略。

<!--more-->

# 首先定义卷

卷的后端有很多中，configmap是一种，也可以使用宿主机的文件

宿主机文件

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    run: test
  name: test
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      run: test
  template:
    metadata:
      labels:
        run: test
    spec:
      volumes:
      - name: test
      #此处提供要绑定的宿主机文件目录
        hostPath:
          path: /etc/

```

# 挂载卷

```yaml
      containers:
      - args:
        - sleep
        - "9999"
        image: busybox
        imagePullPolicy: Always
        name: test
        volumeMounts:
        #需要挂载的卷
        - name: test
        #挂载到容器的目录
          mountPath: /var/spool/yumjj
          #挂载上面定义卷的一个子目录，这里还不会覆盖容器内部的文件
          subPath: yum
```

# subPath

该字段有下面集中需求时需要使用

- 不想挂载volumes的全部内容，只是想挂载其中的一个目录或者文件
- 容器内需要挂载的目录不是空目录，里面本来就有文件，不可以覆盖掉

这里假设`/etc`下面有这样的目录

```bash
#/etc
ls etc

yum
my.cnf
```

需要将文件或者目录挂载到的容器内目录是`/var/spool`，里面有文件`nginx.conf`，里面的文件需要保留

## 只挂载单个目录

```yaml
          #需要在spool下面指定目录，该目录没有会自动创建
          mountPath: /var/spool/yum
          #yum 是volumes内的目录
          subPath: yum
```

上述挂载完成后就是

```bash
ls spool

nginx.conf
yum
```

## 只挂载单个文件

**这种方式configmap不会更新**

```yaml
apiVersion: v1
data:
  game.properties: |
    enemies=aliens
    lives=3
  ui.properties: |
    color.good=purple
    color.bad=yellow

kind: ConfigMap
metadata:
  name: test
  namespace: default

apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - name: nginx
      image: nginx
      volumeMounts:
      - name: config-volume
        mountPath: /etc/nginx/game.properties
        subPath: game.properties
      - name: config-volume
        mountPath: /etc/nginx/ui.properties
        subPath: ui.properties
  volumes:
    - name: config-volume
      configMap:
        name: test
        items:
        - key: game.properties
          path: game.properties
        - key: ui.properties
          path: ui.properties
```




