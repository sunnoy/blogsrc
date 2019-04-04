---
title: apollo配置中心k8s部署
date: 2019-03-28 12:12:28
tags:
- apollo
---

# apollo介绍

Apollo是携程开源的分布式配置中心。那么在微服务时代为什么需要配置中心？请参考[该博文](https://yq.aliyun.com/articles/468274)

![配置中心](https://qiniu.li-rui.top/配置中心.png)

**在k8s中分布式部署中，每一个环境都要有一套admin config 注册中心。webui可以使用一套**

<!--more-->

# 仓库拉取

```bash
git clone https://github.com/ctripcorp/apollo.git
```

# 镜像准备

**在构建的镜像的时候，要把admin、config、portal的dockerfile中的`ENTRYPOINT`行注释掉或者删掉。不然会在k8s集群中出现`read-only file system`报错！！！！**

## apollo-admin

```bash
cd apollo/scripts/apollo-on-kubernetes/apollo-admin-server
docker build -t apollo-admin-server:1.0 .
```

apollo-admin提供配置的修改、发布等功能，对接Apollo Portal管理界面

## config-server

```bash
cd apollo/scripts/apollo-on-kubernetes/apollo-config-server
docker build -t apollo-config-server:1.0 .
```

apollo-config提供配置的读取、推送等功能，对接Apollo客户端，该镜像包含三个服务

- Config Service 
- Eureka 服务注册与发现，让config和admin两个服务进行注册
- Meta Server 用于封装Eureka接口，然后对外提供服务

## portal-server

```bash
cd apollo/scripts/apollo-on-kubernetes/apollo-portal-server
docker build -t apollo-portal-server:1.0 .
```
portal-server提供管理界面

## 总体关系

客户端通过meta Server获取config地址和端口列表来获取配置

管理界面通过meta Server获取admin地址和端口列表来进行更改、推送配置

Apollo会自动在客户端和管理界面服务侧做负载均衡

其中Config Service和Admin Service都是无状态部署，但是meta Server必须要有状态部署，因此需要用到StatefulSet部署和headless service。

![overall-architecture](https://qiniu.li-rui.top/overall-architecture.png)

## alpine-bash

```bash
cd apollo/scripts/apollo-on-kubernetes/alpine-bash-3.8-image
docker build -t alpine-bash-3.8:1.0 .
```

alpine-bash 镜像用来在admin service中的initContainers来检查Apollo meta 服务是否准备好，如果准备好就启动admin service容器，否则就等待

相关执行命令

```yaml
      initContainers:
        - image: alpine-bash:3.8
          name: check-service-apollo-config-server-prod
          command: ['bash', '-c', "curl --connect-timeout 2 --max-time 5 --retry 50 --retry-delay 1 --retry-max-time 120 service-apollo-config-server-prod.sre:8080"]
```

# 数据库准备

## 部署数据库服务

通过docker安装mysql

```yaml
version: '2'

services:
  apollo-db:
    image: mysql:5.7
    container_name: apollo-db
    environment:
      TZ: Asia/Shanghai
      MYSQL_ALLOW_EMPTY_PASSWORD: 'yes'
    ports:
      - "13306:3306"
    volumes:
      - ./data:/var/lib/mysql
```

## 导入数据库文件

这里配置两个环境，因此需要导入两个环境的配置数据库和webui的数据库。因为每一个环境需要对应一个数据库

```bash
#导入
config-db-dev
config-db-prod
portal-db
```

# 安装prod环境

## 准备mysql服务

需要修改字段

- subsets中外部数据库的IP和端口
- service中对应的targetport端口

```yaml
---
kind: Service
apiVersion: v1
metadata:
  namespace: sre
  name: service-mysql-for-apollo-prod-env
  labels:
    app: service-mysql-for-apollo-prod-env
spec:
  ports:
    - protocol: TCP
      port: 3306
      targetPort: 13306
  type: ClusterIP
  sessionAffinity: None

---
kind: Endpoints
apiVersion: v1
metadata:
  namespace: sre
  name: service-mysql-for-apollo-prod-env
subsets:
  - addresses:
      - ip: 10.9.1.146
    ports:
      - protocol: TCP
        port: 13306
```

## 安装config Server + Eureka + Meta Server 三合一服务

该服务是有状态的需要通过StatefulSet来部署

需要修改的字段

- spring.datasource.url中数据库用户名称和密码
- eureka指定的服务发现服务器组，**必须要更改成prod注册服务地址**
- image自己构建镜像的tag
- nodePort端口 可选

```yaml
---
# configmap for apollo-config-server-prod
kind: ConfigMap
apiVersion: v1
metadata:
  namespace: sre
  name: configmap-apollo-config-server-prod
data:
  application-github.properties: |
    spring.datasource.url = jdbc:mysql://service-mysql-for-apollo-prod-env.sre:3306/ProdApolloConfigDB?characterEncoding=utf8
    spring.datasource.username = root
    spring.datasource.password = 
    eureka.service.url = http://statefulset-apollo-config-server-prod-0.service-apollo-meta-server-prod:8080/eureka/,http://statefulset-apollo-config-server-prod-1.service-apollo-meta-server-prod:8080/eureka/,http://statefulset-apollo-config-server-prod-2.service-apollo-meta-server-prod:8080/eureka/
---
kind: Service
apiVersion: v1
metadata:
  namespace: sre
  name: service-apollo-meta-server-prod
  labels:
    app: service-apollo-meta-server-prod
spec:
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
  selector:
    app: pod-apollo-config-server-prod
  type: ClusterIP
  clusterIP: None
  sessionAffinity: ClientIP

---
kind: Service
apiVersion: v1
metadata:
  namespace: sre
  name: service-apollo-config-server-prod
  labels:
    app: service-apollo-config-server-prod
spec:
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
      nodePort: 30005
  selector:
    app: pod-apollo-config-server-prod
  type: NodePort
  sessionAffinity: ClientIP

---
kind: StatefulSet
apiVersion: apps/v1beta2
metadata:
  namespace: sre
  name: statefulset-apollo-config-server-prod
  labels:
    app: statefulset-apollo-config-server-prod
spec:
  serviceName: service-apollo-meta-server-prod
  replicas: 3
  selector:
    matchLabels:
      app: pod-apollo-config-server-prod
  updateStrategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: pod-apollo-config-server-prod
    spec:
      nodeSelector:
        node: "apollo"
      
      volumes:
        - name: volume-configmap-apollo-config-server-prod
          configMap:
            name: configmap-apollo-config-server-prod
            items:
              - key: application-github.properties
                path: application-github.properties
      
      containers:
        - image: harbor.best-link.com/apollo/apollo-config:1.0
          securityContext:
            privileged: true
          imagePullPolicy: IfNotPresent
          name: container-apollo-config-server-prod
          ports:
            - protocol: TCP
              containerPort: 8080

          volumeMounts:
            - name: volume-configmap-apollo-config-server-prod
              mountPath: /apollo-config-server/config/application-github.properties
              subPath: application-github.properties
          env:
            - name: APOLLO_CONFIG_SERVICE_NAME
              value: "service-apollo-config-server-prod.sre"
          
          readinessProbe:
            tcpSocket:
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
          
          livenessProbe:
            tcpSocket:
              port: 8080
            initialDelaySeconds: 120
            periodSeconds: 10
          
      dnsPolicy: ClusterFirst
      restartPolicy: Always
```

# 安装admin service

该服务没有状态，通过正常的deployment来部署

需要修改字段

- spring.datasource.url中数据库用户和密码
- image中自己构建的镜像标签

```yaml
---
# configmap for apollo-admin-server-prod
kind: ConfigMap
apiVersion: v1
metadata:
  namespace: sre
  name: configmap-apollo-admin-server-prod
data:
  application-github.properties: |
    spring.datasource.url = jdbc:mysql://service-mysql-for-apollo-prod-env.sre:3306/ProdApolloConfigDB?characterEncoding=utf8
    spring.datasource.username = root
    spring.datasource.password = 
    eureka.service.url = http://statefulset-apollo-config-server-prod-0.service-apollo-meta-server-prod:8080/eureka/,http://statefulset-apollo-config-server-prod-1.service-apollo-meta-server-prod:8080/eureka/,http://statefulset-apollo-config-server-prod-2.service-apollo-meta-server-prod:8080/eureka/
---
kind: Service
apiVersion: v1
metadata:
  namespace: sre
  name: service-apollo-admin-server-prod
  labels:
    app: service-apollo-admin-server-prod
spec:
  ports:
    - protocol: TCP
      port: 8090
      targetPort: 8090
  selector:
    app: pod-apollo-admin-server-prod  
  type: ClusterIP
  sessionAffinity: ClientIP

---
kind: Deployment
apiVersion: apps/v1beta2
metadata:
  namespace: sre
  name: deployment-apollo-admin-server-prod
  labels:
    app: deployment-apollo-admin-server-prod
spec:
  replicas: 3
  selector:
    matchLabels:
      app: pod-apollo-admin-server-prod
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: pod-apollo-admin-server-prod
    spec:
      nodeSelector:
        node: "apollo"
      
      volumes:
        - name: volume-configmap-apollo-admin-server-prod
          configMap:
            name: configmap-apollo-admin-server-prod
            items:
              - key: application-github.properties
                path: application-github.properties
      
      initContainers:
        - image: alpine-bash:3.8
          name: check-service-apollo-config-server-prod
          command: ['bash', '-c', "curl --connect-timeout 2 --max-time 5 --retry 50 --retry-delay 1 --retry-max-time 120 service-apollo-config-server-prod.sre:8080"]
      
      containers:
        - image: apollo-admin-server:v1.0.0
          securityContext:
            privileged: true
          imagePullPolicy: IfNotPresent
          name: container-apollo-admin-server-prod
          ports:
            - protocol: TCP
              containerPort: 8090
          
          volumeMounts:
            - name: volume-configmap-apollo-admin-server-prod
              mountPath: /apollo-admin-server/config/application-github.properties
              subPath: application-github.properties
          
          env:
            - name: APOLLO_ADMIN_SERVICE_NAME
              value: "service-apollo-admin-server-prod.sre"
          
          readinessProbe:
            tcpSocket:
              port: 8090
            initialDelaySeconds: 10
            periodSeconds: 5
          
          livenessProbe:
            tcpSocket:
              port: 8090
            initialDelaySeconds: 120
            periodSeconds: 10

      dnsPolicy: ClusterFirst
      restartPolicy: Always
```

然后，对dev环境进行响应的替换

# 集群内部署

## 对私有仓库做不安全配置

```bash
vi /etc/docker/daemon.json
```
添加私有源的链接

```json
{
  "registry-mirrors": ["https://registry.docker-cn.com", "https://docker.mirrors.ustc.edu.cn"],
  "insecure-registries": ["harbor.best-xylink.com"],
}
```

重启docker服务

```bash
systemctl restart docker
```

## 给响应node添加标签

服务admin、config、portal均需要标签

```bash
kubectl label nodes ${node-ip} --overwirte node=dev
```

## 创建namespace

```bash
kubectl create namespace sre
```

## 准备相关yaml文件

```bash
#查看文件

tree apollo
├── dev
│   ├── service-apollo-admin-server-dev.yaml
│   ├── service-apollo-config-server-dev.yaml
│   └── service-mysql-for-apollo-dev-env.yaml
├── prod
│   ├── service-apollo-admin-server-prod.yaml
│   ├── service-apollo-config-server-prod.yaml
│   └── service-mysql-for-apollo-prod-env.yaml
└── service-apollo-portal-server.yaml
```

## 执行部署

```bash
#首先部署dev环境，因为其他的服务要使用他的服务注册中心
kubectl apply -f service-mysql-for-apollo-dev-env.yaml --record && \
kubectl apply -f service-apollo-config-server-dev.yaml --record && \
kubectl apply -f service-apollo-admin-server-dev.yaml --record

#然后部署prod环境
kubectl apply -f service-mysql-for-apollo-prod-env.yaml --record && \
kubectl apply -f service-apollo-config-server-prod.yaml --record && \
kubectl apply -f service-apollo-admin-server-prod.yaml --record

# 最后部署 portal管理界面
kubectl apply -f service-apollo-portal-server.yaml --record


```

## 删除部署

```bash
kubectl delete -f service-mysql-for-apollo-dev-env.yaml
```

# 查看部署

## 查看pod

```bash
[root@dev-k8s-node1 apollo]# kubectl get pod -n sre 
NAME                                                   READY   STATUS    RESTARTS   AGE
deployment-apollo-admin-server-dev-6f54ccd796-5x7fz    1/1     Running   0          35m
deployment-apollo-admin-server-dev-6f54ccd796-mncbh    1/1     Running   0          35m
deployment-apollo-admin-server-dev-6f54ccd796-rwjr6    1/1     Running   0          35m
deployment-apollo-admin-server-prod-756bdb4bcc-5hphc   1/1     Running   0          72m
deployment-apollo-admin-server-prod-756bdb4bcc-dhd45   1/1     Running   0          72m
deployment-apollo-admin-server-prod-756bdb4bcc-fkpqn   1/1     Running   0          72m
deployment-apollo-portal-server-868f74d454-fcdmm       1/1     Running   0          33m
deployment-apollo-portal-server-868f74d454-l72kg       1/1     Running   0          42m
deployment-apollo-portal-server-868f74d454-mpmh8       1/1     Running   0          42m
statefulset-apollo-config-server-dev-0                 1/1     Running   0          83m
statefulset-apollo-config-server-dev-1                 1/1     Running   0          82m
statefulset-apollo-config-server-dev-2                 1/1     Running   0          82m
statefulset-apollo-config-server-prod-0                1/1     Running   0          72m
statefulset-apollo-config-server-prod-1                1/1     Running   0          72m
statefulset-apollo-config-server-prod-2                1/1     Running   0          71m

```

## 查看svc

```bash
[root@dev-k8s-node1 apollo]# kubectl get svc -n sre 
NAME                                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
service-apollo-admin-server-dev     ClusterIP   10.68.101.110   <none>        8090/TCP         36m
service-apollo-admin-server-prod    ClusterIP   10.68.234.143   <none>        8090/TCP         73m
service-apollo-config-server-dev    NodePort    10.68.185.10    <none>        8080:30002/TCP   84m
service-apollo-config-server-prod   NodePort    10.68.229.144   <none>        8080:30005/TCP   73m
service-apollo-meta-server-dev      ClusterIP   None            <none>        8080/TCP         84m
service-apollo-meta-server-prod     ClusterIP   None            <none>        8080/TCP         73m
service-apollo-portal-server        NodePort    10.68.67.33     <none>        8070:30001/TCP   66m
service-mysql-for-apollo-dev-env    ClusterIP   10.68.17.239    <none>        3306/TCP         104m
service-mysql-for-apollo-prod-env   ClusterIP   10.68.221.124   <none>        3306/TCP         73m
service-mysql-for-portal-server     ClusterIP   10.68.40.79     <none>        3306/TCP         66m

```










