---
title: helm中chart管理
date: 2019-4-11 12:12:28
tags:
- kubernetes
---

# 创建chart骨架

```bash
helm create test

#打包chart
helm package mychart

#检查chart
helm lint mychart

#试运行
helm install --dry-run --debug

# USER-SUPPLIED VALUES --set ss=ss 的变量
helm install --dry-run --debug --set ss=ss

#获取k8s manifest文件
helm get manifest

```

<!--more-->



## 查看相关文件

```bash
.
├── charts # 该chart的其他的chart
├── Chart.yaml # 该chart的元数据
├── templates # 主要是渲染的模板文件，通过values.yaml中的值来渲染出k8s的yaml文件
│   ├── deployment.yaml
│   ├── _helpers.tpl
│   ├── ingress.yaml
│   ├── NOTES.txt
│   ├── service.yaml
│   └── tests
│       └── test-connection.yaml
└── values.yaml

#还有一些可选的文件
|-- LICENSE #该chart的许可类型
|-- README.md #该chart的使用说明
|-- requirements.yaml #该chart的运行依赖

```

## 语义化版本

版本格式：主版本号.次版本号.修订号，版本号递增规则如下：

- 主版本号：当你做了不兼容的 API 修改，
- 次版本号：当你做了向下兼容的功能性新增，
- 修订号：当你做了向下兼容的问题修正。
先行版本号及版本编译元数据可以加到“主版本号.次版本号.修订号”的后面，作为延伸。

chart中的版本展示休要用到语义化版本

[详见](https://semver.org/lang/zh-CN/)

## chart.yaml

该文件为必须文件，主要包含了一些主要的字段

```yaml
#必填字段
apiVersion: API版本现在为 "v1" 
name: chart的名称
version: chart的当前版本

#可选字段
kubeVersion: 兼容的k8s的版本
description: 对chart的描述
#一些相关联的关键词
keywords:
  - xxx
home: chart的项目地址
#chart的源码地址
sources:
  - 

#维护者信息
maintainers: 
  - name: 
    email: 
    url: 
#模板渲染引擎 默认gotpl
engine: gotpl 
#chart的logourl
icon: 
appVersion: app的版本，就是chart部署应用的版本
deprecated: 该chart是否启用 布尔值。可以通过
tillerVersion: 对tiller服务的版本要求 一般为范围
```

## 依赖管理

可以通过两种方式管理依赖

- 通过文件requirements.yaml来声明依赖
- 通过文件夹charts来自动管理依赖

## requirements.yaml方式

通过定义文件requirements.yaml

```yaml
dependencies:
  - name: apache
    version: 1.2.3
    repository: http://example.com/charts
  - name: mysql
    version: 3.2.1
    repository: http://another.example.com/charts
```
进行依赖更新

```bash
helm dependency update

#相关文件会下载到目录 charts内
```

## 通过charts目录管理依赖

比如WordPress中对Apache有依赖

```bash
wordpress:
  Chart.yaml
  requirements.yaml
  # ...
  charts/
    apache/
      Chart.yaml
      # ...
    mysql/
      Chart.yaml
      # ...
```
# templates 和 values

使用的模板引擎为[go模板](https://golang.org/pkg/text/template/)，以及相关联的函数，比如Sprig library

所有的模板都在目录`templates`内，模板内的变量注入有两种方法

- 通过文件 values.yaml 文件
- 通过命令 helm install --values=myvals.yaml 引入yaml格式的变量文件 回覆盖 values.yaml 里面的值

所有以`_`前缀的文件都不会被渲染成k8s的yaml文件

