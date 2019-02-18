---
title: kubernetes-系统管理
date: 2018-11-18 12:12:28
tags:
- kubernetes
---

# kubernetes-系统管理

![kuberstrac](https://qiniu.li-rui.top/kuberstrac.png)

## 对象和角色

### 对象

主要介绍集群的组件或者对象管理管理，不包含应用级别的管理

kubernetes中的基本对象有

- pod
- Service
- Volume
- Namespace

为了方便操控，kubernetes对基础对象进行高级封装，拓展出一系列对象

- ReplicaSet
- Deployment
- StatefulSet
- DeamonSet
- Job

<!--more-->

### 角色

角色是对象的活动场所，当我们去定义一个对象的状态，集群角色就会对象操作来达到相应的状态

角色有两类：master和nodes

master角色包含组件

- kube-apiserver 
- etcd
- kube-scheduler
- kube-controller-manager
- cloud-controller-manager

nodes包含组件

- kubelet
- kube-proxy
- container Runtime

除此外还有集群附加组件

- DNS
- webUI
- container resource monitoring
- cluster-level Loging

## 理解对象

对象是kubernetes系统中的实体，这些实体描述了集群状态。对象的操作需要通过kubernetes api，操作api方式包含本地`kubectl`命令访问和远程[client libaries](https://kubernetes.io/docs/reference/using-api/client-libraries/)访问，[这里](https://kubernetes.io/docs/reference/#api-reference)为各个版本的api参考

### 对象字段与描述

每个对象都包含两个字段`spec`和`status`，两者分别描述对象的目标状态(desired state)和当前实际状态(actual state)

对象的描述就是描述出对象的`spec`和一些对象的基本属性信息，kubernetes api仅接受json格式数据，常用的是可读性强的yml格式，kubectl可以将yml转换为json格式输入到api中。

对象的描述必备的字段有

- apiVersion
- kind
- metadata

实例

```yml
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2 # tells deployment to run 2 pods matching the template
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

不同对象的`spec`是不一样的，可以参考[这里](https://kubernetes.io/docs/reference/#api-reference)来获取各个对象的详细描述字段

### kubectl对象管理

对象管理有三种方式

- Imperative commands 直接操作对象，适用于开发环境
- Imperative object configuration 作用单个文件，适用于生产环境
- Declarative object configuration 作用单个目录，适用于生产环境

#### Imperative commands

创建对象主要包含四个子命令

- kubectl run 创建一个deployment对象
- kubectl expose 创建一个service对象
- kubectl autoscale 创建一个autoscale对象
- kubectl create <objecttype> [<subtype>] <instancename> 创建子类型对象比如`kubectl create service nodeport <myservicename>`

更新对象，主要用于直接更新对象的一些属性

- kubectl scal 水平伸缩pods
- kubectl annotate 添加/删除对象注释
- kubectl label 添加/删除对象标签
- kubectl set 对对象的某一方面进行配置
- kubectl edit 直接编辑对象字段参数
- kubectl patch 直接更改对象字段

删除对象

- kubectl delete <type>/<name> 直接删除对象，`kubectl delete deployment/nginx`

查看对象

- kubectl get 查看对象的基本信息
- kubectl describe 查看对象的聚合信息
- kubectl logs 查看运行中的pod的stout和stddrr

链式使用

```bash
#创建对象将配置以yaml格式输出到标准输出
kubectl create service clusterip my-svc --clusterip="None" -o yaml --dry-run | \
#从标准输出来读取配置并加入配置然后以yaml输出到标准输出
kubectl set selector --local -f - 'environment=qa' -o yaml | \
#从标准输出来创建对象
kubectl create -f -

###########详细输出##########
$ kubectl create service clusterip my-svc --clusterip="None" -o yaml --dry-run
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: my-svc
  name: my-svc
spec:
  clusterIP: None
  selector:
    app: my-svc
  type: ClusterIP
status:
  loadBalancer: {}
$ kubectl create service clusterip my-svc --clusterip="None" -o yaml --dry-run | kubectl set selector --local -f - 'environment=qa' -o yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: my-svc
  name: my-svc
spec:
  clusterIP: None
  selector:
    environment: qa
  sessionAffinity: None
  type: ClusterIP
status:
  loadBalancer: {}

```

#### Imperative object configuration

用文件来管理对象

创建对象

- kubectl create -f <filename|url>

更新对象

- kubectl replace -f <filename|url> 老的推到重建

删除对象

- kubectl delete -f <filename|url>

查看对象

- kubectl get -f <filename|url> -o yaml 将配置以yaml格式送到标准输出

从imperative commands迁移

```bash
#直接创建的对象导出到配置文件
kubectl get <kind>/<name> -o yaml --export > <kind>_<name>.yaml
#对直接创建的对象使用配置文件进行更新
kubectl replace -f <kind>_<name>.yaml
```

#### Declarative object configuration

用`apply`来管理对象，`-R`来递归子目录

创建对象

- kubectl apply -f <directory>/
该命令执行完成后会在对象附加annotation，annotation为`kubectl.kubernetes.io/last-applied-configuration: '{...}'`内容就是创建该对象的参数内容

示例

```bash
kubectl apply -f https://k8s.io/examples/application/simple_deployment.yaml

kubectl get -f https://k8s.io/examples/application/simple_deployment.yaml -o yaml

```

更新对象

- kubectl apply -f <directory>/，对于已经存在的对象会清除配置里面没有的字段创建配置里面有的字段

不要混用在第三类和第一类之间群容易失去一些关键字

对象更新过程：
- 将对象中注释`last-applied-configuratio`里面的键值对和 -f 后面的配置文件对比，注释中有的配置文件没有的字段就执行删除操作，使用命令`kubectl scale`接修改的就跳过
- 配置文件有的字段，现在对象中没有的字段或者不一样字段内容的就设置为和配置文件一样
- 重新设置注释`last-applied-configuratio`然后提交到api server

分别对不同的字段的更新和相关的动作[参见](https://kubernetes.io/docs/concepts/overview/object-management-kubectl/declarative-config/#how-apply-calculates-differences-and-merges-changes)

删除对象

- kubectl delete -f <filename> 推荐使用
- kubectl apply -f <directory/> --prune -l 

查看对象

- kubectl get -f <filename|url> -o yaml

#### 迁移到第三类

- 从从第一类迁移

```bash
#导出后移除status字段，apply不支持该字段
 kubectl get <kind>/<name> -o yaml --export > <kind>_<name>.yaml

kubectl replace --save-config -f <kind>_<name>.yaml
```

- 从第二类迁移

```bash
kubectl replace --save-config -f <kind>_<name>.yaml
```
### 对象的属性

#### Names和UIDs

同为标记对象，UIDs是系统为对象创建的，主要用于区别同一个对象的不同历史阶段

#### labels和selectors

标签的存在是为了选择的方便，标签不是唯一的也不是独有的。

标签就是附着在对象上的多数键值对，创建时可以加后期也可以加

选择器的[选择方式](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#label-selectors)

#### Annotations

相对于标签，注释有着更大灵活度。在多种场景都可以用到，比如应用debug以及配置，

和标签的区别就是选择器不可以选择，但是他们都是对象的元数据

```json
"metadata": {
  "labels": {
    "key1" : "value1",
    "key2" : "value2"
  },
  "annotations": {
    "key1" : "value1",
    "key2" : "value2"
  }
}
```

### 对象的选择

一个集群中的对象是很多的，同时不同对象存在着不同的状态，对于特定对象的指定状态选择显得很有必要`Field Selectors`用来选择对象

基本格式

```bash
kubectl get object --field-selector foo.bar=baz
```

比如想要得到状态为`Running`pod

```bash
kubectl get pods --field-selector status.phase=Running
```


- default 创建对象直接进入的namespace
- kube-system kubernetes system创建的
- kube-public 自动创建并预留的，用于对象或资源对外的暴露

## 理解角色

### 节点维护模式

开启维护模式后就不会在该节点上调度pod

```yml
#false为关闭维护模式，true为开启维护模式
kubectl patch node 11.11.11.11 -p "{\"spec\":{\"unschedulable\":false}}"
```


