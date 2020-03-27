---
title: docker compose使用
date: 2018-11-04 12:12:28
tags:
- docker
- docker-compose
---
# docker compose使用

## 理解

compose就是把多个容器汇聚到一块儿形成一个容器组来提供服务，体现了微服务的思想。

<!--more-->

## 如何使用

### 常用命令

```bash
#指定yml文件和项目名称
docker-compose -f docker-compose.yml -p dongodng up -d
#进行所需的服务镜像构建
docker-compose build
#打印出详细的config文件
docker-compose config
#创建容器但是不运行
docker-compose create
#停掉服务，删除容器，不删除镜像
docker-compose down
#接受服务之间的互动事件，如进行健康检查等
docker-compose events
#对某个容器执行命令
docker-compose exec 容器名称 命令
#对某个服务查看日志
docker-compose logs -ft mysql
#查看服务状态
docker-compose ps
#重启服务
docker-compose restart/start/stop [服务名称]
#运行某个服务
docker-compose run [服务名称]
#查看服务中使用的镜像
docker-compose images [服务名称]
#强制停止容器，删除
docker-compose kill
#删除停止的容器
docker-compose rm

#想要重启单个服务容器
#先进行停掉服务
docker-compose stop test1
#然后删除容器
docker-compose rm
#再次启动该服务
docker-compose up -d  test1
```

### 定义出容器镜像的Dockerfile

定义出Dockerfile以后就可以方便的创建镜像

### 定义服务

### 启动compose

```bash
#-d为保持后台启动
docker-compose up -d
```

## 安装compose

```bash
curl -L "https://github.com/docker/compose/releases/download/1.22.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

chmod +x  /usr/local/bin/docker-compose
```

## docker-compose.yml配置

### 基本概念

一个docker-compose.yml需要定义出services, networks 和volumes。这三者和容器操作的对应关系如下：

| compose中 | docker中 | 
| ------ | ------ | 
| services | docker container create | 
| networks | docker network create | 
| volumes | docker volume create | 

compose中还可以使用环境变量

compose现在存在三个版本的文件格式，在配置文件的第一行就要指定所用的版本

```bash
version: '3'
services:
  webapp:
    build: ./dir
```

### services主要配置

#### 使用Dockerfile来构建镜像

```yml
version: '3'
services:
  webapp:
    #使用buid字段来通过Dockerfile构建镜像
    build:
      context: ./dir
      dockerfile: Dockerfile-alternate
    container_name: my-web-container
    #容器内目录映射
    devices:
      - "/dev/ttyUSB0:/dev/ttyUSB0"
    #指定自定义dns
    dns: 8.8.8.8
    #覆盖入口
    entrypoint: /code/entrypoint.sh


    
```

#### container_name

配置容器名称

```yml
container_name: my-web-container
```

#### depends_on容器依赖

web服务启动之前会启动db和redis，**只是会等待启动，而不是等db和redis服务可以正常提供后再启动web**比如命令`docker-compose up web`

```yml
version: '3'
services:
  web:
    build: .
    depends_on:
      - db
      - redis
    
  redis:
    image: redis
  db:
    image: postgres
```

#### 环境变量

```yml
version: '3'
services:
  web:
    build: .
    depends_on:
      - db
      - redis
    #引入环境变量文件
    env_file: .env
    #直接给出环境变量
    environment:
      RACK_ENV: development
      SHOW: 'true'
      SESSION_SECRET:
    #暴露出端口，仅仅供容器之间访问，link的容器访问，没有和宿主机绑定
    expose:
      - "3000"
      - "8000"
    
```

#### 健康检查

compose中的健康检查主要和Dockerfile中的一样，只是这个在容器外边执行

```yml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost"]
  interval: 1m30s
  timeout: 10s
  retries: 3
  start_period: 40s
```

#### 指定服务所用的image

可以指定的格式

```yml
image: redis
image: ubuntu:14.04
image: tutum/influxdb
image: example-registry.com:4000/postgresql
image: a4bc65fd
```

#### 日志记录

```yml
services:
  some-service:
    image: some-service
    logging:
      #记录驱动json-file syslog none
      driver: "json-file"
      options:
        max-size: "200k"
        max-file: "10"
```

#### network

网络模式

```yml
network_mode: "bridge"
network_mode: "host"
network_mode: "none"
network_mode: "service:[service name]"
network_mode: "container:[container name/id]"

```

要加入的网络

```yml
services:
  some-service:
    networks:
     - some-network
     - other-network
```

为容器配置静态IP，在compose中service name 和 container_name 默认会做服务发现，也就是说可以在容器container1里面ping通container2和service2，hostname之间不会通。

```yml
version: "2.4"
services:
  service1:
    image: docker.li-rui.top/library/centos:7.5.1804
    container_name: container1
    domainname: domainname1
    hostname: hostname1
    mac_address: 02:42:ac:11:65:41
    networks:
      app_net:
        ipv4_address: 173.16.1.2
    
  service2:
    image: docker.li-rui.top/library/centos:7.5.1804
    container_name: container2
    domainname: domainname2
    hostname: hostname2
    mac_address: 02:42:ac:11:65:42
    networks:
      app_net:
        ipv4_address: 173.16.1.3
    
networks:
  app_net:
    driver: bridge
    ipam:
      driver: default
      config:
        - subnet: 173.16.0.0/16
          ip_range: 173.16.1.0/24
          gateway: 173.16.1.1

```

#### 端口映射

```yml
#短语法
ports:
  - "3000"
  - "3000-3005"
  - "8000:8000"
  - "9090-9091:8080-8081"
  - "49100:22"
  - "127.0.0.1:8001:8001"
  - "127.0.0.1:5000-5010:5000-5010"
  - "6060:6060/udp"
#长语法
ports:
  #targer为容器内的端口
  - target: 80
    published: 8080
    protocol: tcp
    mode: host
```

#### 容器内的内核参数调整

```yml
sysctls:
  net.core.somaxconn: 1024
  net.ipv4.tcp_syncookies: 0

sysctls:
  - net.core.somaxconn=1024
  - net.ipv4.tcp_syncookies=0
#文件句柄数设置
ulimits:
  nproc: 65535
  nofile:
    soft: 20000
    hard: 40000
```

#### 容器的数据持久化

```yml
version: "3.2"
services:
  web:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      #长语法
      #target为容器内的路径
      - type: volume
        source: mydata
        target: /data
        volume:
          nocopy: true
      - type: bind
        source: ./static
        target: /opt/app/static
      #短语法
      - /opt/data:/var/lib/mysql
      - mydata:/var/lib/mysql

networks:
  webnet:

volumes:
  mydata:
```

#### 重启策略restart

```yml
restart: "no"
restart: always
restart: on-failure
restart: unless-stopped
```
### volumes主要配置

这个是和services平级的字段，其所定义的volume可以被多个服务进行复用

docker中的数据库存储

![dockervolumer](https://qiniu.li-rui.top/dockervolumer.png)

compose中volumes使用的是Volume方式

下面例子可以让数据库的数据库文件进行周期性备份

```yml
version: "3"

services:
  db:
    image: db
    #使用的是Volume方式
    volumes:
      - data-volume:/var/lib/db
  backup:
    image: backup-service
    volumes:
      - data-volume:/var/lib/backup/data

volumes:
  #使用的是Bind mounts方式
  data-volume:
```
#### driver


#### external

表示持久化存储卷不受compose控制，compose启动前就应该创建好

```yml
version: '2'

services:
  db:
    image: postgres
    volumes:
      - data:/var/lib/postgresql/data

volumes:
  data:
    external: tru
```

### networks主要配置

#### 网络查看

```bash
docker network ls
docker network inspect bridg
```

compose会在up的时候建立一个默认网络

#### driver

可选有：bridge(默认) overlay host 或者 none

#### internal和external

如果要创建外部隔离的覆盖网络，您可以将此internal选项设置为true

```yml
networks:
  harbor:
    #docker会创建网络
    external: false
```

首先建立docker网络

```bash
host_net=eth3
bri_name=pcp
bri_ip=173.16.1.119
docker_net=app_net

ip link add name $bri_name type bridge
ip link set $bri_name up
ip link set dev $host_net master $bri_name
ip addr add $bri_ip/16 dev $bri_name
 
docker network create --subnet=173.16.0.0/16 --gateway=$bri_ip -o com.docker.network.bridge.name=$bri_name $docker_net
```

docker compose使用创建的网络

```yml
networks:
  outside:
    external: true
```


### 变量使用

可以使用`${POSTGRES_VERSION}`变量，如果`POSTGRES_VERSION`定义的话。

# docker stack

## 介绍
docker stack可以直接使用docker-compose.yml文件而不用安装`docker-compose`

## 使用区别

```bash
docker-compose -f docker-compose up
docker stack deploy -c docker-compose.yml somestackname
```

## 常用命令

```bash
Usage:  docker stack [OPTIONS] COMMAND
Manage Docker stacks
Options:
      --orchestrator string   Orchestrator to use (swarm|kubernetes|all)
Commands:
  deploy      Deploy a new stack or update an existing stack
  ls          List stacks
  ps          List the tasks in the stack
  rm          Remove one or more stacks
  services    List the services in the stack
```






