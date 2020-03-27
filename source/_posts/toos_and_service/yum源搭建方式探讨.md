---
title: yum源搭建方式探讨
date: 2018-11-04 12:12:28
tags:
- linux
---
# yum源

yum源就是centos系统安装软件包用的仓库，yum服务端是一个http服务器，yum客户端也就是yum命令本质上是一个http客户端。安装文件就是yum客户端向服务端发送http请求的过程。本文介绍几中搭建方法。

<!--more-->

# rsync源

rsync源是同步共有源到本地

## nginx安装

```bash
yum install nginx -y
```

### 目录准备

```bash
cd /data/yum/
mkdir -p centos/7/os/x86_64/
mkdir -p centos/7/extras/x86_64/
mkdir -p centos/7/updates/x86_64/

mkdir -p epel/7/x86_64/

```

## 同步命令

### 目录大小概览

- os 9.8g
- extras 2.7g
- updates 19g
- epel 14g

### 同步命令

```bash
# 同步base源，小技巧，我们安装系统的光盘镜像含有部分rpm包，大概3G，这些就不用重新下载。
/usr/bin/rsync -avzP rsync://rsync.mirrors.ustc.edu.cn/centos/7/os/x86_64/ /data/yum/centos/7/os/x86_64/

/usr/bin/rsync -avzP rsync://rsync.mirrors.ustc.edu.cn/centos/7/extras/x86_64/ /data/yum/centos/7/extras/x86_64/

/usr/bin/rsync -avzP rsync://rsync.mirrors.ustc.edu.cn/centos/7/updates/x86_64/ /data/yum/centos/7/updates/x86_64/
# epel源
/usr/bin/rsync -avzP --exclude=debug rsync://rsync.mirrors.ustc.edu.cn/epel/7/x86_64/ /data/yum/epel/7/x86_64/
```

# 本地源

在本地创建源

## 创建包的目录

```bash
/data/yum/centos/7/os/x86_64/Packages/
```

## 创建repo文件

local.repo

```bash
[Local]
name=Local Yum
baseurl=file:///data/yum/centos/7/os/x86_64/Packages/
gpgcheck=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
enabled=1
```

## 文件目录内下载包

```bash
yum -y install --downloadonly --downloaddir=/home/yum/new_yum ceph-12.2.7-0.el7.x86_64 

#增加索引
yum install createrepo -y 
createrepo /home/yum/new_yum
```

该命令会在rpm包文件内创建repodata目录并创建相关文件

```bash
[root@bogon repodata]# ls
029eb7e0371af46426c61d9de29c1cedfb01a94ed354ef4c705c8e7c7eeec126-other.sqlite.bz2
051e87a08b5c33c6b6bbfea39dbb255a013573525664e5e155dc7e3210657b58-filelists.xml.gz
59c7a4d7beaaf9f6ec552adbaa368cbc5ced7c60470a1b4604938164bcb2aa21-primary.sqlite.bz2
b13ad2ea3c6feb1480e5a927fa5251d6b32aab151ab14b5b6d4328b173c3aad2-other.xml.gz
bb691b0d5717f58a697f2e1ce730bd33c2ccaa001f6e4e018d1a98baac6592bd-filelists.sqlite.bz2
d79391c0de217eb6b2ea7c6378ce887e12178271d747158c50b8dfce541c52cf-primary.xml.gz
repomd.xml
[root@bogon repodata]# pwd
/data/nginx/wwwroot/yum/repodata

```

# 内网源
其实是在本地源的基础上向本机外提供yum源服务

## nginx配置

主要是开启文件索引功能

### 核心配置

```bash
autoindex on;
autoindex_exact_size off;
autoindex_localtime on;
```

### 全部nginx主配置

```bash
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    include /etc/nginx/conf.d/*.conf;

    server {
        listen       80 default_server;
        server_name  _;

        include /etc/nginx/default.d/*.conf;

        location / {
        root         /home/yum;
                autoindex on;
                autoindex_exact_size off;
                autoindex_localtime on;
        }

        error_page 404 /404.html;
            location = /40x.html {
        }

        error_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }
    }


}

```

## docker-compose启动

```yml
version: '3.1'
services:
  jenkins:
    image: nginx
    container_name: nginx
    user: root
    ports:
      - 81:80
    volumes:
      - /data/nginx/nginx.conf:/etc/nginx/nginx.conf
      - /data/nginx/wwwroot:/home/yum

```

## repo文件配置

baseurl就是个站点url可以为非80端口，但是入口必须是rpm包所在目录，因为会在rpm所在目录下找文件夹repodata，如果不是rpm所在目录就会报出下面的错误

```bash
[root@com03 yum.repos.d]# yum makecache
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
http://173.16.1.54:81/repodata/repomd.xml: [Errno 14] HTTP Error 404 - Not Found

```

## 配置示例

```bash
[base]
name=CentOS-$releasever - Base
baseurl=http://192.168.6.13:81/yum/
enable=1
gpgcheck=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

#released updates
[updates]
name=CentOS-$releasever - Updates
baseurl=http://192.168.6.13:81/yum/
gpgcheck=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
```

## 完成后记得重新生成缓存

```bash
yum makecache
```



