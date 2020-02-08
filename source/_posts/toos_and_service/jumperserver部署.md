---
title: jumperserver部署
date: 2020-02-02 12:12:28
tags:
- Linux
---

# 介绍

jumperserver是一个开源的堡垒机

<!--more-->

# 部署

使用docker-compose

```yaml
version: '3'
services:

  redis:
    image: wojiushixiaobai/jms_redis:${Version}
    container_name: jms_redis
    hostname: redis
    restart: always
    tty: true
    environment:
      REDIS_PORT: $REDIS_PORT
      REDIS_PASSWORD: $REDIS_PASSWORD
    volumes:
      - /opt/data-jump/redis:/var/lib/redis/
    networks:
      - jumpserver

  core:
    image: wojiushixiaobai/jms_core:${Version}
    container_name: jms_core
    hostname: core
    restart: always
    tty: true
    environment:
      SECRET_KEY: $SECRET_KEY
      BOOTSTRAP_TOKEN: $BOOTSTRAP_TOKEN
      DB_HOST: $DB_HOST
      DB_PORT: $DB_PORT
      DB_USER: $DB_USER
      DB_PASSWORD: $DB_PASSWORD
      DB_NAME: $DB_NAME
      REDIS_HOST: $REDIS_HOST
      REDIS_PORT: $REDIS_PORT
      REDIS_PASSWORD: $REDIS_PASSWORD
    depends_on:
      - redis
    volumes:
      - /opt/data-jump/core/static:/opt/jumpserver/data/static
      - /opt/data-jump/core/media:/opt/jumpserver/data/media
    networks:
      - jumpserver

  koko1:
    image: wojiushixiaobai/jms_koko:${Version}
    container_name: jms_koko1
    hostname: koko1
    restart: always
    tty: true
    environment:
      CORE_HOST: http://core:8080
      BOOTSTRAP_TOKEN: $BOOTSTRAP_TOKEN
    depends_on:
      - core
      - redis
    volumes:
      - /opt/data-jump/koko1:/opt/koko/data/keys
    # ports:
    #   - 2222:2222
    networks:
      jumpserver:
        ipv4_address: 10.10.10.11

  koko2:
    image: wojiushixiaobai/jms_koko:${Version}
    container_name: jms_koko2
    hostname: koko2
    restart: always
    tty: true
    environment:
      CORE_HOST: http://core:8080
      BOOTSTRAP_TOKEN: $BOOTSTRAP_TOKEN
    depends_on:
      - core
      - redis
    volumes:
      - /opt/data-jump/koko2:/opt/koko/data/keys
    # ports:
    #   - 2222:2222
    networks:
      jumpserver:
        ipv4_address: 10.10.10.12

  koko3:
    image: wojiushixiaobai/jms_koko:${Version}
    container_name: jms_koko3
    hostname: koko3
    restart: always
    tty: true
    environment:
      CORE_HOST: http://core:8080
      BOOTSTRAP_TOKEN: $BOOTSTRAP_TOKEN
    depends_on:
      - core
      - redis
    volumes:
      - /opt/data-jump/koko3:/opt/koko/data/keys
    # ports:
    #   - 2222:2222
    networks:
      jumpserver:
        ipv4_address: 10.10.10.13

  koko4:
    image: wojiushixiaobai/jms_koko:${Version}
    container_name: jms_koko4
    hostname: koko4
    restart: always
    tty: true
    environment:
      CORE_HOST: http://core:8080
      BOOTSTRAP_TOKEN: $BOOTSTRAP_TOKEN
    depends_on:
      - core
      - redis
    volumes:
      - /opt/data-jump/koko4:/opt/koko/data/keys
    # ports:
    #   - 2222:2222
    networks:
      jumpserver:
        ipv4_address: 10.10.10.14

  # guacamole:
  #   image: wojiushixiaobai/jms_guacamole:${Version}
  #   container_name: jms_guacamole
  #   hostname: guacamole
  #   restart: always
  #   tty: true
  #   environment:
  #     JUMPSERVER_SERVER: http://core:8080
  #     BOOTSTRAP_TOKEN: $BOOTSTRAP_TOKEN
  #     JUMPSERVER_KEY_DIR: /config/guacamole/keys
  #     GUACAMOLE_HOME: /config/guacamole
  #     GUACAMOLE_LOG_LEVEL: ERROR
  #     JUMPSERVER_CLEAR_DRIVE_SESSION: 'true'
  #     JUMPSERVER_ENABLE_DRIVE: 'true'
  #   depends_on:
  #     - core
  #     - redis
  #   volumes:
  #     - /opt/data-jump/guacamole/keys:/config/guacamole/keys
  #   networks:
  #     - jumpserver

  nginx:
    image: wojiushixiaobai/jms_nginx:${Version}
    container_name: jms_nginx
    hostname: nginx
    restart: always
    tty: true
    depends_on:
      - core
      - koko1
      - koko2
      - koko3
      - koko4
      - redis
    volumes:
      - /opt/data-jump/core/static:/opt/jumpserver/data/static
      - /opt/data-jump/core/media:/opt/jumpserver/data/media
      - /opt/data-jump/nginx/nginx.conf:/etc/nginx/nginx.conf
    ports:
      - 80:80
    networks:
      - jumpserver


networks:
  jumpserver:
    driver: bridge
    ipam:
      config:
        - subnet: 10.10.10.0/24

```

# koko和guacamole注册逻辑

koko和guacamole像core进行注册，注册的身份认证主要通过环境变量来约定同一个key

```bash
$BOOTSTRAP_TOKEN
```

注册后koko会在/opt/koko/data/keys下面生成一个文件,该文件用来标识自己

```bash
# .access_key 
e84196b6-b16iiiiiii3c79914bttttt
```

同样会将自己IP进行上报

## 注册信息持久化

通过固定IP的方式将IP固定，通过将标识文件挂出来进行固定信息

# nginx

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
                      '"$http_user_agent" "$http_x_forwarded_for" "$upstream_addr"';

    # access_log  /var/log/nginx/access.log  main;
    access_log off;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;
    # include /etc/nginx/conf.d/*.conf;

    upstream koko {
        # ip_hash;
        server koko1:5000;
        server koko2:5000;
        server koko3:5000;
        server koko4:5000;
    }

    server {
        listen 80;
        server_name  _;

        client_max_body_size 100m;  # 录像及文件上传大小限制

        location /luna/ {
            try_files $uri / /index.html;
            alias /opt/luna/;
        }
        location /media/ {
            add_header Content-Encoding gzip;
            root /opt/jumpserver/data/;
        }
        location /static/ {
            root /opt/jumpserver/data/;
        }
        location /koko/ {
            proxy_pass       http://koko;
            proxy_buffering off;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header Host $host;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            access_log off;
        }
        # location /guacamole/ {
        #     proxy_pass       http://guacamole:8080/;
        #     proxy_buffering off;
        #     proxy_http_version 1.1;
        #     proxy_request_buffering off;
        #     proxy_set_header Upgrade $http_upgrade;
        #     proxy_set_header Connection $http_connection;
        #     proxy_set_header X-Real-IP $remote_addr;
        #     proxy_set_header Host $host;
        #     proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        #     access_log off;
        # }
        location /ws/ {
            proxy_pass http://core:8070;
            proxy_buffering off;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header Host $host;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            access_log off;
        }
        location / {
            proxy_pass http://core:8080;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header Host $host;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }
}
```

# 环境变量

```bash
# 版本号可以自己根据项目的版本修改
Version=1.5.6

# MySQL
DB_HOST=mmmmmm
DB_PORT=3306
DB_USER=jumpserver
DB_PASSWORD=mmmmmm
DB_NAME=jumpserver

# Redis
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_PASSWORD=mmmm

# Core
SECRET_KEY=mmmmmm
BOOTSTRAP_TOKEN=mmmmm

##
# SECRET_KEY 保护签名数据的密匙, 首次安装请一定要修改并牢记, 后续升级和迁移不可更改, 否则将导致加密的数据不可解密。
# BOOTSTRAP_TOKEN 为组件认证使用的密钥, 仅组件注册时使用。组件指 koko、guacamole
```

# 关于升级

## 前提

jumperserver本身已经做好了数据库的相关更改，这依赖与djanggo的数据库迁移功能

## 迁移升级

适用与迁移部署然后升级场景，应该先使用老版本在新的环境部署，然后升级

## 核心步骤

- 在新的环境部署好和老版本一样的版本的应用
- 数据库导出

```bash
mysqldump -uroot -p jumpserver > /opt/jumpserver.sql
```
- 数据库导入

```bash
docker run --rm -it -v /root:/root mysql:5.7 bash
cd 
mysql -h rm-xxxxxxxxxx.mysql.rds.aliyuncs.com -u jumpserver -pxxxxxxx

use jumper;
source jumpserver.sql;
```

- 启动就版本的jumperserver

- 将新的jumperserver镜像tag进行替换

- 启动新的服务，就升级完成

## 环境清理

```bash
rm -rf /opt/data-jump/koko1/.access_key
rm -rf /opt/data-jump/koko2/.access_key
rm -rf /opt/data-jump/koko3/.access_key
rm -rf /opt/data-jump/koko4/.access_key
rm -rf /opt/data-jump/core/static/*
rm -rf /opt/data-jump/core/media/*
```




