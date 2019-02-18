---
title: docker构建镜像
date: 2018-11-04 12:12:28
tags:
- docker
---
# docker构建镜像

## 构建tomcat
<!--more-->
### Dockerfile

**tomcat启动需要使用catalina.sh而不是startup.sh**

```Dockerfile
FROM docker.li-rui.top/library/centos:7.5.1804
MAINTAINER pcp-team@lirui
LABEL author="pcp-team" \
      version="0.3"

WORKDIR /home

RUN mkdir -p {runtime,server} && \
    yum install iproute telnet tcpdump mkisofs kde-l10n-Chinese -y && \
    yum -y reinstall glibc-common && \
    yum clean all && \
    cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
    echo "Asia/shanghai" > /etc/timezone && \
    mkdir -p /data/image && \
    mkdir -p /home/iso_tool && \
    localedef -c -f UTF-8 -i zh_CN zh_CN.utf8

ADD jdk1.8.0_112.tar.gz runtime
ADD apache-tomcat-7.0.69.tar.gz server

ENV LC_ALL "zh_CN.UTF-8"
ENV APPPORT 8080
ENV CATALINA_HOME /home/server/apache-tomcat-7.0.69
ENV CATALINA_BASE /home/server/apache-tomcat-7.0.69
ENV JAVA_HOME /home/runtime/jdk1.8.0_112
ENV CLASSPATH $JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
ENV PATH $PATH:$JAVA_HOME/bin:$CATALINA_HOME/lib:$CATALINA_HOME/bin

EXPOSE $APPPORT

CMD ["catalina.sh", "run"]


```
### 构建

进入Dockerfile所在目录进行构建

```bash
#-t用于指定镜像名称
#可以a a:0.2等
docker build -t a .
```

### 查看

查看docker镜像

```bash
docker images
```

查看某一个镜像的详细信息

```bash
docker inspect a
```
查看镜像构建历史，以及每一层的大小

```bash
docker history a
```

### 删除镜像

```bash
docker rmi a
#强制删除镜像
docker rmi -f a
```

## Dockerfile指令

### FROM

Dockerfile第一条指令必须是FROM，这个指令指出所构建镜像的基础镜像FORM 后面的内容就是命令`docker pull`后面的镜像书写形式

**AS**在多阶段构建的时候使用

```Dockerfile
FROM docker.li-rui.top/library/centos:7.5.1804
FROM <image> [AS <name>]
FROM <image>[:<tag>] [AS <name>]
```

### LABEL

加入镜像的元数据，是键值对

```Dockerfile
#多行
LABEL "com.example.vendor"="ACME Incorporated"
LABEL com.example.label-with-value="foo"
LABEL version="1.0"
LABEL description="This text illustrates \
that label-values can span multiple lines."

#单行
LABEL multi.label1="value1" \
      multi.label2="value2" \
      other="value3"
```

这个元数据可以用命令`docker inspect a`镜像来查看

### WORKDIR

为Dockerfile内的指令添加工作目录，定义后可以使用相对路径。镜像构建完成后进入容器会自动进入该WORKDIR目录，相当于在Dockerfile中使用了`cd WORKDIR`命令

```Dockerfile
WORKDIR /home
#这里使用的相对目录，相当于RUN mkdir -p `WORKDIR`/{runtime,server}
RUN mkdir -p {runtime,server}
ADD jdk1.8.0_112.tar.gz runtime
```

### RUN

RUN用来执行命令，用来进行各种操作，有两种格式。

RUN会新建docker镜像层，所以要尽量将多个命令一行写完，用于减少镜像体积。

- RUN <command> 在Linux相当于`/bin/sh -c`，在windows中相当于`cmd /S /C`
- RUN ["executable", "param1", "param2"]

```Dockerfile
RUN mkdir -p {runtime,server} && \
    yum install iproute telnet tcpdump mkisofs -y && \
    yum clean all
```

### ADD

ADD指令将文件加入到镜像内用于应用使用，支持直接解压文件到镜像内，docker官方建议使用`COPY`多一点

```Dockerfile
ADD jdk1.8.0_112.tar.gz runtime
```

### COPY   

COPY也可以将文件加入到镜像内用于应用使用，相对于`ADD`没有解压功能。ADD和COPY其他都是一样的，都支持通配符和用户权限

```Dockerfile
COPY [--chown=<user>:<group>] <src>... <dest>
COPY hom* /mydir/ 
COPY hom?.txt /mydir/  
COPY --chown=55:mygroup files* /somedir/
COPY --chown=1 files* /somedir/ 
```
### ENV

用来添加环境变量，这里添加的环境变量有多种作用，Dockerfile中的ENV定义的环境变量为键值对，其中值不可以留空

**ARG也可以设置环境变量，只不过是构建时的环境变量**

- Dockerfile文件中作为变量使用
- 容器运行环境中作为环境变量使用如java环境变量
- ENV所定义的环境变量可以通过命令`docker run --env HCHECK=38087/calf`来覆盖

```Dockerfile
ENV <key> <value>
ENV <key1>=<value1> <key2>=<value2>...
#当有空格时
ENV myName="John Doe" myDog=Rex\ The\ Dog \
    myCat=fluffy

#示例
#定义健康检查的变量
ENV HCHECK test

#下面定义java运行环境变量
ENV CATALINA_HOME /home/server/apache-tomcat-7.0.69 
ENV CATALINA_BASE /home/server/apache-tomcat-7.0.69
ENV JAVA_HOME /home/runtime/jdk1.8.0_112
ENV CLASSPATH $JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar 
ENV PATH $PATH:$JAVA_HOME/bin:$CATALINA_HOME/lib:$CATALINA_HOME/bin 

HEALTHCHECK --interval=5s --timeout=3s \
  #这里$HCHECK当作变量使用，容器运行时通过docker run --env HCHECK=38087/calf来注入变量值
  CMD curl -fs curl -fs 127.0.0.1:$HCHECK || exit 1
```

### EXPOSE

用来声明容器的服务端口和协议，默认协议为TCP，这端口要和镜像内服务的监听端口一致。

这个端口可以让其他容器直接访问，也可以通过命令`docker run -p 宿主机端口:EXPOSE端口`向容器外暴露服务

```Dockerfile
EXPOSE <port> [<port>/<protocol>...]
EXPOSE 80/tcp
EXPOSE 80/udp
EXPOSE 80 443

#对于多端口暴露
docker run -p 80:80/tcp -p 80:80/udp
```

### CMD 

用来执行容器启动时开启容器内服务的指令，特别对于tomcat容器内启动的指令为`catalina.sh run`

CMD有三种格式

- CMD ["executable","param1","param2"] (exec 格式，推荐的格式，这种格式下没有shll环境无法获取环境变量CMD [ "echo", "$HOME" ]不会打出环境变量，容器内进程为1)
- CMD ["param1","param2"] (该格式会作为ENTRYPOINT后面的参数)
- CMD command param1 param2 (shell格式通过`/bin/sh -c`执行，容器1进程为/bin/sh -c) 

```Dockerfile
#exec格式
CMD ["/usr/bin/wc","--help"]
#nginx启动
CMD ["nginx", "-g", "daemon off;"]
#shell格式
CMD echo "This is a test." | wc -
```

`docker run`命令后面的参数会覆盖CMD后面的指令

```Dockerfile
FROM ubuntu
CMD [ "top" ]
```
运行

```bash
#命令ps aux 会覆盖CMD指令 top
 docker run --rm b ps aux
```




### ENTRYPOINT

和CMD作用是一样的，都是用于启动容器内的应用，多条ENTRYPOINT和CMD指令同时存在只会执行最后一条。但是和CMD有点区别

- 同时存在的话CMD会做为ENTRYPOINT的参数<ENTRYPOINT> "<CMD>"
- docker run命令覆盖需要--entrypoint来指定
- docker run <image> -d会将-d传入到ENTRYPOINT参数
- CMD和ENTRYPOINT至少应该有一个在Dockerfile内

#### 执行的两种方式

- ENTRYPOINT ["executable", "param1", "param2"] (exec格式，推荐格式)
- ENTRYPOINT command param1 param2 (shell格式 执行/bin/sh -c)

```Dockerfile
FROM ubuntu
ENTRYPOINT ["top", "-b"]
CMD ["-c"]

#执行启动容器
docker run -it --rm --name test  top -H

FROM ubuntu
ENTRYPOINT exec top -b
docker run -it --rm --name test top
```

#### 场景使用

把镜像作为命令使用，获取当前公网IP

```Dockerfile
FROM ubuntu:16.04
RUN apt-get update \
    && apt-get install -y curl \
    && rm -rf /var/lib/apt/lists/*
ENTRYPOINT [ "curl", "-s", "http://ip.cn" ]

#执行启动命令，直接传入curl参数
docker run myip -i
```

为应用执行做准备工作，redis的启动

ENTRYPOINT脚本会判断传递过来的参数，默认是redis-server，当直接启动时`docker run redis`就会直接使用redis用户运行，我们可以覆盖CMD的参数，那么就会使用root用户运行

```bash
#id就会覆盖
docker run -it redis id
uid=0(root) gid=0(root) groups=0(root)
```

```Dockerfile
FROM alpine:3.4
...
RUN addgroup -S redis && adduser -S -G redis redis
...
ENTRYPOINT ["docker-entrypoint.sh"]

EXPOSE 6379
CMD [ "redis-server" ]
```

docker-entrypoint.sh

```bash
#!/bin/sh
...
# allow the container to be started with `--user`
if [ "$1" = 'redis-server' -a "$(id -u)" = '0' ]; then
    chown -R redis .
    #执行run后面的参数
    exec su-exec redis "$0" "$@"
fi

#直接执行run后面的参数
exec "$@"
```

#### 和CMD以及docker run后面的参数混合

##### exec模式，docker run后面参数加到ENTRYPOINT后面

```Dockerfile
FROM docker.li-rui.top/library/centos:7.5.1804
ENTRYPOINT [ "top", "-b" ]

```

运行docker run --rm test1 -c，容器内执行命令为 top -b -c

##### exec模式，CMD中会提给ENTRYPOINT供默认参数

```Dockerfile
FROM docker.li-rui.top/library/centos:7.5.1804
ENTRYPOINT [ "top", "-b" ]
CMD [ "-c" ]
```

运行docker run --rm test1 容器内执行命令为 top -b -c

运行docker run --rm test1 -n 1 容器内执行命令为 top -b -n 1，-n 1会覆盖CMD指令-c

##### shell模式ENTRYPOINT，CMD 指令会以及docker run image 指令 会被忽略

>可见，容器启动可以传入参数方式有两种：
>- Dockerfile中定义ENV环境变量，通过docker run --env key=value来传递
>- Dockerfile中CMD指令参数，ENTRYPOINT执行脚本预留参数位置，通过docker run image 参数或者指令 来重写CMD参数和传递参数给ENTRYPOINT脚本

详细区别表格

![CMDENTER](https://qiniu.li-rui.top/CMDENTER.png)

### HEALTHCHECK 

为容器内的应用创建健康检查，该指令应该在CMD指令之前

两种执行方式

- HEALTHCHECK [OPTIONS] CMD command 
- HEALTHCHECK NONE (禁用健康检查，在镜像继承中)

接受两个退出状态码

- 0: success - 容器状态healthy
- 1: unhealthy - 容器状态unhealthy

相关选项

- --interval=<间隔>：两次健康检查的间隔，默认为 30 秒；
- --timeout=<时长>：健康检查命令运行超时时间，如果超过这个时间，本次健康检查就被视为失败，默认 30 秒；
- --retries=<次数>：当连续失败指定次数后，则将容器状态视为 unhealthy，默认 3 次。

```Dockerfile
HEALTHCHECK --interval=5s --timeout=3s \
  #这里$HCHECK当作变量使用，容器运行时通过docker run --env HCHECK=38087/calf来注入变量值
  CMD curl -fs curl -fs 127.0.0.1:$HCHECK || exit 1
  #或者tcp，端口开启返回值为0 否则为1
  CMD tcping 127.0.0.1:$HCHECK 
```

状态查看

```bash
docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                 PORTS                      NAMES
5dbd21a8091f        a                   "catalina.sh run"   2 hours ago         Up 2 hours (healthy)   0.0.0.0:38087->38087/tcp   aa

#查看容器详细的运行状态
docker container stats
```

### ONBUILD

处理镜像继承的镜像体积，ONBUILD在当前的Dockerfile中不会执行，会在以当前镜像为基础镜像的Dockerfile中执行

基础镜像

```Dockerfile
FROM node:slim
RUN mkdir /app
WORKDIR /app
ONBUILD COPY ./package.json /app
ONBUILD RUN [ "npm", "install" ]
ONBUILD COPY . /app/
CMD [ "npm", "start" ]
```

子镜像，会执行基础镜像的ONBUILD指令

```Dockerfile
FROM my-node
```

## 多阶段构建

#### go代码示例

主要处理应用的编译环境和运行环境合并，从而直接减小镜像体积，可以允许Dockerfile内多个FROM指令存在

下面以go代码示例

```Dockerfile
#as给编译代码的镜像起个名字
FROM golang:alpine as builder

WORKDIR /go/src
COPY httpserver.go .
#进行编译代码，输出二进制文件httpd到工作目录/go/src/httpd
RUN go build -o httpd ./httpserver.go

#选择运行应用的基础镜像
From alpine:latest

WORKDIR /root/
#关键指令 将编译镜像中编译好的二进制文件 /go/src/httpd copy到当前镜像工作目录/root下
#--from=builder可以指向编译镜像
COPY --from=builder /go/src/httpd .
#给二进制用权限
RUN chmod +x /root/httpd
#执行应用
ENTRYPOINT ["/root/httpd"]
```

### 指定构建阶段

持续构建直到遇到builder镜像就构建停止
```bash
docker build --target builder -t alexellis2/href-counter:latest .
```

### 从其他地方引入

from不只是自己在Dockerfile内构建的镜像，from还可以从本地的镜像和docker源中的镜像，不论是共有和私有

```Dockerfile
#可以是本地或docker hub中的镜像
COPY --from=nginx:latest /etc/nginx/nginx.conf /nginx.conf
#也可以是私有源中的镜像
COPY --from= docker.li-rui.top/pcp/my-nginx:0.1 /etc/nginx/nginx.conf /nginx.conf
```

## 现有基础构建

从当前的一个容器做些操作后构建镜像

```bash
#将主机/www/runoob目录拷贝到容器96f7f14e99ab的/www目录下。
docker cp /www/runoob 96f7f14e99ab:/www/
#将主机/www/runoob目录拷贝到容器96f7f14e99ab中，目录重命名为www。
docker cp /www/runoob 96f7f14e99ab:/www
#将容器96f7f14e99ab的/www目录拷贝到主机的/tmp目录中。
docker cp  96f7f14e99ab:/www /tmp/
```

进入容器后进行各种操作然后执行

```bash
$ docker commit \
    #-a
    --author "Tao Wang <twang2218@gmail.com>" \
    #-m
    --message "修改了默认网页" \
    #这个是容器名称
    webserver \
    #在本地打tag
    #也可以打上私有镜像源tag docker.li-rui.top/pcp/my-nginx:0.1
    nginx:v2
```

## 构建vsftp镜像

开匿名访问，开启本地用户模式

### Dockerfile

```Dockerfile
FROM docker.li-rui.top/library/centos:7.5.1804
MAINTAINER pcp team
LABEL Description="vsftpd Docker image based on Centos 7. Supports passive mode and virtual users." \
        License="Apache License 2.0" \
        Usage="docker run -d -p [HOST PORT NUMBER]:21 -v [HOST FTP HOME]:/home/vsftpd fauria/vsftpd" \
        Version="1.0"

RUN yum -y update && yum clean all
RUN yum install -y \
        vsftpd \
        && yum clean all

ENV FTP_USER **String**
ENV FTP_PASS **Random**
ENV PASV_ADDRESS **IPv4**
ENV PASV_MIN_PORT 21100
ENV PASV_MAX_PORT 21110
ENV LOG_STDOUT **Boolean**

COPY vsftpd.conf /etc/vsftpd/
COPY run-vsftpd.sh /usr/sbin/

RUN chmod +x /usr/sbin/run-vsftpd.sh

EXPOSE 20 21

CMD ["/usr/sbin/run-vsftpd.sh"]

```

### 依赖文件

vsftpd.conf文件

```bash
anonymous_enable=YES
local_enable=YES
write_enable=YES
local_umask=022
dirmessage_enable=YES
xferlog_enable=YES
connect_from_port_20=YES
xferlog_std_format=YES
pam_service_name=vsftpd
userlist_enable=YES
tcp_wrappers=YES

anon_upload_enable=YES
anon_other_write_enable=YES
anon_umask=022
anon_mkdir_write_enable=YES
anon_world_readable_only=no

background=NO
pasv_enable=YES
listen=YES

```

run-vsftpd.sh 脚本

```bash
#!/bin/bash

# If no env var for FTP_USER has been specified, use 'admin':
if [ "$FTP_USER" = "**String**" ]; then
    export FTP_USER='admin'
fi

# If no env var has been specified, generate a random password for FTP_USER:
if [ "$FTP_PASS" = "**Random**" ]; then
    export FTP_PASS=`cat /dev/urandom | tr -dc A-Z-a-z-0-9 | head -c${1:-16}`
fi

# Do not log to STDOUT by default:
if [ "$LOG_STDOUT" = "**Boolean**" ]; then
        export LOG_STDOUT=''
else
        export LOG_STDOUT='Yes.'
fi

# Create home dir and update vsftpd user db:
chmod 757 -R /var/ftp/pub
/usr/sbin/useradd $FTP_USER -s /sbin/nologin
echo -e "$FTP_PASS\n$FTP_PASS" | passwd $FTP_USER
chmod 757 -R /home/$FTP_USER


# Set passive mode parameters:
if [ "$PASV_ADDRESS" = "**IPv4**" ]; then
    export PASV_ADDRESS=$(/sbin/ip route|awk '/default/ { print $3 }')
fi

echo "pasv_address=${PASV_ADDRESS}" >> /etc/vsftpd/vsftpd.conf
echo "pasv_max_port=${PASV_MAX_PORT}" >> /etc/vsftpd/vsftpd.conf
echo "pasv_min_port=${PASV_MIN_PORT}" >> /etc/vsftpd/vsftpd.conf
# Get log file path
export LOG_FILE=`grep xferlog_file /etc/vsftpd/vsftpd.conf|cut -d= -f2`

# stdout server info:
if [ ! $LOG_STDOUT ]; then
cat << EOB
        *************************************************
        *                                               *
        *    Docker image: fauria/vsftd                 *
        *    https://github.com/fauria/docker-vsftpd    *
        *                                               *
        *************************************************

        SERVER SETTINGS
        ---------------
        · FTP User: $FTP_USER
        · FTP Password: $FTP_PASS
        · Log file: $LOG_FILE
        · Redirect vsftpd log to STDOUT: No.
EOB
else
    /usr/bin/ln -sf /dev/stdout $LOG_FILE
fi

# Run vsftpd:
&>/dev/null /usr/sbin/vsftpd /etc/vsftpd/vsftpd.conf

```

### 启动命令

```bash
docker run -d -v /data/iso_tool:/home/iso_tool \
-v /data/test:/var/ftp/pub \
-p 20:20 -p 21:21 -p 21100-21110:21100-21110 \
-e FTP_USER=iso_tool -e FTP_PASS=mypass \
-e PASV_ADDRESS=192.168.6.60 -e PASV_MIN_PORT=21100 -e PASV_MAX_PORT=21110 \
--name aa --restart=always vsftpd:0.1
```

## vnc镜像

```Dockerfile
FROM docker.li-rui.top/library/centos:7.5.1804
MAINTAINER pcp-team@lirui
LABEL author="pcp-team" \
      version="0.3"

WORKDIR /opt

ADD novnc.tar.gz ./
ADD setuptools-1.4.2.tar.gz ./
ADD run-novmc.sh /usr/bin/

RUN mkdir kvm-novnc && \
    yum install iproute telnet tcpdump kde-l10n-Chinese sqlite gcc gcc-c++ python-devel  -y && \
    yum clean all && \
    cd numpy-1.10.4 && python setup.py install && cd .. && \
    cd setuptools-1.4.2 && python2.7 setup.py install && cd .. && \
    cd websockify-master && python setup.py install && cd .. && \
    cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
    cp novnc_service.py kvm-novnc && cp -r novnc kvm-novnc && \
    echo "Asia/shanghai" > /etc/timezone && \
    localedef -c -f UTF-8 -i zh_CN zh_CN.utf8 && \
        chmod +x /usr/bin/run-novmc.sh

ENV LC_ALL "zh_CN.UTF-8"


EXPOSE 6080 9000

ENTRYPOINT ["run-novmc.sh"]
```

脚本run-novmc.sh见下面**单容器多进程**

## 单容器多进程

### 脚本方式

```bash
#!/bin/bash
# Start the first process
nohup websockify --web /opt/kvm-novnc/novnc \
--token-plugin=TokenDB \
--token-source=/opt/kvm-novnc/vnc_tokens.db 6080 >> /opt/kvm-novnc/novnc.log &
status=$?
if [ $status -ne 0 ]; then
  echo "Failed to start websockify: $status" >> /opt/novnc-run.log
  exit $status
fi

# Start the second process
nohup /usr/bin/python /opt/kvm-novnc/novnc_service.py &
status=$?
if [ $status -ne 0 ]; then
  echo "Failed to start novnc_service: $status" >> /opt/novnc-run.log
  exit $status
fi


while sleep 10; do
  ps aux |grep websockify |grep -q -v grep
  PROCESS_1_STATUS=$?
  ps aux |grep novnc_service |grep -q -v grep
  PROCESS_2_STATUS=$?
  if [ $PROCESS_1_STATUS -ne 0 -o $PROCESS_2_STATUS -ne 0 ]; then
    echo "One of the processes has already exited." >> /opt/novnc-run.log
    exit 1
  fi
done
```

### supervisord方式

supervisord配置文件

```bash
[unix_http_server]
file=/tmp/supervisor.sock   


[supervisord]
logfile=/tmp/supervisord.log 
logfile_maxbytes=50MB        
logfile_backups=10           
loglevel=info                
pidfile=/tmp/supervisord.pid 
nodaemon=true               
minfds=1024                  
minprocs=200                 

[supervisorctl]
serverurl=unix:///tmp/supervisor.sock 

[program:xx]
command=/opt/apache-tomcat-8.0.35/bin/catalina.sh run  ; 程序启动命令
autostart=true       
startsecs=10         
startretries=3      
```

启动

```Dockerfile
CMD ["/usr/bin/supervisord"]
```

## 多次构建失败

多次构建同一个Dockerfile出错需要把构建的镜像以及容器删除

镜像的构建过程就是把基础镜像运行容器然后添加层的过程，构建失败需要删除失败构建的镜像以及相关停止的容器。

## 






