---
title: nginx ingress使用
date: 2019-05-30 12:12:28
tags:
- mesh
---

# ingress概览P

ingress本质上还是NodePort类型，只不过众多集群边缘节点的前面还有一个lb以及相应的服务发现工具。

![ingress](https://qiniu.li-rui.top/ingress.png)

<!--more-->

# 主要工具

![ingress统计](https://qiniu.li-rui.top/ingress统计.png)

这些ingress controller使用中都会在ingress对象或者service对象中添加相应的annotations来实现不同的功能。

# nginx ingress

go语言组件控制器，底层是nginx和一些lua控制脚本。其底层还是nginx，因此配置nginx的方式有三种

- 通过configmap进行全局配置
- 通过annotations在ingress对象上配置
- 通过模板/etc/nginx/template配置


# 安装

使用helm安装

```bash
helm install stable/nginx-ingress --name nginx --set rbac.create=true --set controller.stats.enabled=true --set controller.stats.service.type=NodePort --set controller.service.type=NodePort --set defaultBackend.enabled=false
```

# 使用

## 不同url对应serveice

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: simple-fanout-example
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - path: /foo
        backend:
          serviceName: service1
          servicePort: 4200
      - path: /bar
        backend:
          serviceName: service2
          servicePort: 8080
```