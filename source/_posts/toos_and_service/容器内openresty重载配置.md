---
title: 容器内openresty重载配置
date: 2019-08-02 12:12:28
tags:
- nginx
---

# openresty

openresty就是nginx加上lua脚本模块，因此可以利用lua脚本去执行一些系统命令，比如重载服务服务配置

<!--more-->

# 配置

通过location字段配置，内部执行lua脚本

```bash
#关键配置，要是root用户
user  root;
worker_processes  1;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;


    sendfile        on;
    keepalive_timeout  65;

    # 可以设定一个专用的虚拟主机
    server {
        listen       81;
        server_name  localhost;

      #配置重载
      location /reload {
        #这里这是重载openresty，主要是执行了一个shell命令
        content_by_lua 'os.execute("/usr/bin/openresty -s reload")';
      }
    }
}
```

# pod中使用

通过监测configmap挂载的目录，当有文件变动的时候就去执行一个web请求，进而触发相关的lua脚本

```yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - args:
      - --volume-dir=/etc/nginx/conf.d
      - --webhook-url=http://127.0.0.1:1989/reload
      image: jimmidyson/configmap-reload:v0.2.2
      imagePullPolicy: IfNotPresent
      name: openresty-configmap-reload
      volumeMounts:
      - mountPath: /etc/nginx/conf.d
        name: openresty-confd
        readOnly: true
    - name: nginx
      image: openresty/openresty:centos
      volumeMounts:
      - name: openresty-nginx
        mountPath: /usr/local/openresty/nginx/conf/nginx.conf
        subPath: nginx.conf
      - name: openresty-confd
        mountPath: /etc/nginx/conf.d
  volumes:
    - name: openresty-confd
      configMap:
        name: openresty-confd
    - name: openresty-nginx
      configMap:
        name: openresty-nginx
        items:
        - key: nginx.conf
          path: nginx.conf
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: openresty-nginx
  namespace: default
data:
  nginx.conf: |
    user  root;
    worker_processes  1;
    error_log  logs/error.log;
    events {
        worker_connections  1024;
    }
    http {
        include       mime.types;
        default_type  application/octet-stream;
        access_log  logs/access.log  main;
        sendfile        on;
        keepalive_timeout  65;
        server {
          listen       1989;
          server_name  localhost;
          location /reload {
            content_by_lua 'os.execute("/usr/bin/openresty -s reload")';
          }
        }
        include /etc/nginx/conf.d/*.conf;
    }

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: openresty-confd
  namespace: default
data:
  upstream.conf: |
    upstream kubernetes {
      server kubernetes:443;
    }

  server.conf: |
    server {
      listen       80;
      server_name  localhost;
      proxy_http_version 1.1;
      proxy_set_header Host $host:$server_port;
      proxy_set_header X-Forwarded-Host $host:$server_port;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Port $server_port;
      proxy_set_header X-Forwarded-Server $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Nginx-IP $server_addr;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection $connection_upgrade;

      location = /favicon.ico {
        return 404;
      }

      location / {
        deny all;
      }
    }
```