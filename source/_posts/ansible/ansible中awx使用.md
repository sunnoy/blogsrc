---
title: ansible中awx使用
date: 2018-11-29 12:12:28
tags:
- ansible
---

# ansible中awx使用

## 安装

### 安装依赖

使用docker进行安装，材料准备

- docker
- pip
- `pip install docker-py`
- [awx项目地址](https://github.com/ansible/awx/releases)

<!--more-->

### 安装

下载好项目文件

```bash
cd installer
#inventory中修改变量
postgres_data_dir=/opt/pgdocker
host_port=8001
use_docker_compose=false

```

部署命令

```bash
cd installer
ansible-playbook -i inventory install.yml

```
初始登录

用户名：admin
密码：password

## 基本使用

### 凭据配置credential

因为是在本地执行主机，因此选择credential type 为 `Machine`。

要为host指定一个organization

登录方式为密码登录或者sshkey登录

### 创建项目project

添加project，playbook的存放位置在gitlab上面，只需要添加`playbook`文件目录在仓库中，awx会自动寻找palybook

### 创建主机清单inventory

创建主机清单`inventory`，在主机清单里面创建组`groups`，在组里面添加主机`host`

#### 创建任务模板job template

找到模板看到小火箭进行查看