---
title: 仓库服务nexus使用
date: 2019-07-23 12:12:28
tags:
- tools
---

# 为什么需要nexus

nexus本质上一个文件服务器，那就先来一下能干啥

- 提供maven仓库服务
- 提供apt，yum源服务
- 提供raw也就是文件服务器使用

<!--more-->

# 安装

## docker部署

```bash
mkdir /opt/nexus && chown -R 200 /opt/nexus
docker run -d \
          -p 8081:8081 \
          -p 5000:5000 \
          --name nexus \
          -v /opt/nexus:/nexus-data \
          sonatype/nexus3
```

# 获取密码

首次登陆需要修改密码

```bash
cat /opt/nexus/admin.password
```

# 核心概念

## Blob Stores

Blob Stores就是nexus的存储“驱动”，目前支持两种，本质上都是文件存储，只不过一个在本地一个在远程

- file 就是指定一个目录，可以进行Soft限制，比如使用了多少就禁止存储了
- s3

当创建Repository的时候就要选择指定的Blob Stores。这样可以为不同的Repository指定不同的Blob Stores，在物理上进行区分

## Repository

### 类型

nexus的核心，不同的文件服务是通过不同种类的Repository提供的，每种Repository都含有下面几种类型

- Proxy Repository 就是远程仓库的镜像，比如maven仓库代理
- Hosted Repository 本地的仓库，文件就是存储在本地的一个文件目录内
- Repository Group 将多个仓库组成一个组，就可以通过一个url来访问多个仓库

### 种类

在nexus里面为Formats

![nexeusformage](https://qiniu.li-rui.top/nexeusformage.png)

其他的组件都是围绕Repository以及里面的文件来展开，比如

- Content Selectors 来选择Repository中的文档
- Routing Rules 来做Repository的防火墙配置
- Cleanup Policies 来为Repository做清理工作

### Components

Components就是仓库里面的文件各种压缩文件以及java文件等

### Assets

描述Components的元数据，和Components有映射的关系

# 认证与授权

## 认证

nexus可以使用ldap以及本地账户形式

## 授权

可以为不同的用户配置不同的权限，权限划分很细致

# 部署docker源

新建docker的时候注意选择端口

![docernexus](https://qiniu.li-rui.top/docernexus.png)

## push image

和harbor不同，不用新建专门的project直接tag路径就可以

```bash
docker tag sonatype/nexus3:latest 172.18.247.204:5000/su/kg:9.0
```

可能需要配置docker配置

```json
// /etc/docker/daemon.json
{
  "insecure-registries": ["172.18.247.204:5000"]
}
```

# raw配置

可以部署一个文件服务器

