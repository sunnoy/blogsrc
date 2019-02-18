---
title: jenkins多节点
date: 2018-11-04 12:12:28
tags:
- jenkins
---
## jenkins多节点

### 1. 建立从节点

#### 1.1 容器方式

- 建立工作目录

```bash
#主机内创建目录
mkdir /opt/agent
#创建从节点工作目录
mkdir /opt/agent/workagent

#赋予Jenkins用户权限1000
chown 1000:1000 -R /opt/agent

```
<!--more-->

- 容器启动

```bash
#镜像系统环境基于Debian 9，主机系统环境Centos 7
#-p 50000可以不加，因为使用的docker网桥
docker run -d -p 8005:8080 \
       -v /opt/ansible:/var/jenkins_home \
       -v /var/run/docker.sock:/var/run/docker.sock \
       -v $(which docker):/usr/bin/docker \
       -v /usr/lib64/libltdl.so.7:/lib/libltdl.so.7 \
      #注意ssh文件权限为root，到容器中需要更改权限
       -v /root/.ssh:/var/jenkins_home/.ssh \
	     --name ansible --rm jenkins
```

- 容器内配置ssh

```bash
#进入agent容器
docker exec -it -u root agent bash

#更新库并安装sshd
apt update && apt install openssh-server -y

service ssh start

#切换jenkins用户修改密码
su jenkins
passwd
#然后两次输入密码
```
#### 1.2. agent添加

- 添加从节点credentials

`credentials` -> `global` -> `add credentials` ->

`kind` -> input `Username` -> input `password` -> input `id` other things

- ssh方式通讯

`Manage jenkins` -> `Manage Nodes` ->

`New Node` -> input `Node name` -> checked `Parmanent Agent` -> `OK`

`of executors` may cpus

`remote root directory` is `/opt/agent/workagent`

`labels` is `agent`or other some things

`Launch method` is `... via ssh` -> select credentials


- java web start通讯

`Manage jenkins` -> `Manage Nodes` ->

`New Node` -> input `Node name` -> checked `Parmanent Agent` -> `OK`

`of executors` may cpus

`remote root directory` is `/opt/agent/workagent`

`labels` is `agent`or other some things

`Launch method` is `via java web start`

`Manage jenkins` -> `configure global security` -> Agents

-> TCP port for JNLP agents `fixed : Aport` **docker run -p `Aport`:port**

download `agent.jar` to agent Node , run

```bash
nohup java -jar agent.jar -jnlpUrl http://devops.li-rui.top:8002/computer/agent2/slave-agent.jnlp -secret 89f3b8662212fe8f0e08648ad4119f008c2d5f9246d9c6663bd4011e6ad66fe2 -workDir "/var/jenkins_home/dong" &
```
