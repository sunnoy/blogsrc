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

```go
// CronJobSpec defines the desired state of CronJob
type CronJobSpec struct {
    // +kubebuilder:validation:MinLength=0

    // The schedule in Cron format, see https://en.wikipedia.org/wiki/Cron.
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

# 未完待续








