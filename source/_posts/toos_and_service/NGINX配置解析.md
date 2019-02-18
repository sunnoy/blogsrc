---
title: NGINX配置解析
date: 2018-11-04 12:12:28
tags:
- nginx
---

# NGINX配置解析

### 1. 源码安装脚本

```
wget -c http://git.centos8.com/lirui/nor_service/raw/master/nginx/nginx.sh && bash nginx.sh
```
<!--more-->
### 2. NGINX目录及配置
脚本执行完成后，自动创建应用以及配置目录
#### 2.1 应用目录
- **nginx安装位置/usr/local/nginx**
- **启动二进制文件/usr/sbin/nginx**
- **日志文件位置/var/log/nginx**

- **服务启动文件/etc/init.d/nginx**

#### 2.2 配置
- **配置文件目录在 /etc/nginx**
- **主配置文件/etc/nginx/nginx.conf**
- **站点配置文件在/etc/nginx/conf.d/sites**


``` bash
/etc/nginx

├── conf.d
│   ├── cache.conf
│   ├── default.conf
│   ├── proxy.conf
│   ├── sites
│   │   ├── site.conf
│   │   ├── ssl
│   │   ├── upstream.conf
│   │   └── zzidc_kuaiyun
│   └── ssl
├── fastcgi.conf
├── fastcgi.conf.default
├── fastcgi_params
├── fastcgi_params.default
├── koi-utf
├── koi-win
├── mime.types
├── mime.types.default
├── nginx.conf
├── nginx.conf.default
├── scgi_params
├── scgi_params.default
├── uwsgi_params
├── uwsgi_params.default
└── win-utf


```
#### 2.3 配置详解
##### 2.3.1 主配置文件
```bash

#定义nginx运行的用户
user  nginx;
#master进程将任务分配给worker进程，其进程数，默认auto
worker_processes  auto;
#进程文件
pid /var/run/nginx.pid;

#单个nginx进程打开最多文件描述符数目，推荐和ulimit -n 一致
#worker_rlimit_nofile 65535;

events {
    #事件模型，默认使用epoll，内核2.6以上使用
    use epoll;
	#单进程最大连接数=连接数/进程数
    worker_connections  65535;
}


http {

    ##-------------------log_seting-------------------##

    #指定日志记录格式
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for" "$upstream_cache_status" "$body_bytes_sent" "$host"';


    ##-------------------type_seting-------------------##

	#web server依据浏览器请求的静态资源拓展名，
	#从mime.types里面设置响应头的Content-Type，浏览器依据Content-Type处理文件
    include       mime.types;
	#web server默认tpye
    default_type  application/octet-stream;
    #指定文件编码
	charset     utf-8;

	##-------------------client_request_seting-------------------##

	#长连接超时时间，单位是秒
    keepalive_timeout  65;
	#定义HTTP请求体最大字节，上传文件超额返回413错误，设置0禁用检查请求体大小
	client_max_body_size 4096M;
	#定义缓冲区浏览器请求最大字节，越大向内存中写入越多，磁盘中写入减少，性能增强
	client_body_buffer_size 64k;


	##-------------------socket_seting-------------------##

	#开启高效传输模式，高磁盘IO应用设置off
    sendfile       on;
	#仅在sendfile on可以使用，默认开启，数据不立即发出
    tcp_nopush     on;
	#禁用Nagle算法keep-live时开启，数据包直接发送
    tcp_nodelay    on;

	#影响type哈希表冲突概率，默认1024，增大会增大内存消耗，提高检索速度
	types_hash_max_size 4096;
	#站点名称哈希表设置，站点名称较多时增大bucket值32倍数
    server_names_hash_bucket_size  128;
    server_names_hash_max_size 4096;


	#在HTTP响应头中隐藏nginx版本信息
    server_tokens off;

	#开启gzip压缩输出
    gzip on;
	#被压缩的最小响应文件
    gzip_min_length  1k;
	#压缩缓冲区4个16k
    gzip_buffers     4 16k;
	#gzip压缩版本，默认1.1
    gzip_http_version 1.1;
	#压缩等级1-9，越打压缩比越大
    gzip_comp_level 2;
	#指定mime-type类型去压缩，默认text/plain
    gzip_types       text/plain application/x-javascript text/css application/xml;
    #是否在响应头中加入“Vary：Accept-Encoding”
	gzip_vary on;

    ##-------------------proxy_seting-------------------##

	#nginx和后端服务器连接超时时间
	proxy_connect_timeout    600;
	#代理接收，后端服务器响应超时
    proxy_read_timeout       600;
	#代理发送，后端服务器回传超时
    proxy_send_timeout       600;

	#开启后nginx会将后端服务器的响应缓存，而不是立即发送给客户端，默认开启
	#超过缓存区后就会写入磁盘由proxy_max_temp_file_size and proxy_temp_file_write_size
	#proxy_buffering on
	#后端服务器响应头缓冲区
    proxy_buffer_size        16k;
	#最多设置4个64k的缓冲池
    proxy_buffers            4 64k;
	#高负荷下的缓冲池大小，2*proxy_buffers
    proxy_busy_buffers_size 128k;
	#后端服务器响应大小超过缓冲区大小，写入硬盘临时文件，默认1024m
    #proxy_max_temp_file_size 1024m;
	#每次写入临时文件内的数据大小默认8/16k
    proxy_temp_file_write_size 128k;
	#后端服务器响应到磁盘，设置磁盘临时文件目录，最多可以设置3级目录
    proxy_temp_path   /var/tmp/nginx/proxy/proxy_temp;

	##--------------------cache_seting--------------------##
	#对后端服务器其他数据的缓存目录设置，最多3级目录，缓存文件名称运算为md5值，也可以设置
	#所有的缓存key会存储到共享内存空间，空间名字和大小由keys_zone参数定义，1m存储4000个key
	#缓存有效时间inactive默认10分钟，超过max_size就会删除早期的缓存文件
    proxy_cache_path /var/tmp/nginx/proxy/proxy_cache levels=1:2 keys_zone=cache_one:100m inactive=1d max_size=5g;


    ##-------------------include_seting-------------------##

	#包含其他配置文件注意配置文件读取顺序


    include conf.d/sites/*.conf;
    include conf.d/default.conf;


}

```
##### 2.3.2 虚拟主机配置
```bash
server {
    #虚拟主机监听端口
    listen       80;
	listen       443 ssl;

	#主机名称，可接受通配符，正则表达式
    server_name  localhost;

    access_log  /var/log/nginx/ee_access.log  main;
    error_log  /var/log/nginxee_error.log;

##------------------ssl seting-------------##

	#指定ssl证书以及key
    ssl_certificate /etc/nginx/conf.d/ssl/hahh.crt;
    ssl_certificate_key /etc/nginx/conf.d/ssl/hahh.key;
    #ssl缓存会话超时，默认5分钟
	ssl_session_timeout 5m;
	#可以接受的ssl协议版本
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
	#启用ssl加密套件
    ssl_ciphers EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH;
    #和客户端协商加密算法时，优先使用服务端加密套件
	ssl_prefer_server_ciphers on;
    #ssl301跳转
	if ($scheme != "https"){
        return 301 https://$host$request_uri;
    }
    #配置代理
	location / {
        proxy_pass http://tomcat;
        include conf.d/sites/prca/proxy.conf;
        include conf.d/sites/prca/cache.conf;
    }


}

```
##### 2.3.3 cache配置文件

```bash
#定义nginx调用的缓存名称，可以使用变量，默认off位关闭缓存
proxy_cache cache_one;
#添加ATS-Cache到响应头中，后面接格式
add_header ATS-Cache $upstream_cache_status;
#对特定请求方法的响应进行缓存，默认GET HEAD
proxy_cache_methods GET;
#对不同的响应状态码色丁缓存时间
proxy_cache_valid  200 304 301 302 1d;
proxy_cache_valid 404 1m;
#定义一个缓存key名称，下面接受参数
proxy_cache_key $host$uri$is_args$args;
#缓存过期时间
expires 1d;

```
##### 2.3.4 porxy配置文件
```bash
#修改代理服务给浏览器的响应头中地址，替换代理服务器内网IP和非80端口
proxy_redirect    off;
#proxy_set_header更改传送到后端服务器的请求头
#后接 [区域] [值]，值可以是变量文本和其他组合
#Squid 起草X-Forwarded-For协议，经过一次代理服务器增加一个，
#X-Forwarded-For: client, proxy1, proxy2，最后一个代理IP没有
#默认为空，$proxy_add_x_forwarded_for添加用户ip，多一层多一个
#nginx服务器IP
proxy_set_header  X-Forwarded-For  $proxy_add_x_forwarded_for;
#后端服务器获取用户ip而不是nginx的IP
proxy_set_header  X-Real-IP  $remote_addr;
#添加sever中域名，避免请求头中host丢失
proxy_set_header  Host $host;
#将客户端与负载均衡器之间的协议传递给负载均衡器与后端服务器之间
proxy_set_header X-Forwarded-Proto $scheme;

```

##### 2.3.5 upstream配置文件
```bash


#定义一组后端服务器
upstream tomcat {

	#使用ip_hash算法解决session
	ip_hash;
        #使用粘性session
        #查看服务器给客户端的cookie为route=md5，来检测
        sticky;

    server 127.0.0.1:89;
#加入后端检查模块
#添加指令check interval既可打开该模块，后面接毫秒，检查时间间隔
#rise和fall分别表示，检查多少次后成功和失败后判就判定为up和down
#timeout后端检查请求的超时时间单位为毫秒
#default_down=true则认为服务初始就是down，后面经过健康检查之后才能判定up
#type=http，检查数据包的类型，可接tcp/ssl_hello/http/mysql/ajp/port等
#默认值是：interval=30000 fall=5 rise=2 timeout=1000 default_down=true type=tcp

check interval=3000 rise=2 fall=5 timeout=1000 default_down=true type=http [port=check_port];

#HTTP基于TCP，首先建立TCP握手连接然后发送HTTP请求，该指令指定单个连接发送的最大请求数，默认为1，即发送一次HTTP请求后就关闭连接。
check_keepalive_requests 10;

#当check interval中type为http时，该指令指定HTTP请求头的内容，默认为"GET / HTTP/1.0\r\n\r\n"，推荐HEAD可减少数据传输，当请求为长连接时应加上kepp-alive请求头。GET请求数据不可过大一定要保证在1个interval内传输完成，以免后端服务器被判定down。
#请求头中每一行结束需要添加“换行符”\r\n。在请求头和请求体之间需要使用\r\n间隔，加上最后一行请求头中的\r\n因此需要四个\r\n结尾。
check_http_send "HEAD / HTTP/1.0\r\nConnection: keep-alive\r\n\r\n";

#该指令指定HTTP响应状态码为多少则判定后端服务器为up
#默认为http_2xx http_3xx;
check_http_expect_alive http_2xx http_3xx;

#所有后端服务器检查的状态数据将会存放在共享内存(shm,share memory)中，该指令指定这块共享内存的大小，默认为1M，后端服务器较多时应增大该数值。该指令应在主配置文件http块中使用。
check_shm_size 1M;

#返回后端服务器的健康状态，后面可接多种返回格式html|csv|json
#默认为html，需要在location中使用
check_status html
#示例
 location /nstatus {
         check_status;
         access_log off;
         #allow IP;
         #deny all;
       }



  }

```


### 3. 配置调整案例
##### 3.1 后端tomcat非默认应用
假设tomcat应用为site

```bash
    location /site/ {
        proxy_pass http://site;
      }

    location = / {
        return 302 /site/;
    }

```

##### 3.2 后端健康检查与缓存清理，建议配置default内

- default配置文件
- 访问ip:81/s来查看节点健康状态
- 访问ip:81/purge/ss.jpg来清除该文件缓存
- 缓存状态查看在客户端浏览器`ATS-Cache:HIT`
- 缓存状态在服务端查看日志

```bash
"-"| "HIT"| "21630" "192.168.12.105"
"-"| "MISS"| "2351" "192.168.12.105"

```

- 批量清除缓存

```bash
rm -rf /var/tmp/nginx/proxy/proxy_cache/*
```

```bash
server {
    listen       81;
    server_name  null;
    access_log  off;

    root html;

    location / {
            root   html;
            index  index.html index.htm;
    }

    location /s {
            check_status;

        }

    location ~ /purge(/.*) {
       allow all;
       proxy_cache_purge cache_one $host$1$is_args$args;
      #将405 改为200 并定向到/purge$1
       error_page 405 =200 /purge$1;

    }

  }

```

- upstream文件

```bash
upstream www {

    session_sticky;

    server 127.0.0.1:8080;

    check interval=3000 rise=2 fall=5 timeout=1000 type=http;
    check_keepalive_requests 100;
    check_http_send "HEAD / HTTP/1.0\r\nConnection: keep-alive\r\n\r\n";
    check_http_expect_alive http_2xx http_3xx;

    }

```

##### 3.2 代理php

```bash
server {
        listen 80;
        listen [::]:80;

        root /var/;
        index index.php index.html index.htm;

        # Make site accessible from http://localhost/
        server_name null;

        location / {

                try_files $uri $uri/ /index.php?$query_string;

        }
        location ~ \.php$ {
                try_files $uri =404;
                fastcgi_split_path_info ^(.+\.php)(/.+)$;
                fastcgi_pass unix:/var/run/php/php7.2-fpm.sock;
                fastcgi_index index.php;
                fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
                include fastcgi_params;
        }
}

```

##### 3.2 子url通入不同的后端服务器

```bash

location /documents/ {
    192.168.2.30:8800
}
#一定放在最后
location / {
  192.168.2.20:80
}

#后端服务器要配置
192.168.2.30:8800/documents
```

##### 3.3 reload原理

**reload不会中断nginx连接**

![nginx](https://qiniu.li-rui.top/nginx.png)
