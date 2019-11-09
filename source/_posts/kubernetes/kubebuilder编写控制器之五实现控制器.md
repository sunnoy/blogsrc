---
title: kubebuilder编写控制器之五实现控制器
date: 2019-10-09 12:22:28
tags:
- kubernetes
- crd
---

# 控制器的工作

依照API的设计我们的控制器需要做那些支撑呢？也就是说做那些具体的逻辑呢？

- 1 对定时任务的载入
- 2 对任务的罗列和状态监控
- 3 对历史任务的清理
- 4 任务的暂停
- 5 获取下一个执行的时间点
- 6 在没有限制和并发的情况下进行创建新的任务
- 7 对任务重排

<!--more-->


# 代码实现

下面的代码均是对文件controllers/cronjob_controller.go的补充。分别对不同的功能点来实现代码

# 基础代码准备

## 依赖引入

这些引入在以后的代码中进行说明

```go
import (
    "context"
    "fmt"
    "sort"
    "time"

    "github.com/go-logr/logr"
    "github.com/robfig/cron"
    kbatch "k8s.io/api/batch/v1"
    corev1 "k8s.io/api/core/v1"
    apierrs "k8s.io/apimachinery/pkg/api/errors"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/runtime"
    ref "k8s.io/client-go/tools/reference"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"

    batch "tutorial.kubebuilder.io/project/api/v1"
)
```

## Reconciler完善

[具体完善代码](https://github.com/sunnoy/CronJob/blob/master/controllers/cronjob_controller.go)