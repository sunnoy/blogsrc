---
title: docker常用命令
date: 2018-11-04 12:12:28
tags:
- docker
---
## docker常用命令

### 1. docker

```bash
#查看docker版本
docker -v
#查看docker安装信息，以及状态信息
docker info

-t 指定镜像的REPOSITORY名称

#登录docker，然后输入用户名和密码
docker login
#创建自己的版本库
#先打标签
docker tag dong:latest sunnoy/dong:test
#然后推送
docker push sunnoy/dong
#从官方库中拉取镜像
docker pull sunnoy/dong:test

#查看容器启动日志
docker logs -f 容器id/name

#获取镜像容器的详细信息
docker inspect 镜像id/容器id


```

<!--more-->
### 2. docker 镜像 image

- 基本使用

```bash
#在Dockerfile所在的目录构建image
docker build -t friendlyhello .
docker image COMMAND
ls 查看本地的镜像
#删除所有的镜像
docker image rm $(docker image ls -a -q)
#查看镜像列表
docker images

```

- 保存修改过的镜像

```bash
#找到修改过的容器id
a32023b2614c

#提交修改
docker commit a32023b2614c  centos:lr
```

### 3. dcontainers 容器

```bash
docker containers COMMAND
ls 查看本地的镜像
	-a 所有的容器
	-q 仅列出容器id
#删除所有的容器
docker container rm $(docker container ls -a -q)
#查看运行中的容器
docker ps
#查看所有容器
docker ps -a
#删除容器
docker container rm 容器id/name
#强制删除容器
docker container rm -f 容器id/name

#在容器内部执行一条命令
docker exec -it 容器id/名称 ls ~
#进入容器中终端交互式使用
docker exec -it 容器id/名称 bash
#要运行系统基础镜像需要添加-it选项，给予一个前台进程，可以保证容器不退出
docker run --name cen --rm -dit  centos

#已经安装好的
docker run --name cen --rm -dit  centos：lr

#查看对容器的修改
docker diff centos


```
- centos中配置sshd

```bash
yum update
yum insatll sshdserver
ssh-keygen -t rsa -f /etc/ssh/ssh_host_rsa_key
ssh-keygen -t rsa -f /etc/ssh/ssh_host_ecdsa_key
ssh-keygen -t rsa -f /etc/ssh/ssh_host_ed25519_key
/usr/sbin/sshd

```

### 4. run

docker run = docker create + docker start

```bash

#从镜像新建一个容器
docker create --name web -d -p 400:80 sunnoy/dong:test
#运行新的容器
docker start web

#docker run 参数
-p 主机与容器端口映射
-v 主机与容器持久化存储使用bind mount方式
--name 为容器命名
-d 容器后台运行
-u root 以哪个用户进入容器
-e 设定容器环境变量
-m 容器内存使用限制
--rm 容器退出时删除容器
--privileged 特权模式，在容器中后台运行程序

#容器内调用主机docker环境
-v /var/run/docker.sock:/var/run/docker.sock -v /usr/bin:/usr/bin/

#运行示例
docker run --name cen --rm -dit -v /var/run/docker.sock:/var/run/docker.sock -v /usr/bin/:/usr/bin/  centos

#停止一个运行的容器
docker stop id
#启动一个停止的容器
docker start id
```
### 5. 容器中使用主机docker环境

- 使用sock方式

主机为centos7容器内环境为centos7

```bash
docker run --name cen --rm -it  -v /var/run/docker.sock:/var/run/docker.sock  -v $(which docker):/usr/bin/docker -v /usr/lib64/libltdl.so.7:/usr/lib64/libltdl.so.7 centos

```

主机环境为centos7容器内环境为debian

```bash
docker run -p 8008:8080 -p 50001:50000 -v /opt/agent/:/var/jenkins_home -v /var/run/docker.sock:/var/run/docker.sock -v $(which docker):/usr/bin/docker -v /usr/lib64/libltdl.so.7:/lib/libltdl.so.7 --name agent --rm jenkins

```

## 镜像和本地文件传递

### 创建镜像

```bash
docker create --name teat e5e71bf8cc89
```

### 从镜像中复制文件

```bash
docker cp teat:/go/src/github.com/kumina/libvirt_exporter/libvirt_exporter ./
```

### 关于docker命令

```bash]
docker cp [OPTIONS] CONTAINER:SRC_PATH DEST_PATH|-
docker cp [OPTIONS] SRC_PATH|- CONTAINER:DEST_PATH
```

## 镜像转移

```bash
docker save -o pcp_images vnc:1.0
docker load --input pcp_images
```
