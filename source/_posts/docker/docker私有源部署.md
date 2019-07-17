---
title: docker私有源部署
date: 2018-11-04 12:12:28
tags:
- docker
---
## docker私有源部署

### registry容器方式

#### 镜像拉取

```bash
docker pull registry
```

#### 目录创建

```bash
#创建目录
mkdir -p /data/registry
mkdir -p /data/certs
#创建基本http认证
mkdir -p /data/auth

```

<!--more-->

#### 用户认证创建

```bash
docker run \
  --entrypoint htpasswd \
  registry -Bbn pc pcpasswd > /data/auth/htpasswd

```

#### 启动容器

```bash
#启动容器
docker run -d \
  --restart=always \
  --name registry \
  -v /data/auth:/auth \
  -v /data/certs:/certs \
  -v /data/registry:/var/lib/registry \
  -e "REGISTRY_AUTH=htpasswd" \
  -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
  -e REGISTRY_HTTP_ADDR=0.0.0.0:443 \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/fullchain1.pem \
  -e REGISTRY_HTTP_TLS_KEY=/certs/privkey1.pem \
  -p 443:443 \
  registry
```

### nginx反代registry容器

#### 安装docker-compose

```bash
sudo curl -L "https://github.com/docker/compose/releases/download/1.22.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

chmod +x  /usr/local/bin/docker-compose
```

#### 准备文件夹

```bash
mkdir -p auth registry

├── auth
│   ├── domain.crt
│   ├── domain.key
│   ├── nginx.conf
│   └── nginx.htpasswd
├── docker-compose.yml
└── registry

```

#### 用户添加

```bash
docker run --rm --entrypoint htpasswd registry -Bbn testuser testpassword >> auth/nginx.htpasswd
```

#### nginx配置

```bash
events {
    worker_connections  1024;
}

http {

  upstream docker-registry {
    server registry:5000;
  }

  ## Set a variable to help us decide if we need to add the
  ## 'Docker-Distribution-Api-Version' header.
  ## The registry always sets this header.
  ## In the case of nginx performing auth, the header is unset
  ## since nginx is auth-ing before proxying.
  map $upstream_http_docker_distribution_api_version $docker_distribution_api_version {
    '' 'registry/2.0';
  }

  server {
    listen 443 ssl;
    server_name docker.li-rui.top;

    # SSL
    ssl_certificate /etc/nginx/conf.d/domain.crt;
    ssl_certificate_key /etc/nginx/conf.d/domain.key;

    # Recommendations from https://raymii.org/s/tutorials/Strong_SSL_Security_On_nginx.html
    ssl_protocols TLSv1.1 TLSv1.2;
    ssl_ciphers 'EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH';
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;

    # disable any limits to avoid HTTP 413 for large image uploads
    client_max_body_size 0;

    # required to avoid HTTP 411: see Issue #1486 (https://github.com/moby/moby/issues/1486)
    chunked_transfer_encoding on;

    location /v2/ {
      # Do not allow connections from docker 1.5 and earlier
      # docker pre-1.6.0 did not properly set the user agent on ping, catch "Go *" user agents
      if ($http_user_agent ~ "^(docker\/1\.(3|4|5(?!\.[0-9]-dev))|Go ).*$" ) {
        return 404;
      }

      # To add basic authentication to v2 use auth_basic setting.
      auth_basic "Registry realm";
      auth_basic_user_file /etc/nginx/conf.d/nginx.htpasswd;

      ## If $docker_distribution_api_version is empty, the header is not added.
      ## See the map directive above where this variable is defined.
      add_header 'Docker-Distribution-Api-Version' $docker_distribution_api_version always;

      proxy_pass                          http://docker-registry;
      proxy_set_header  Host              $http_host;   # required for docker client's sake
      proxy_set_header  X-Real-IP         $remote_addr; # pass on real client's IP
      proxy_set_header  X-Forwarded-For   $proxy_add_x_forwarded_for;
      proxy_set_header  X-Forwarded-Proto $scheme;
      proxy_read_timeout                  900;
    }
  }
}
```

#### composer.yml文件

```bash
nginx:
  image: "nginx:alpine"
  ports:
    - 5043:443
  links:
    - registry:registry
  volumes:
    - ./auth:/etc/nginx/conf.d
    - ./auth/nginx.conf:/etc/nginx/nginx.conf:ro

registry:
  image: registry
  ports:
    - 127.0.0.1:5000:5000
  volumes:
    - ./registry:/var/lib/registry
```

#### 启动composer

```bash
#启动
docker-compose up -d
#停止容器
docker-compose stop
#开始容器
docker-compose start
#删除停止的容器
docker-compose rm
#更改配置以后重启服务
docker-compose restart 
```

#### 测试登陆

```bash
#登陆
docker login -u=testuser -p=testpassword docker.li-rui.top:5043
#登出
docker logout docker.li-rui.top:5043
#打标签
docker tag nginx docker.li-rui.top:5043/test
#推送到镜像源
docker push docker.li-rui.top:5043/test
#如果没有认证通过就会提示
no basic auth credentials
#从镜像源拉取
docker pull docker.li-rui.top:5043/test
```

### harbor方式

#### 下载部署离线或者在线脚本

```bash
https://github.com/goharbor/harbor/releases
```

#### 配置harbor.cfg
主要配置

```bash
hostname = docker.li-rui.top
ui_url_protocol = https
customize_crt = off
ssl_cert = /opt/certs/fullchain.cer
ssl_cert_key = /opt/certs/hub.ymq.io.key

#日志文件
/var/log/harbor/

```
#### 私有项目创建

访问域名docker.li-rui.top 默认用户名admin 密码Harbor12345

登陆后可以创建私有项目

![harbor](https://qiniu.li-rui.top/harbor.png)

#### 登陆测试

```bash
#登陆
docker login -u=admin -p=Harbor12345 docker.li-rui.top
#登出
docker logout docker.li-rui.top
#打标签
#镜像源/项目/镜像名称
docker tag nginx docker.li-rui.top/pcp/my-nginx:0.1
#取消tag
docker image remove docker.li-rui.top/pcp/my-nginx:0.1
#推送到镜像源
docker push docker.li-rui.top/pcp/my-nginx:0.1
#如果没有认证通过就会提示
no basic auth credentials
#从镜像源拉取
docker pull docker.li-rui.top/pcp/my-nginx:0.1
```

#### http使用

文件/etc/docker/daemon.json

```json
{
  "registry-mirrors": ["https://jngj6il2.mirror.aliyuncs.com"],
  "insecure-registries" : ["docker.li-rui.top"]
}

```

#### 创建的文件夹

```bash
cd /data
rm -rf ca_download job_logs psc redis registry secretkey 
```

#### 更换配置

```bash
docker-compose down -v
vim harbor.cfg
./prepare
docker-compose up -d

```

# 自签证书

## 签发证书

使用脚本

```bash
https://gist.github.com/sunnoy/0c3dee33fefb302f37a13033a9da3852

bash one-key-ssl.sh <ip or url>
```

harbor.yaml部署证书

```yaml
# https related config
https:
#   # https port for harbor, default is 443
  port: 443
   # The path of cert and key files for nginx
  certificate: /root/ssl-key/server/server-cert.pem
  private_key: /root/ssl-key/server/server-key.pem
```

## docker配置

配置好即可，不用重启docker服务

```bash
#创建文件夹
/etc/docker/certs.d/<ip or url>:<port>

#包含的文件
ca.crt  client.cert  client.key
```

# ssl配置双向认证

## nginx配置

客户端端的认证和服务端认证没有直接关系，也就是说

- 服务端证书和ca，客户端证书ca可以不是同一个ca
- 进行客户端ca创建的时候，CN 的值和服务端的域名 IP不一样也是可以使用的，创建客户端的CN 也是为了方便识别，脚本里面配置一样是为了方便识别
- 进行服务端ca申请的时候 CN 的值也可以和服务端的域名不一样，脚本里面配置一样是为了方便识别
- 进行客户端认证的时候，直接使用别的server端证书也是可以的，也就是说创建客户端证书的时候，echo extendedKeyUsage = clientAuth > extfile.cnf 可以不加，有时也是可以的，这个需要应用层面去支持

```bash
server {

  listen 80;
  listen 443 ssl http2;
  server_name dong.cn;

  #配置服务端的证书和key
  ssl_certificate /data/ssl/server/server-cert.pem;
  ssl_certificate_key /data/ssl/server/server-key.pem;


  #配置客户端证书 ca 文件
  ssl_client_certificate /data/ssl/server/ca.pem;
  #开启客户端认证

  #其他的可选参数
  ssl_verify_client on;
  ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
  ssl_prefer_server_ciphers on;
  ssl_ciphers 'EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH';
  ssl_ecdh_curve secp384r1;
  ssl_session_timeout  10m;
  ssl_session_cache shared:SSL:10m;
  ssl_session_tickets off;
  ssl_stapling on;
  ssl_stapling_verify on;
  resolver 223.5.5.5 114.114.114.114 valid=300s;
  resolver_timeout 5s;


  # 强制http跳转为https
  if ($scheme != "https") {
    return 301 https://$http_host$request_uri;
  }
    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

}
```

## 客户端需要

### 导入服务端ca

导入服务端ca用于认证服务端

### 导入客户端证书

格式转换,mac可以读取

```bash
cd /data/ssl/client
openssl pkcs12 -export -clcerts -in ./cert.pem -inkey ./key.pem -out ./client.p12
```

## 客户端测试

没有导入客户端证书，浏览器报400错误

导入后会让选择自己导入的客户端证书，然后输入自己的mac的认证信息，可以浏览了

# gcr镜像

微软中国提供

[地址](http://mirror.azure.cn/help/gcr-proxy-cache.html)

## 使用

### gcr.io

```bash
# gcr
docker pull gcr.io/spinnaker-marketplace/halyard:1.21.1
# 镜像
docker pull gcr.azk8s.cn/spinnaker-marketplace/halyard:1.21.1

```

### k8s.gcr.io

```bash

#k8s.gcr.io
docker pull k8s.gcr.io/xxx:yyy
#镜像
docker pull gcr.azk8s.cn/google-containers/xxx:yyy

```

### quay.io

```bash

#quay.io
docker pull quay.io/xxx/yyy:zzz
#镜像
docker pull quay.azk8s.cn/xxx/yyy:zzz
```

### dockerhub

```bash
# docker hub
docker pull sunnoy/certbot:710
# 镜像
docker pull dockerhub.azk8s.cn/sunnoy/certbot:710
```

[具体使用](https://www.ilanni.com/?p=14534)












