---
title: go基础
date: 2018-11-04 12:12:28
tags:
- go
---
# go基础

本文档根据go官方tour生成

## go环境变量

### go系统变量
- $GOROOT为系统中go的安装位置
- $GOARCH为系统的处理器架构可选为386、amd64 或 arm
- $GOOS操作系统类型，可选为darwin、freebsd、linux 或 windows
- $GOBIN为编译器和链接器的安装位置，默认为$GOROOT/bin
<!--more-->
### go交叉编译
go可以进行交叉编译，交叉编译的时候需要设置环境变量

- $GOHOSTOS目标机器的操作系统
- $GOHOSTARCH目标机器的处理器架构

### 项目环境变量

$GOPATH为项目目录，该目录下必须有src,pkg,bin三个文件夹，分别存放源码，包和编译过后的二进制文件

GOPATH可以有多个环境变量，但是go get只会下载到第一个目录，编译的时候会寻找所有的目录。


## 环境搭建

### 下载go

```bash
https://golangtc.com/download
```

### 设定GOROOT变量

```bash
#设定GOROOT
echo "export GOROOT=/data/go/go" >>  /etc/profile
#设定go二进制，go命令随处可用
echo "export PATH=$GOROOT/bin:$PATH" >>  /etc/profile
source /etc/profile
```

### 创建项目

#### 创建项目目录

```bash
mkdir -p /data/go/pcp
mkdir -p /data/go/pcp/{src,pkg,bin}
```

目录结构

- src存放源码，注意是存放的所有源码，推荐下面一个目录一个项目
- pkg编译后存放的文件
- bin编译后可以执行的二进制文件

src下面的文件夹可以一个应用也可以是一个包，这与main相关，main是程序的入口
#### 设定GOPATH

```bash
#此为临时设置
export GOPATH=/data/go/pcp
```

## 开发示意

### 建立包文件目录

```bash
mkdir /data/go/pcp/src/mymath
```

### 创建包源码

在mymath下创建文件sqrt.go。注意package的名称和目录名保持一致

```go
package mymath

func Sqrt(x float64) float64 {
    z := 0.0
    for i := 0; i < 1000; i++ {
        z -= (z*z - x) / (2 * x)
    }
    return z
}
```

### 编译包

有两种方法编译包

- 在任意地方执行`go install mymath`
- 进入/data/go/pcp/src/mymath执行`go install`

编译过后会在/data/go/pcp/pkg/linux_amd64($GOPATH/pkg/${GOOS}_${GOARCH})目录内创建`.a`文件，该文件就是编译好的应用包

```bash
ll /data/go/pcp/pkg/linux_amd64
total 4
-rw-r--r-- 1 root root 958 Nov  6 17:57 mymath.a
```

### 调用包

创建目录mathapp

```bash
mkdir /data/go/pcp/src/mathapp
```

在该目录下创建应用程序入口mathapp/main.go

```go
package main

import (
    "mymath"
    "fmt"
)

func main() {
    fmt.Printf("Hello, world.  Sqrt(2) = %v\n", mymath.Sqrt(2))
}
```

### 编译应用

在/data/go/pcp/src/mathapp下执行命令

```bash
#仅仅去编译
go build
```

该命令会在目录/data/go/pcp/src/mathapp内生成二进制文件mathapp

执行命令

```bash
go install
```
该命令会在目录/data/go/pcp/bin内生成二进制文件mathapp

### 现在的目录结构

```bash
.
├── bin
│   └── mathapp
├── pkg
│   └── linux_amd64
│       └── mymath.a
└── src
    ├── mathapp
    │   └── main.go
    └── mymath
        └── sqrt.go
```

### 获取远程包

使用命令`go get`来获取远程包，该命令主要完成

- 使用git等工具把源码拉取到目录/data/go/pcp/src内
- 执行`go install`命令将拉取的远程包编译

go get支持的远程包类型有github、googlecode、bitbucket、Launchpad，但是要有相应的工具，改命令会自动解析import来安装所有的依赖。

```bash
├── bin
│   └── mathapp
├── pkg
│   └── linux_amd64
│       ├── github.com
│       └── mymath.a
└── src
    ├── github.com
    │   └── astaxie
    ├── mathapp
    │   └── main.go
    └── mymath
        └── sqrt.go
```

## 包，变量，函数

### 包

go程序有多个包组成，运行程序的入口是名称为`main`的包，

下面程序在入口`main`，在控制台返回10以内的随机整数
```go
package main

//也可以单行引入import "fmt"
import (
    //IO包
    "fmt"
    //生成随机数包
	"math/rand"
)

func main() {
    //字符链接为逗号
	fmt.Print("the nub is"," ",rand.Intn(10))
}
```

### 函数

函数示例

```go
package main

import "fmt"

func main() {
	fmt.Print(add(66,66))
	
}

//变量名称接类型 要有返回函数类型值
//func add(x int, y int) int {

//可以
func add(x, y int) int {
	return x + y
}
```

## 数组









