---
title: kubernetes-ingress路由
date: 2018-11-04 12:12:28
tags:
- kubernetes
---

# ingress介绍

ingress可以理解为nginx反向代理集群中的服务。ingress反向代理集群中的服务有多中方式，这里使用nodePort方式，外部客户端通过nodeport方式去向ingress发送http请求，该http请求中携带请求头主机信息。ingress接收到请求信息后通过内置的规则去找相应的集群service

![ingress暴露服务以及高可用](https://qiniu.li-rui.top/ingress暴露服务以及高可用.jpg)

<!--more-->

# 安装

```yaml
#创建clusterrole
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: traefik-ingress-controller
rules:
  - apiGroups:
      - ""
    resources:
      - pods
      - services
      - endpoints
      - secrets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch

#将创建的clusterrole和ServiceAccount进行绑定
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: traefik-ingress-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: traefik-ingress-controller
subjects:
- kind: ServiceAccount
  name: traefik-ingress-controller
  namespace: kube-system

#创建ingress使用的ServiceAccount
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: traefik-ingress-controller
  namespace: kube-system

#已deployment的方式进行部署
---
kind: Deployment
apiVersion: apps/v1beta1
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
  labels:
    k8s-app: traefik-ingress-lb
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: traefik-ingress-lb
  template:
    metadata:
      labels:
        k8s-app: traefik-ingress-lb
        name: traefik-ingress-lb
    spec:
      serviceAccountName: traefik-ingress-controller
      terminationGracePeriodSeconds: 60
      containers:
      - image: traefik:v1.7.4
        imagePullPolicy: IfNotPresent
        name: traefik-ingress-lb
        args:
        - --api
        - --kubernetes
        - --logLevel=INFO
#创建ingress服务，方式为nodeport
---
kind: Service
apiVersion: v1
metadata:
  name: traefik-ingress-service
  namespace: kube-system
spec:
  selector:
    k8s-app: traefik-ingress-lb
  ports:
    - protocol: TCP
      # 该端口为 traefik ingress-controller的服务端口
      port: 80
      # 集群hosts文件中设置的 NODE_PORT_RANGE 作为 NodePort的可用范围
      # 从默认20000~40000之间选一个可用端口，让ingress-controller暴露给外部的访问
      nodePort: 23456
      name: web
    - protocol: TCP
      # 该端口为 traefik 的管理WEB界面
      port: 8080
      name: admin
  type: NodePort
```

# 使用

比如暴露某一项服务

```yaml
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

# 后续ssl

