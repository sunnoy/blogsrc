---
title: docker之安装与卸载
date: 2018-11-04 12:12:28
tags:
- docker
---

# docker之安装与卸载

## 1.1 在线安装docker

### 卸载旧的docker

```bash
sudo yum remove docker docker-common docker-selinux docker-engine
```

### 设置库

```bash
yum install -y yum-utils device-mapper-persistent-data lvm2
```

<!--more-->
### 设定稳定分支
这个是必须要设置的

```bash
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
#接着更新一下
yum makecache fast
```
### 安装docker CE 社区版

```bash
#安装最新版
yum install docker-ce -y
#到此下载最新版离线包
https://mirrors.ustc.edu.cn/docker-ce/linux/centos/7/x86_64/stable/Packages/

yum install docker-ce-18.06.1.ce-3.el7.x86_64.rpm
#安装指定版本
#首先检索版本
yum list docker-ce --showduplicates | sort -r
#然后安装
sudo yum install <FULLY-QUALIFIED-PACKAGE-NAME> -y
```

### 启动docker，上面的安装已经将docker安装成服务

```bash
sudo systemctl start docker
```
## 1.2 卸载docker

- 卸载安装包
```bash
sudo yum remove docker-ce
```

- 删除镜像文件

```bash
sudo rm -rf /var/lib/docker
```

## 1.4 docker添加代理
- 创建文件夹以及文件

```bash
mkdir -p /etc/systemd/system/docker.service.d
vi /etc/systemd/system/docker.service.d/http-proxy.conf
```
- 文件内写入代理配置并对局域网不代理

```bash
[Service]
Environment="HTTP_PROXY=http://10.100.0.236:1080/" "NO_PROXY=localhost,127.0.0.1,daocloud.io"
```

- 使配置生效，并检查

```bash
systemctl daemon-reload
systemctl show --property=Environment docker
Environment=HTTP_PROXY=http://proxy.example.com:80/
```
- 重启docker

```bash
systemctl restart docker
```
## 1.5 docker添加更换阿里云源


```bash

echo "net.bridge.bridge-nf-call-ip6tables = 1" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-iptables = 1" >> /etc/sysctl.conf
sysctl -p
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://jngj6il2.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

## 默认位置更改

```bash
sudo systemctl stop docker
ln -s /root/data/docker /var/lib/docker
sudo systemctl start docker
```

