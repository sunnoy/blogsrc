---
title: traefik ingress使用
date: 2019-05-30 12:12:28
tags:
- mesh
---

# traefik介绍

![traefik](https://qiniu.li-rui.top/traefik.png)

<!--more-->

# 安装

```bash
helm install --name traefik \
  --set dashboard.enabled=true,dashboard.serviceType=NodePort,serviceType=NodePort,rbac.enabled=true stable/traefik
```

# 使用

## 不同子URL对应不同service

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: cheeses
  annotations:
    kubernetes.io/ingress.class: traefik
    traefik.frontend.rule.type: PathPrefixStrip
spec:
  rules:
  - host: cheeses.minikube
    http:
      paths:
      - path: /stilton
        backend:
          serviceName: stilton
          servicePort: http
      - path: /cheddar
        backend:
          serviceName: cheddar
          servicePort: http
      - path: /wensleydale
        backend:
          serviceName: wensleydale
          servicePort: http
```

## 