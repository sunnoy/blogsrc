---
title: kubebuilder编写控制器之三API设计
date: 2019-09-28 12:32:28
tags:
- kubernetes
- crd
---

# API设计原则

kubernetes的API中资源对象的信息是通过json来处理的。因此就涉及到序列化和反序列化

- 从对象信息到json就是序列化
- 反之为反序列化

另外在序列化的过程中会通过json的tag来控制序列化的过程，比如空值处理

<!--more-->

# spec字段设计

## 设计

对于定时任务我们定义了下面几个字段

- 任务触发时间周期
- 任务模版
- 任务执行超时
- 多任务并发处理
- 任务的暂定操作
- 保留历史任务数量限制

对于每个字段我们都会使用下面的注释来告诉controller-tools去生成相应的CRD manifest

```go
// +kubebuilder:validation:MinLength=0
```

可选字段具有注释

```go
// +optional
```

## 代码

代码在文件api/v1/cronjob_types.go内

### ConcurrencyPolicy

为了复用代码我们将并发策略进行抽取，创建一个ConcurrencyPolicy的type

```go
// ConcurrencyPolicy describes how the job will be handled.
// Only one of the following concurrent policies may be specified.
// If none of the following policies is specified, the default one
// is AllowConcurrent.
// +kubebuilder:validation:Enum=Allow;Forbid;Replace
type ConcurrencyPolicy string

const (
    // AllowConcurrent allows CronJobs to run concurrently.
    AllowConcurrent ConcurrencyPolicy = "Allow"

    // ForbidConcurrent forbids concurrent runs, skipping next run if previous
    // hasn't finished yet.
    ForbidConcurrent ConcurrencyPolicy = "Forbid"

    // ReplaceConcurrent cancels currently running job and replaces it with a new one.
    ReplaceConcurrent ConcurrencyPolicy = "Replace"
)
```

### CronJobSpec

下面对一些spec字段进行指定

```go
// CronJobSpec defines the desired state of CronJob
type CronJobSpec struct {
    // +kubebuilder:validation:MinLength=0

    // The schedule in Cron format, see https://en.wikipedia.org/wiki/Cron.
    // cron的格式

    Schedule string `json:"schedule"`

    // +kubebuilder:validation:Minimum=0

    // Optional deadline in seconds for starting the job if it misses scheduled
    // time for any reason.  Missed jobs executions will be counted as failed ones.
    // +optional 
    StartingDeadlineSeconds *int64 `json:"startingDeadlineSeconds,omitempty"`

    // Specifies how to treat concurrent executions of a Job.
    // Valid values are:
    // - "Allow" (default): allows CronJobs to run concurrently;
    // - "Forbid": forbids concurrent runs, skipping next run if previous run hasn't finished yet;
    // - "Replace": cancels currently running job and replaces it with a new one
    // +optional
    ConcurrencyPolicy ConcurrencyPolicy `json:"concurrencyPolicy,omitempty"`

    // This flag tells the controller to suspend subsequent executions, it does
    // not apply to already started executions.  Defaults to false.
    // +optional
    Suspend *bool `json:"suspend,omitempty"`

    // Specifies the job that will be created when executing a CronJob.
    JobTemplate batchv1beta1.JobTemplateSpec `json:"jobTemplate"`

    // +kubebuilder:validation:Minimum=0

    // The number of successful finished jobs to retain.
    // This is a pointer to distinguish between explicit zero and not specified.
    // +optional
    SuccessfulJobsHistoryLimit *int32 `json:"successfulJobsHistoryLimit,omitempty"`

    // +kubebuilder:validation:Minimum=0

    // The number of failed finished jobs to retain.
    // This is a pointer to distinguish between explicit zero and not specified.
    // +optional
    FailedJobsHistoryLimit *int32 `json:"failedJobsHistoryLimit,omitempty"`
}
```

### status

至于CronJob对象的status处理由构造器根据我们的CronJobSpec来自动完成，下面的代码保持就行

```go
// +kubebuilder:object:root=true

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

func init() {
	SchemeBuilder.Register(&CronJob{}, &CronJobList{})
}
```

# groupversion_info与zz_generated.deepcopy

## groupversion_info

我们知道，构造器就是将go语言中的type生成集群资源，那么我们首先需要引入schema的包，接着我们要配置我们自己的schema

```go
// Package v1 contains API Schema definitions for the batch v1 API group
// +kubebuilder:object:generate=true
// +groupName=batch.tutorial.kubebuilder.io
package v1

import (
	"k8s.io/apimachinery/pkg/runtime/schema"
	"sigs.k8s.io/controller-runtime/pkg/scheme"
)
```

创建schema的时候，我们一定是需要将我们的所有的type都生成集群资源，构造器为我们抽象出来了一些接口

```go
var (
	// 通过Group和Version的方式来将所有的type进行"搜集"
	// 然后统统加入构造器
	// GroupVersion is group version used to register these objects
	GroupVersion = schema.GroupVersion{Group: "batch.tutorial.kubebuilder.io", Version: "v1"}

	// SchemeBuilder is used to add go types to the GroupVersionKind scheme
	SchemeBuilder = &scheme.Builder{GroupVersion: GroupVersion}

	// AddToScheme adds the types in this group-version to the given scheme.
	AddToScheme = SchemeBuilder.AddToScheme
)
```

## zz_generated.deepcopy

这里面主要通过deepcopy方法来生成资源对象