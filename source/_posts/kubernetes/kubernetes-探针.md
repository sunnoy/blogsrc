---
title: kubernetes-探针
date: 2019-7-30 12:12:28
tags:
- kubernetes
---

# 什么是探针

kubernetes中对应用进行健康检查的一种手段，为了调用该服务的其他服务使用的

<!--more-->

# 探针类型

- Liveness 是否在运行，防止存在死锁，一旦发生死锁就会重启重启
- Readiness 是否可以接受请求访问，一旦发生无法访问将从service的后端endpoint列表中剔除，pod为notready

# 公用参数

- initialDelaySeconds: 第一次检查的延时 单位秒
- periodSeconds: 检查间隔，默认10s。最小1s
- timeoutSeconds: 单次检查超时，默认1s。最小1s
- successThreshold: 成功次数默认1
- failureThreshold: 失败次数就重启容器或者pod notready，默认为3，最小1

# 检查方式

## exec

```yaml
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
```

## http

```yaml
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
        httpHeaders:
        - name: Custom-Header
          value: Awesome
```

## tcp

```yaml
    readinessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 10
    livenessProbe:
      tcpSocket:
        port: 8080
```

## 端口可以使用name

```yaml
ports:
- name: liveness-port
  containerPort: 8080
  hostPort: 8080

livenessProbe:
  httpGet:
    path: /healthz
    port: liveness-port
```

