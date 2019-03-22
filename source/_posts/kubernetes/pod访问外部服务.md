---
title: pod访问外部服务
date: 2019-03-22 12:12:28
tags:
- kubernetes
---

# 访问外部服务

集群中的pod如何像访问集群内的服务一样使用服务名称访问外部服务呢？

<!--more-->

# 使用endpoint方式

下面创建了一个endpoint对象，然后创建一个服务把endpoint对象暴露出来，注意service和endpoint的name要一样。最后使用ingress对象暴露出服务

```yaml
apiVersion: v1
kind: Service
metadata:
  name: apollo
spec:
  ports:
    - port: 80
---
apiVersion: v1
kind: Endpoints
metadata:
  name: apollo
subsets:
  - addresses:
      - ip: 10.9.1.19
    ports:
      - port: 8070
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: apollo-web-ui
spec:
  rules:
  - host: apollo.test.com
    http:
      paths:
      - path: /
        backend:
          serviceName: apollo
          servicePort: 80

```