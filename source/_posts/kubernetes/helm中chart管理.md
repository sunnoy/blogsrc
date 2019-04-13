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

## 定义变量

### 基础变量

通过文件 values.yaml定义

```yaml
#文件 values.yaml
imageRegistry: "quay.io/deis"
dockerTag: "latest"
pullPolicy: "Always"
storage: "s3"
```

### 变量范围与继承

```yaml
title: "My WordPress Site" # 当前chart
mysql:
  max_connections: 100 # 被当前chart的依赖mysql使用
  password: "secret"

apache:
  port: 8080 # 被当前chart的依赖Apache使用
```

当前chart可以通过`.Values.mysql.password`来访问变量，依赖chart不可以访问`.Values.title`。mysql可以访问`apache.port`


## 预定义变量

- Release 对象
- Chart 对象

## 渲染变量

文件values.yaml中的变量需要通过对象`.Values`来引入

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: deis-database
  namespace: deis
  labels:
    app.kubernetes.io/managed-by: deis
spec:
  replicas: 1
  selector:
    app.kubernetes.io/name: deis-database
  template:
    metadata:
      labels:
        app.kubernetes.io/name: deis-database
    spec:
      serviceAccount: deis-database
      containers:
        - name: deis-database
          image: {{.Values.imageRegistry}}/postgres:{{.Values.dockerTag}}
          imagePullPolicy: {{.Values.pullPolicy}}
          ports:
            - containerPort: 5432
          env:
            - name: DATABASE_STORAGE
              value: {{default "minio" .Values.storage}}
```



# HOOK配置


hook可以在执行chart前后做一些与chart紧密关联的动作，主要是在字段`annotations`中配置

```yaml
apiVersion: ...
kind: ....
metadata:
  annotations:
    "helm.sh/hook": "pre-install"
```

## hook类型

- pre-install: 模板渲染完成后执行，然后创建各种资源
- post-install: 各种资源在集群中创建完成后执行
- pre-delete: 删除集群内资源之前进行的删除操作
- post-delete: 删除集群内资源之后进行的删除操作
- pre-upgrade: 执行升级，创建集群资源之前
- post-upgrade: 执行升级在各种资源升级之后
- pre-rollback: 模板渲染完成后，在进行资源rollback之前
- post-rollback: 所以rollback操作完成后
- crd-install: 在进行crd操作时进行安装crd资源

## hook参与的chart生命周期

假设我们现在需要安装一个带pre-install和post-install的chart

1. 我们安装了一个chart
2. chart被载入到tiller
3. tiller进行一些验证后开始渲染出资源yaml文件

开始进行hook步骤

4. tiller进行pre-install中的资源部署
5. 具有多个hook的化按字母排序进行部署
6. 从排序的底向下部署
7. 除了crd资源，tiller都会等待资源ready

开始进行chart内资源部署

8. 开始进行chart内资源创建

开始进行post-install hook部署

9. tiller进行部署post-install的资源
10. Ttiller等待资源状态成ready
11. Tiller返回release名称
12. helm客户端退出



# 私有仓库认证

## values.yaml中定义

```yaml
imageCredentials:
  registry: quay.io
  username: someone
  password: sillyness
```

## 创建helper

_helpers.tpl

```yaml
{{- define "imagePullSecret" }}
{{- printf "{\"auths\": {\"%s\": {\"auth\": \"%s\"}}}" .Values.imageCredentials.registry (printf "%s:%s" .Values.imageCredentials.username .Values.imageCredentials.password | b64enc) | b64enc }}
{{- end }}
```

## 模板中使用

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: myregistrykey
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: {{ template "imagePullSecret" . }}
```

# 模板编写

## 文件概览

```bash
├── templates # 主要是渲染的模板文件，通过values.yaml中的值来渲染出k8s的yaml文件
│   ├── deployment.yaml
│   ├── _helpers.tpl #在模板渲染中可以复用的小模板段 一般以tpl结尾
│   ├── ingress.yaml
│   ├── NOTES.txt #执行helm install显示的内容
│   ├── service.yaml
│   └── tests
│       └── test-connection.yaml
```
## NOTE.txt

进行安装的显示提示信息，里面也是可以进行相关的变量引入

```yaml
Thank you for installing {{ .Chart.Name }}.

Your release is named {{ .Release.Name }}.

To learn more about the release, try:

  $ helm status {{ .Release.Name }}
  $ helm get {{ .Release.Name }}
```

## 变量的namespace与内建对象

helm认为变量都是在不同的namespace中，namespace和其中的变量使用`.`隔离。比如`.Release.Name` 含义就是：在顶层namespace中找对象`Release`然后获取里面的对象`Name` 。

内建对象一般首字母大写，主要有
- Release 每次的helm install就是一次Release，该对象包含了该次安装的一些元数据
- Values 文件values.yml构成的对象，主要是同文件定义中获取变量定义
- Chart 文件Chart.yaml构成的对象，来获取该文件内的信息
- Files 获取chart中的其他文件
- Capabilities 获取k8s中的元数据，比如API版本以及集群版本等信息
- Template 包含当前执行的模板文件的元数据，比如文件名称以及文件所在路径


### Values Files

通过`Values`可以获取到的变量有四种类型，变量优先级依次提高

- 文件values.yaml
- 依赖chart中的也可以获取到主chart中的变量
- 命令 helm install 和 helm upgrade 中参数 -f 传递的变量文件中的变量
- 通过 --set 参数传递的键值对变量 优先级最高

values.yaml中的层级调用，类似json对象调用

```yaml

favorite:
  drink: coffee
  food: pizza
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink }}
  food: {{ .Values.favorite.food }}
```

### 改变yaml中的字段

通过将相关字段的值设置为`null`

原来为

```yaml
livenessProbe:
  httpGet:
    path: /user/login
    port: http
  initialDelaySeconds: 120
```

先要探针类型改为exec，执行命令

```bash
#关键字段 --set livenessProbe.httpGet=null
helm install stable/drupal --set livenessProbe.exec.command=[cat,docroot/CHANGELOG.txt] --set livenessProbe.httpGet=null
```

结果为

```yaml
livenessProbe:
  exec:
    command:
    - cat
    - docroot/CHANGELOG.txt
  initialDelaySeconds: 120

```

## 模板函数和管道

### 函数

函数的使用格式为

```yaml
{{ 函数名称 参数1 参数2 }}
```

示例

quote 函数会给变量添加双引号

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ quote .Values.favorite.drink }}
  food: {{ quote .Values.favorite.food }}
```

默认函数

```yaml
drink: {{ .Values.favorite.drink | default "tea" | quote }}
```

### 管道

管道和Linux中的管道是一样的，管道符号为`|`。管道左侧为管道右侧函数的参数

示例

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
```

## 流程控制

### 运算符号

运算符号和Linux中shell的运算符号也是一样的

### if/else

基础格式

```yaml
{{ if PIPELINE }}
  # Do something
{{ else if OTHER PIPELINE }}
  # Do something else
{{ else }}
  # Default case
{{ end }}
```
### 使用with来指定对象

看一下示例就明白了

```yaml
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  {{- end }}
  #超范围的要另起一行
  release: {{ .Release.Name }}
```
### range循环

还是看示例

数据源

```yaml
favorite:
  drink: coffee
  food: pizza
pizzaToppings:
  - mushrooms
  - cheese
  - peppers
  - onions
```

使用range迭代

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  {{- end }}

  # 注意下面使用 . 来进行代表数组pizzaToppings的元素
  toppings: |- 
    {{- range .Values.pizzaToppings }}
    - {{ . | title | quote }}
    {{- end }}
```
yaml中的字符块，详见[yaml标准](https://yaml.org/spec/1.2/spec.html)

```bash
|: 保留换行看，末尾空行删除
|-: 删除换行，末尾空行删除
|+: 保留换行，末尾空行保留
```

当然如果迭代的数据源比较少就可以直接写出迭代的数据元素，还是看示例

这里用了数据类型`tuple`

```yaml
  sizes: |-
    {{- range tuple "small" "medium" "large" }}
    - {{ . }}
    {{- end }}
```


## 变量

模板变量中的赋值与引用

```yaml
#赋值
{{- $relname := .Release.Name -}}

#引用
release: {{ $relname }}

#仅使用一个 $ 就回指向 root namespace变量
{{ $.Chart.Name }}-{{ $.Chart.Version }}
```

在with中的使用

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  {{- $relname := .Release.Name -}}
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  release: {{ $relname }}
  {{- end }}
```

在range中的使用，类似python中的字典迭代

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end}}
```

## 子模板

子模板文件名称都是以`_`开始，拓展名称一般为`.tpl`

模板的定义和引用，定义使用关键字`define`，使用则是`tmplate`

```yaml
#定义模板，这个一般放在_helpers.tpl里面，也是可以访问得到的
{{- define "mychart.labels" }}
  labels:
    generator: helm
    date: {{ now | htmlDate }}

    #这里用到了root namesapce中的变量
    chart: {{ .Chart.Name }}
    version: {{ .Chart.Version }}
{{- end }}


apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap

  #使用模板，因为使用了root namespace中的变量 因此需要在
  #使用子模板的时候在后面加上其他的namespace  . 以及 .Vaules 等
  {{- template "mychart.labels" . }}
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
```



### include 函数

include函数主要是以函数的方式引入子模板，上面的template是以动作的方式来引入模板，造成yaml中的缩进问题。通过include的函数引入就可以使用管道然后传递给nindent函数来修正缩进



```yaml
#include包含两个参数，一个是子模板名称，另一个是子模板中的变量范围如：`{{- include "mychart.app" . }}`
#定义变量
{{- define "mychart.app" -}}
app_name: {{ .Chart.Name }}
app_version: "{{ .Chart.Version }}+{{ .Release.Time.Seconds }}"
{{- end -}}

#使用
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  labels:
    # nindent后面的数字 为向又缩进几个字符的空间
    {{- include "mychart.app" . | nindent 4 }}
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
  {{- include "mychart.app" . | nindent 2 }}
```





