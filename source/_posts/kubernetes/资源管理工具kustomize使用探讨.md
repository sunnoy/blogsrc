---
title: 资源管理工具kustomize使用探讨
date: 2019-09-26 12:12:28
tags:
- kubernetes
---

# 和helm它有啥不同

helm使用的是模板渲染机制来渲染出我们所需要的yaml文件，kustomize使用的overlay的机制，学过ps的同学可以类比图层的概念，就很好理解了。

kustomize分为base和overlay部分，在overlay上对base的增删改通过“patch”的操作来完成

kustomize是官方sig出的，所以和集群本身结合就比较紧密，本身就是个命令行工具，不是helm的c/s架构

<!--more-->

# 安装

[下载地址](https://github.com/kubernetes-sigs/kustomize/releases)

# 基本使用

基本使用官网讲的很详细了，这里不在赘述，[具体可参考](https://github.com/kubernetes-sigs/kustomize)

# 插件平台

kustomize也是一个插件平台，默认有了很多插件，插件的使用分为三种

- 通过kustomization.yaml中的[专用字段](https://github.com/kubernetes-sigs/kustomize/blob/master/docs/fields.md)
- 通过字段generators or transformers来使用

插件字段的具体定义也分两种一种是写在文件里面来调用，一个是直接写进kustomization.yaml里面

# 功能点

下面主要介绍工具的一些功能点，取材下面两个地方

- [doc](https://github.com/kubernetes-sigs/kustomize/tree/master/docs)
- [example](https://github.com/kubernetes-sigs/kustomize/tree/master/examples)

# 远程base接入

以远程的base比如是git仓库来作为base，这样就可以维护一处base让其他的来调用

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namePrefix: pre-
resources:
# git连接，http或者ssh
- ssh://**********/java-base.git
```

# Generator插件

通过文件或者键值对来直接创建configmap或者secret对象

```yaml
configMapGenerator:
- name: my-java-server-props
  files:
  - application.properties
- name: my-java-server-env-vars
  literals:	
  - JAVA_HOME=/opt/java/jdk
  - JAVA_TOOL_OPTIONS=-agentlib:hprof
generators:
- config.yaml

secretGenerator:
- literals:
  - env=pre
  name: tets
  type: Opaque
```

config.yaml里面是configmap或者secret对象

```yaml
# config.yaml
apiVersion: builtin
kind: ConfigMapGenerator
metadata:
  name: mymap
# envs:
# - devops.env
# - uxteam.env
literals:
- FRUIT=apple
- VEGETABLE=carrot
```

## name哈希处理

进行build以后configmap和secret的name会有一个哈希后缀，这个是为了升级方便。工具会自动处理好带有哈希值的对应关系。

比如mymap的configmap生成的是mymap-ajdng，在deployment中引用直接使用myapp就行了

[具体讨论请参考](https://github.com/kubernetes-sigs/kustomize/blob/master/examples/configGeneration.md)

# transformers 插件

transformers插件有很多，具体使用官方文档写的也很详细，[请参考](https://github.com/kubernetes-sigs/kustomize/blob/master/docs/plugins/builtins.md)


# josnpath

josnpath可以进行非常灵活的自定义字段，[具体可以参考](https://github.com/kubernetes-sigs/kustomize/blob/master/examples/jsonpatch.md)

yaml格式的时候添加列表

```yaml
- op: add
  # - 表示后面跟的是列表
  path: /spec/rules/0/http/paths/-
  value:
    path: '/test'
    backend:
      serviceName: my-test
      servicePort: 8081
```

# 所有的patch都在一起

通过inline来将所有的patch都放到kustomization.yaml文件内，[可参考](https://github.com/kubernetes-sigs/kustomize/blob/master/examples/inlinePatch.md)

# 一次patch多个对象

通过patches字段可以进行通过name以及标签选择器对多个对象添加同一个字符串，比如istio的sidecar容器注入

```yaml
patches:
- path: <PatchFile>
  target:
    group: <Group>
    version: <Version>
    kind: <Kind>
    name: <Name>
    namespace: <Namespace>
    labelSelector: <LabelSelector>
    annotationSelector: <AnnotationSelector>
```

官方的示例写的也很好，[可以参考](https://github.com/kubernetes-sigs/kustomize/blob/master/examples/patchMultipleObjects.md)

# 命令行

kustomize命令可以进行一些patch的操作，这个的优势在cicd中优势就体现出来了，比如更改一个image版本

非命令行

```yaml
images:
# 这个name可以不带原始image的tag
- name: nginx
  newName: nginx
  newTag: "99"
```

命令行

```bash
kustomize edit set image nginx=nginx:88
```

## 核心命令

```bash
kustomize edit add
kustomize edit set

# add
Adds an item to the kustomization file.

Usage:
  kustomize edit add [command]

Examples:

	# Adds a secret to the kustomization file
	kustomize edit add secret NAME --from-literal=k=v

	# Adds a configmap to the kustomization file
	kustomize edit add configmap NAME --from-literal=k=v

	# Adds a resource to the kustomization
	kustomize edit add resource <filepath>

	# Adds a patch to the kustomization
	kustomize edit add patch <filepath>

	# Adds one or more base directories to the kustomization
	kustomize edit add base <filepath>
	kustomize edit add base <filepath1>,<filepath2>,<filepath3>

	# Adds one or more commonLabels to the kustomization
	kustomize edit add label {labelKey1:labelValue1},{labelKey2:labelValue2}

	# Adds one or more commonAnnotations to the kustomization
	kustomize edit add annotation {annotationKey1:annotationValue1},{annotationKey2:annotationValue2}


Available Commands:
  annotation  Adds one or more commonAnnotations to kustomization.yaml
  base        Adds one or more bases to the kustomization.yaml in current directory
  configmap   Adds a configmap to the kustomization file.
  label       Adds one or more commonLabels to kustomization.yaml
  patch       Add the name of a file containing a patch to the kustomization file.
  resource    Add the name of a file containing a resource to the kustomization file.
  secret      Adds a secret to the kustomization file.


#set
Sets the value of different fields in kustomization file.

Usage:
  kustomize edit set [command]

Examples:

	# Sets the nameprefix field
	kustomize edit set nameprefix <prefix-value>

	# Sets the namesuffix field
	kustomize edit set namesuffix <suffix-value>


Available Commands:
  image       Sets images and their new names, new tags or digests in the kustomization file
  nameprefix  Sets the value of the namePrefix field in the kustomization file.
  namespace   Sets the value of the namespace field in the kustomization file
  namesuffix  Sets the value of the nameSuffix field in the kustomization file.
```
