---
title: kubebuilder编写控制器之一项目初始化
date: 2019-09-28 12:12:28
tags:
- kubernetes
---

# kubebuilder介绍

kubebuilder是官方的crd控制器编写脚手架

<!--more-->

# 安装

kubebuilder是个二进制文件，跟随官方文档安装即可

[安装文档](https://book.kubebuilder.io/quick-start.html#installation)

# 创建CronJob控制器

本文将顺着官方的示例进行记录

## 初始配置

```bash
export GO111MODULE=on
export GOPROXY=https://mirrors.aliyun.com/goproxy/

mkdir CronJob
go mod init
```

## 项目初始化

```bash
cd CronJob
kubebuilder init --domain tutorial.kubebuilder.io
```

这一步主要生成基础文件

```bash
tree
.
├── Dockerfile
├── Makefile # 用来构建和部署的make脚本
├── PROJECT # 在kubebuilder中保存与项目相关的元数据
├── bin
│   └── manager
├── config # 用来部署控制器配置的 kustomize项目，每个目录都含有kustomization.yaml文件
│   ├── certmanager  # cert-manager 证书管理的crd  Issuer 和 Certificate
│   ├── default #包含基础的部署控制器的 kustomize文件，该目录可以部署与控制器相关的全部资源
│   ├── manager # 创建控制器的deploymnet资源
│   │   ├── kustomization.yaml
│   │   └── manager.yaml
│   ├── rbac # 与控制器有关的rbac资源
│   └── webhook # webhook service资源
├── go.mod # 处理包依赖关系
├── go.sum
├── hack
│   └── boilerplate.go.txt
└── main.go ##### 总入口
```

# 主入口介绍

主入口主要做了下面几件事

- 核心依赖包引入，包含controller-runtime和zap日志
- 声明scheme
- 实例化manger实例用于创建controller和webhook

```go
package main

import (
	"flag"
	"os"

	"k8s.io/apimachinery/pkg/runtime"
	clientgoscheme "k8s.io/client-go/kubernetes/scheme"
	_ "k8s.io/client-go/plugin/pkg/client/auth/gcp"
	//核心依赖引入
	ctrl "sigs.k8s.io/controller-runtime"
	//引入高性能日志库 zap
	"sigs.k8s.io/controller-runtime/pkg/log/zap"
	// +kubebuilder:scaffold:imports
)

var (
	// scheme 是Kinds和go type的映射
	scheme   = runtime.NewScheme()
	setupLog = ctrl.Log.WithName("setup")
)

func init() {
	_ = clientgoscheme.AddToScheme(scheme)

	// +kubebuilder:scaffold:scheme
}

func main() {

	// 进行命令行参数和监控指标的处理
	var metricsAddr string
	var enableLeaderElection bool
	flag.StringVar(&metricsAddr, "metrics-addr", ":8080", "The address the metric endpoint binds to.")
	flag.BoolVar(&enableLeaderElection, "enable-leader-election", false,
		"Enable leader election for controller manager. Enabling this will ensure there is only one active controller manager.")
	flag.Parse()

	// zap日志配置
	ctrl.SetLogger(zap.Logger(true))

	// 初始化一个manger实例，manger实例用来controller 和 webhooks
	// 需要传入 kubeconfig对象，用于和api-service进行认证交互 传入一些其他的选项
	mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
		Scheme:             scheme,
		MetricsBindAddress: metricsAddr,
		LeaderElection:     enableLeaderElection,
	})
	if err != nil {
		setupLog.Error(err, "unable to start manager")
		os.Exit(1)
	}

	// +kubebuilder:scaffold:builder


	//进行日志记录
	setupLog.Info("starting manager")
	if err := mgr.Start(ctrl.SetupSignalHandler()); err != nil {
		setupLog.Error(err, "problem running manager")
		os.Exit(1)
	}
}

```







