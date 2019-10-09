---
title: kubebuilder编写控制器之四认识控制器
date: 2019-10-08 12:22:28
tags:
- kubernetes
- crd
---

# 控制器的作用

控制器本质上是一个调协器，他会持续监测集群中目标对象的状态，然后去操作对象使之与期望的对象状态相同

<!--more-->

# 控制器的工作模式

对于k8s的事件，一般都是针对某些资源的一系列的行为（action，比如增删改等行为）以及一组key（通常是 命名空间/名字 namespace/name 的形式）

如同上面控制器模式的描述，控制器会订阅一个队列。

当我们执行类似 kubectl apply 等命令时，会触发对应的api，进而更新缓存（cache）中的内容。

而在此时，我们自定的服务中会定义一个通知器（informer），它会监控缓存（cache）中的内容，并将所需的事件扔到一个我们定义的队列中。

我们定义的控制器（controller）则会订阅这个队列，获取相应的事件。

![controller](https://qiniu.li-rui.top/controller.png)

# 一个空的控制器

刚刚生成的controllers/cronjob_controller.go

## 初始引入

首先肯定是初始的必须引入

```go
import (
	"context"

	"github.com/go-logr/logr"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"

	batchv1 "CronJob/api/v1"
)
```

## 基础的Reconciler功能

要对对象进行管理一定是要连接apiserver的，然后还需要一些日志的记录

```go
// CronJobReconciler reconciles a CronJob object
type CronJobReconciler struct {
	client.Client
	Log logr.Logger
}
```

## rbac

控制器总是在集群中部署，又要访问集群对象资源，因此需要还有权限管理，这里使用代码自动生成标记来完成这些代码

```go
// +kubebuilder:rbac:groups=batch.tutorial.kubebuilder.io,resources=cronjobs,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=batch.tutorial.kubebuilder.io,resources=cronjobs/status,verbs=get;update;patch
```

## Reconcile函数


Reconcile用来完成调协操作。

```go
// 该函数是一个接口实现
// https://github.com/kubernetes-sigs/controller-runtime/blob/master/pkg/reconcile/reconcile.go
func (r *CronJobReconciler) Reconcile(req ctrl.Request) (ctrl.Result, error) {

	// Background returns a non-nil, empty Context. It is never canceled, has no
	// values, and has no deadline. It is typically used by the main function,
	// initialization, and tests, and as the top-level Context for incoming
	// requests.
	_ = context.Background()

	// 增加日志配置
	_ = r.Log.WithValues("cronjob", req.NamespacedName)

	// your logic here

	return ctrl.Result{}, nil
}
```

## SetupWithManager

最后我们将我们的调谐器加入到manager里面

```go
// Manager initializes shared dependencies such as Caches and Clients, and provides them to Runnables.
func (r *CronJobReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&batchv1.CronJob{}).
		Complete(r)
}
```