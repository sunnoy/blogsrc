---
title: go常用命令
date: 2020-01-06 12:12:28
tags:
- go
---

# go的命令

go提供了一些命令用来编译、依赖的管理

<!--more-->

# go build

主要用于编译代码

有三种编译的对象

## 无参数编译

直接执行命令`go build`，会搜索当前的文件目录中的go文件，然后编译出来还有包名称的二进制文件

## 编译指定的文件

```bash
go build file1.go file2.go……
```

## 编译包

```bash
# -o 为输出的文件，后面接的包名
go build -o main chapter11/goinstall
```

## 并发编译

```bash
-p n 开启并发编译，默认情况下该值为 CPU 逻辑核数
```

# go run

go run 命令会编译源码，并且直接执行源码的 main() 函数，不会在当前目录留下可执行文件。

# go install

该命令分为两步

- 第一步是生成结果文件（可执行文件或者 .a 包）
- 第二步会把编译好的结果移到 $GOPATH/pkg 或者 $GOPATH/bin

# go clean

go clean 命令是用来移除当前源码包和关联源码包里面编译生成的文件

# go get

借助代码管理工具通过远程拉取或更新代码包及其依赖包，并自动完成编译和安装

```bash
go get github.com/davyxu/cellnet
```

# go mod

go命令直接支持使用modules，包括记录和解析对其他模块的依赖性。modules替换旧的基于GOPATH的方法来指定在给定构建中使用哪些源文件。

## 配置mod

```bash
export GO111MODULE=on
```

## 创建mod

```bash
mkdir hello
cd hello

go mod init
```

## 常用命令

```bash
# 拉取缺少的模块，移除不用的模块。
go mod tidy 

# 下载依赖包
go mod download 

# 打印模块依赖图
go mod graph

# 将依赖复制到vendor下
go mod vendor 

# 校验依赖
go mod verify

# 解释为什么需要依赖
go mod why

# 依赖详情
go list -m -json all
```


