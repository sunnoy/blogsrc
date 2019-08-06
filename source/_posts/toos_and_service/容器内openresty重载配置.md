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

apiVersion: v1
kind: ConfigMap
metadata:
  name: openresty
  namespace: default
data:
  up.conf: |
    upstream backend {
        server xxxxx.xxx.xxx;
    }
    server {
        listen       80;
        server_name  localhost;

      location /oio {
         proxy_pass http://backend;
     }
    }

---
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - args:
      - --volume-dir=/etc/config
      - --webhook-url=http://127.0.0.1/reload
      image: jimmidyson/configmap-reload:v0.2.2
      imagePullPolicy: IfNotPresent
      name: openresty-configmap-reload
      volumeMounts:
      - mountPath: /etc/config
        name: config-volume
        readOnly: true
    - name: nginx
      image: openresty/openresty:centos
      volumeMounts:
      - name: config-volume
        mountPath: /etc/nginx/conf.d/
  volumes:
    - name: config-volume
      configMap:
        name: openresty
        items:
        - key: up.conf
          path: up.conf
```