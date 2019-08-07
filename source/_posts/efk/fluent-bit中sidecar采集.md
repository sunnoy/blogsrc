---
title: fluent-bit中sidecar采集
date: 2019-8-7 12:12:28
tags:
- efk
---

# 为什么使用fluent-bti采集

fluent-bit占用资源比较少，仅仅做采集和转发工作

本文使用fluent-bit采集，传递pod所在的node上的fluentd聚合

<!--more-->

# 资源文件

## fluentd配置

### 使用转发插件

```xml
<source>
    @type forward
    port 1989
    bind 0.0.0.0
</source>
```

### 开启nodeport

```yaml
        ports:
        - containerPort: 1989
          hostPort: 1989
          protocol: TCP
```

## fluent-bit

```yaml
apiVersion: v1
data:
  fluent-bit.conf: |
    @INCLUDE ex-*.conf
  ex-input.conf: |
    [INPUT]
        Name        tail
        Path        /data/*.log
        Path_Key    liveddd
        Refresh_Interval 3
        DB /data/aaa.db
        #Multiline   on
        #Parser_Firstline docker 
  ex-parser.conf: |
    [PARSER]
        Name        docker
        Format      json
        Time_Key    time
        Time_Format %Y-%m-%dT%H:%M:%S %z
  ex-output.conf: |
    [OUTPUT]
        Name          forward
        Match         *
        Host          ${NODE_IP}
        Port          1989   
kind: ConfigMap
metadata:
  name: fluentd-config
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    name: dev-tomcatapp
  name: log
spec:
  replicas: 1
  selector:
    matchLabels:
      app: log
  template:
    metadata:
      labels:
        app: log
    spec:
      containers:
      - image: nginx
        imagePullPolicy: IfNotPresent
        name: nginx
        ports:
        - containerPort: 80
          protocol: TCP
        volumeMounts:
        - mountPath: /opt
          name: logs
      - name: fluentd
        image: fluent/fluent-bit
        env:
        - name: NODE_IP
          valueFrom:
            fieldRef:
              fieldPath: status.hostIP
        volumeMounts:
        - name: logs
          mountPath: /opt
        - name: config-volume
          mountPath: /fluent-bit/etc
      volumes:
      - name: logs
        emptyDir: {}
      - name: config-volume
        configMap:
          name: fluentd-config
      restartPolicy: Always
```