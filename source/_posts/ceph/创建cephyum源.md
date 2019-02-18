---
title: 创建cephyum源
date: 2019-02-10 12:12:28
tags:
- ceph
---

# 自建ceph yum源

配合ceph ansible来使用

<!--more-->

# 搭建nginx服务器

## 准备wwwroot目录

```bash
mkdir /data/nginx/wwwroot

```

## 配置nginx

### nginx配置文件

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

### docker服务配置文件

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

### 启动nginx

```bash
docker-compose -f /data/nginx/docker-compose.yml up -d
```

# 创建yum源

## 创建yum源目录

```bash
/data/nginx/wwwroot/ceph_12.2.8/rpm-luminous/el7/x86_64
```

## 下载ceph相关包

```bash
yum install --downloadonly --downloaddir=/data/nginx/wwwroot/ceph_12.2.8/rpm-luminous/el7/x86_64 ceph*
```

## 创建yum元数据

```bash
createrepo /data/nginx/wwwroot/ceph_12.2.8/rpm-luminous/el7/x86_64
```

## 下载yum源key

```bash
curl -L  http://mirrors.163.com/ceph/keys/release.asc -o /data/nginx/wwwroot/ceph_12.2.8
```

# playbook中使用

## 文件group_vars/all.yml

```yml
repo_ip: 192.168.6.13:81
ceph_mirror: http://{{ repo_ip }}/ceph_12.2.8
ceph_stable_key: http://{{ repo_ip }}/ceph_12.2.8/release.asc
ceph_stable_release: luminous
ceph_stable_repo: "{{ ceph_mirror }}/rpm-{{ ceph_stable_release }}"
```

## 执行完成后会创建源文件ceph_stable.repo

```bash
[root@matrix_03 yum.repos.d]# cat /etc/yum.repos.d/ceph_stable.repo
[ceph_stable]
baseurl = http://192.168.6.13:81/ceph_12.2.8/rpm-luminous/el7/$basearch
gpgcheck = 1
gpgkey = http://192.168.6.13:81/ceph_12.2.8/release.asc
name = Ceph Stable repo
```

