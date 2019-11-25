---
title: kubernetes-client-go
date: 2019-11-09 12:22:28
tags:
- kubernetes
---

# client-go介绍

client go是kubernetes的go语言客户端库

- 地址 https://github.com/kubernetes/client-go

<!--more-->

# 通过go mod 安装

```bash
export GO111MODULE=on
go mod init
# 需要指定集群的版本
go get k8s.io/client-go@kubernetes-1.15.0
```

# 使用

## 示例

### 代码

```go
package main

import (
	"flag"
	"fmt"
	"os"
	"path/filepath"
	"time"

	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
)

func main() {
	var kubeconfig *string
	if home := homeDir(); home != "" {
		kubeconfig = flag.String("kubeconfig", filepath.Join(home, ".kube", "config"), "(optional) absolute path to the kubeconfig file")
	} else {
		kubeconfig = flag.String("kubeconfig", "", "absolute path to the kubeconfig file")
	}
	flag.Parse()

	// use the current context in kubeconfig
	config, err := clientcmd.BuildConfigFromFlags("", *kubeconfig)
	if err != nil {
		panic(err.Error())
	}

	// create the clientset
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		panic(err.Error())
	}
	for {
		pods, err := clientset.CoreV1().Pods("").List(metav1.ListOptions{})
		if err != nil {
			panic(err.Error())
		}
		fmt.Printf("There are %d pods in the cluster\n", len(pods.Items))

		time.Sleep(10 * time.Second)
	}

}

func homeDir() string {
	if h := os.Getenv("HOME"); h != "" {
		return h
	}
	return os.Getenv("USERPROFILE") // windows
}
```

### 测试

```bash
go run connect-cluster/connect.go

There are 126 pods in the cluster
```