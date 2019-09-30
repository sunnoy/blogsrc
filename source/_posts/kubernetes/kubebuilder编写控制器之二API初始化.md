---
title: kubebuilder编写控制器之二API初始化
date: 2019-09-28 12:22:28
tags:
- kubernetes
- crd
---

# k8s中的API

API涉及到四个概念

- groups
- versions
- kinds
- resources

采用API groups的机制是为了方便对集群拓展，[详见](https://kubernetes.io/docs/reference/using-api/#api-versioning)

<!--more-->

## groups和versions

API groups是一系列相关功能函数的集合。随着函数功能的演进引入了API groups version的概念。


## kinds和resources

API groups version会包含多种一个或者多个API type，比如查看app API groups中的Kind就有deployments和daemonsets等

```bash
kubectl api-resources --api-group apps -o wide

NAME                  SHORTNAMES   APIGROUP   NAMESPACED   KIND                 VERBS
controllerrevisions                apps       true         ControllerRevision   [create delete deletecollection get list patch update watch]
daemonsets            ds           apps       true         DaemonSet            [create delete deletecollection get list patch update watch]
deployments           deploy       apps       true         Deployment           [create delete deletecollection get list patch update watch]
replicasets           rs           apps       true         ReplicaSet           [create delete deletecollection get list patch update watch]
statefulsets          sts          apps       true         StatefulSet    
```

当创建某个API groups的某一个Kind的时候，我们就创建了resources，Kind和resource有一对一和一对多的关系，但是在crd中只有一对一的关系。

另外resource总是以Kind的全部小写字母表示。

## 与go联系和Scheme

GVK(GroupVersionKind)就是特定某种API groups的version中的一组资源类型，在go代码package中每个GVK对应一个root type

描述这种对应关系的就是scheme，[详见](https://godoc.org/k8s.io/apimachinery/pkg/runtime#Scheme)

```bash
Scheme defines methods for serializing and deserializing API objects, a type registry for converting group, version, and kind information to and from Go schemas, and mappings between Go schemas of different versions. A scheme is the foundation for a versioned API and versioned configuration over time.

In a Scheme, a Type is a particular Go struct, a Version is a point-in-time identifier for a particular representation of that Type (typically backwards compatible), a Kind is the unique name for that Type within the Version, and a Group identifies a set of Versions, Kinds, and Types that evolve over time. An Unversioned Type is one that is not yet formally bound to a type and is promised to be backwards compatible (effectively a "v1" of a Type that does not expect to break in the future).

Schemes are not expected to change at runtime and are only threadsafe after registration is complete.
```

本文讨论通过构造器建立type到kind的映射关系过程

# 创建api

使用命令创建API groups以及type(Kind)

```bash
kubebuilder create api --group batch --version v1 --kind CronJob

# 该命令会创建一个type.go的文件

api/v1/cronjob_types.go

# 相关帮助
Flags:
      --controller       if set, generate the controller without prompting the user (default true)
      --example          if true an example reconcile body should be written while scaffolding a resource. (default true)
      --group string     resource Group
  -h, --help             help for api
      --kind string      resource Kind
      --make             if true, run make after generating files (default true)
      --namespaced       resource is namespaced (default true)
      --resource         if set, generate the resource without prompting the user (default true)
      --version string   resource Version
```

## cronjob_types.go

### meta/v1引入

首先会引入meta/v1 API group，该API groups通过库https://github.com/kubernetes/apimachinery创建，通常该库用来和api server交互。

meta/v1 被所有的Kind来引用。

```go
//api/v1/cronjob_types.go
package v1

import (
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)
```

### 获取Spec和Status 

通过集群的运行机制我们知道，大部分的对象都有spec和status的概念。控制器的作用就是通过不断观察集群中的对象的状态写入到status中，然后和spec中的内容进行比较，直到调整的一直。这个过程成为调协。

```go
// EDIT THIS FILE!  THIS IS SCAFFOLDING FOR YOU TO OWN!
// NOTE: json tags are required.  Any new fields you add must have json tags for the fields to be serialized.

// CronJobSpec defines the desired state of CronJob
type CronJobSpec struct {
	// INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
	// Important: Run "make" to regenerate code after modifying this file
}

// CronJobStatus defines the observed state of CronJob
type CronJobStatus struct {
	// INSERT ADDITIONAL STATUS FIELD - define observed state of cluster
	// Important: Run "make" to regenerate code after modifying this file
}
```

### 获取CronJob和CronJobList types

- CronJob type用来描述 CronJob Kind，包含两种类型的信息
	- TypeMeta 描述api的version和kind
	- ObjectMeta 描述name, namespace, and label等
- CronJobList 就是包含多个 CronJob 对象

下面代码默认在文件api/v1/cronjob_types.go中

```go
// CronJob is the Schema for the cronjobs API
type CronJob struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   CronJobSpec   `json:"spec,omitempty"`
	Status CronJobStatus `json:"status,omitempty"`
}

// +kubebuilder:object:root=true

// CronJobList contains a list of CronJob
type CronJobList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata,omitempty"`
	Items           []CronJob `json:"items"`
}
```

### kubebuilder注释

```go
// +kubebuilder:object:root=true
```
这个注释是个标记，这个标记说明了下面的type是个资源对象，也就是说type和kind的映射关系。[controller-tools](https://github.com/kubernetes-sigs/controller-tools)会通过这个标记让自己的object生成器去实现一个[runtime.Object](https://godoc.org/k8s.io/apimachinery/pkg/runtime#Object) 一个接口，一个type如果映射一个kind那么他必须要实现这个接口

### Scheme注册

Builder用来实现从type到GroupVersionKinds的映射

这里首先实例化一个构造器

```go
//api/v1/groupversion_info.go

	// SchemeBuilder is used to add go types to the GroupVersionKind scheme
	SchemeBuilder = &scheme.Builder{GroupVersion: GroupVersion}
```

然后调用Register方法对type和kind建立映射关系

```go
func init() {
	SchemeBuilder.Register(&CronJob{}, &CronJobList{})
}
```