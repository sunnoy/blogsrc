---
title: kubernetes集群内服务暴露方式
date: 2018-12-14 12:12:28
tags:
- kubernetes
---

访问kubernetes集群中运行的服务有两类：

- 集群内服务之间进行互相访问
- 集群外访问集群提供的服务

# workload和service

<!--more-->

## workload

workload通常管理pod，pod是集群中的最小调度单元。为了进行复杂的调度和管理，kubernetes将workload细分了许多类型：

- Deployments 常用于部署无状态应用如nginx
- StatefulSets 相对于deployment，常用于部署有状态的应用如Zookeeper
- DaemonSets 在pod中的常驻对象，用于搜集pod日志和检测pod性能
- Jobs 用来完成一次行的工作，如定期统计订单量
- CronJobs 对jobs进行有计划的执行

## service

service则用来将workload(pod)暴露在集群外让外界访问

# pod和service

pod在集群中的生命周期很短暂，随时都有可能被销毁，显然直接访问pod并不是一个好的方式。能不能有个对象对pod进行包装该，然后该对象提供一个固定的地址来保证pod中的应用总是可以被访问到的，从而忽略pod的生命周期？这个对象就是service。

每个service对象绑定一个虚拟IP，这个ip仅在集群内可用，这样只要将pod进行相关的标签分类，然后创建service对象的时候选择相应的pod标签，这样以来，集群内其他pod就可以通过绑定的虚拟IP来访问pod中的应用。

service的这个功能是由noed上Kube-Proxy来完成的，Kube-Proxy完成该功能由多种方式。

- UserSpace 会降低性能，1.2以后废弃
- IPtables 当前默认方式
- IPVS 需要安装ipvsadm、ipset和加载ip_vs内核模块

## IPtables

Kube-Proxy作为控制器来动态操作iptables规则，并不是直接操作Netilter内核模块，此模式下的vip仅仅存在iptables规则里面，没有落在网卡实体上面，因此ping不通

![kuiptables](https://qiniu.li-rui.top/kuiptables.png)

## ipvs

在1.9开始支持，将pod作为realserver

![kkrs](https://qiniu.li-rui.top/kkrs.png)

集群内服务暴露大致可分为通过pod暴露和通过service暴露以及通过ingress暴露

# pod暴露

将pod直接暴露在集群外供外界访问，而不经过service对象。有两种方式：

- hostNetwork
- hostPort
- Port Forward

## hostNetwork

hostNetwork模式是将pod直接使用宿主机网络，pod中使用的端口就是主机上监听的端口，一般网络插件会放进这里pod里面来调整宿主机网络。

示例

```yml
apiVersion: v1
kind: Pod
metadata:
name: test
spec:
hostNetwork: true
```

## hostPort

hostPort将容器内端口和主机端口进行一个绑定，相当于端口映射`主机端口:容器端口`

Ingress Controller一般使用该类型

示例

```yml
apiVersion: v1
kind: Pod
metadata:
  generateName: nginx-ingress-controller-
  labels:
    app: ingress-nginx
    controller-revision-hash: 7c5f6985b8
    pod-template-generation: "1"
  name: nginx-ingress-controller-lgfk4
  namespace: ingress-nginx
  containers:
  - args:
    - /nginx-ingress-controller
    - --default-backend-service=$(POD_NAMESPACE)/default-http-backend
    - --configmap=$(POD_NAMESPACE)/nginx-configuration
    - --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services
    - --udp-services-configmap=$(POD_NAMESPACE)/udp-services
    - --annotations-prefix=nginx.ingress.kubernetes.io
    image: rancher/nginx-ingress-controller:0.16.2-rancher1
    imagePullPolicy: IfNotPresent
    name: nginx-ingress-controller
    ports:
    - containerPort: 80
    #hostport
      hostPort: 80
      name: http
      protocol: TCP
    - containerPort: 443
      hostPort: 443
      name: https
      protocol: TCP
  dnsPolicy: ClusterFirst
  #hostnetwork
  hostNetwork: true
  nodeName: ansible
  priority: 0
  restartPolicy: Always
  schedulerName: default-scheduler
```

hostNetwork和hostPort都是和host相关的，一旦目标pod进行调度就需要重新配置。

## Port Forward

Port Forward是命令`kubectl port-forward`来定义的数据转发类型。这种方式可以在当前host上的某个端口和集群中所有host中的指定pod上的端口建立映射关系。

![for](https://qiniu.li-rui.top/for.png)

数据路径从左到右，从Kubctl监听的本地Port开始，经过ApiServer到达目的host上的Kubelet，最后经Socat到指定pod端口。

需要在每个host上安装端口转发工具Socat。SPDY协议是Google提出的基于传输控制协议(TCP)的应用层协议，通过压缩、多路复用和优先级来缩短加载时间。该协议是一种更加快速的内容传输协议。

执行命令

```bash
kubectl port-forward pod-name local-port:container-port
```

# Service暴露

service暴露的具体实现和相应的ServiceTypes相关，kubernetes目前提供四种类型

- ClusterIP
- NodePort
- LoadBalancer
- ExternalName
- externalIPs

服务中的pod提供服务的端口是targetPort

## ClusterIP

ClusterIP是默认的服务类型，该类型会为每个服务绑定一个vip就是ClusterIP，和ClusterIP绑定端口就是port。ClusterIP只可以在集群内访问。

示例

```yml
apiVersion: v1
kind: Service
metadata:
name: my-nginx
selector:
app: my-nginx
spec:
type: ClusterIP
ports:
- name: http
port: 80
targetPort: 80
protocol: TCP
```

## NodePort

NodePort使用较多，该类型相当于做了一个`ClusterIP:port-Nodeip:NodePort`映射。通过NodePort集群内任何一个node都可以通过`NodeIP:NodePort`来访问暴露的服务，端口默认范围30000-32767。创建后Kube-Proxy会以轮询的方式转发给该Service的每一个Pod。

![nodeport](https://qiniu.li-rui.top/nodeport.png)

示例

```yml
kind: Service
apiVersion: v1
metadata:
name: my-nginx
spec:
type: NodePort
ports:
#ClusterIP的port
- port: 80
#容器的port
targetPort: 80
#不指定随机分配，nodeport
nodePort: 31000
selector:
name: my-nginx
```

## LoadBalancer

LoadBalancer主要和一些公有云上的lb做对接。只提供4层负载，Kubernetes必须运行在特定的云服务上。创建后可以通过公有云上的负载均衡器提供的EXTERNAL-IP来访问服务，该EXTERNAL-IP是个vip，在集群中的node之间漂移

效果

```bash
kubectl get svc my-nginx
NAME       CLUSTER-IP     EXTERNAL-IP     PORT(S)          AGE
my-nginx   22.97.121.42   22.13.242.236   8086:30051/TCP   39s
```

## ExternalName 

ExternalName类似占位符，当集群内去访问某个dns名称时返回一个`CNAME`解析。

示例

```yml
kind: Service
apiVersion: v1
metadata:
  name: my-service
  namespace: prod
spec:
  type: ExternalName
  externalName: my.database.example.com
```
当查询my-service.prod.svc.cluster.local时，集群的 DNS 服务将返回一个值为 my.database.example.com的CNAME记录。 访问这个服务的工作方式与其它的相同，唯一不同的是重定向发生在DNS层，而且不会进行代理或转发。

## externalIPs

externalIPs可以在当前集群中的node上创建相关的规则，这个externalIP仅在iptables规则里面，并不在当前节点的网卡上，用于接受到此IP的流量。

externalIPs可以在nodeport+ipvs直接路由模式中使用。位于后端realsever(node)只需配置externalIPs为vip既可，并不需要配置vip到node网卡上，也不需要禁用arp，因为对于VIP限制在iptables层面就已经完成。相关配置[请参考](https://jishu.io/kubernetes/ipvs-loadbalancer-for-kubernetes/)

# Ingress暴露

Ingress在service之前，可以作为一个集群的总路由入口。

![ingress](https://qiniu.li-rui.top/ingress.png)

在集群中Ingress包含两部分：Ingress对象和Ingress控制器，Ingress控制器用来维护Ingress对象。Ingress控制器有很多实现，包括nginx，F5，HAProxy等。控制器内部包含nginx等应用和Ingress控制器应用。

Ingress作为插件安装到集群上来提供服务，天然具有灵活的服务暴露方式，可以作为nginx来用。本文主要介绍nginx ingress控制器。

## rule和default backend

### rule

定义ingress，重点定于rule：

- host 没有就是所有流量，相当于nginx中使用default域名
- paths 访问路径
- backend 使用哪个service

```yml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  #host: foo.bar.com
  - http:
      paths:
      - path: /testpath
        backend:
          serviceName: test
          servicePort: 80
```

### default backend

default backend就是没有被rule匹配到的访问，比如统一返回404

## Ingress类型

### 单个服务

仅仅指定backend

```yml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test-ingress
spec:
  backend:
    serviceName: testsvc
    servicePort: 80
```

### 单域名

一个域名可以在多个service上使用

效果

```bash
foo.bar.com -> 178.91.123.132 -> / foo    service1:4200
                                 / bar    service2:8080
```
对应yml文件

```yml
apiVersion: extensions/v1beta1
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

### 多域名

效果

```bash
foo.bar.com --|                 |-> foo.bar.com s1:80
              | 178.91.123.132  |
bar.foo.com --|                 |-> bar.foo.com s2:80
```
对应yml文件

```yml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: name-virtual-host-ingress
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - backend:
          serviceName: service1
          servicePort: 80
  - host: bar.foo.com
    http:
      paths:
      - backend:
          serviceName: service2
          servicePort: 80
```

### tls

先创建Secret

```yml
apiVersion: v1
data:
  tls.crt: base64 encoded cert
  tls.key: base64 encoded key
kind: Secret
metadata:
  name: testsecret-tls
  namespace: default
type: Opaque
```

将secret应用到ingress

```yml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: tls-example-ingress
spec:
  tls:
  - hosts:
    - sslexample.foo.com
    secretName: testsecret-tls
  rules:
    - host: sslexample.foo.com
      http:
        paths:
        - path: /
          backend:
            serviceName: service1
            servicePort: 80
```

对于nginx ingress控制器，更多设置请参考[官方文档](https://kubernetes.github.io/ingress-nginx/)















