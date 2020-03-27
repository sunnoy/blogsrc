---
title: docker remote api使用
date: 2018-11-04 12:12:28
tags:
- docker
---

# docker remote api使用

## docker服务端配置

### 配置监听

#### 套接字方式

在Linux下docker服务默认会创建套接字文件`/var/run/docker.sock`来让本地客户端调用，进行操作

#### http方式

docker还支持通过http来调用docker服务

通过启动命令来开启，可以设置多个监听客户端和端口，下面分别让本地监听和让IP192.168.6.6和192.168.1.5使用，配置为`-H tcp://0.0.0.0:2377`让所有客户端都可以访问docker服务

```bash
dockerd -H unix:///var/run/docker.sock -H tcp://192.168.6.6:2376 -H tcp://192.168.1.5:2377
````

 <!--more-->

使用配置文件/etc/docker/daemon.json配置，推荐配置

```json
{
  "hosts": [
    "tcp://0.0.0.0:2375",
    "unix:///var/run/docker.sock"
  ]
}
```

### 安全设置

使用自签证书使用tls来进行安全加固

#### 证书签发

使用openssl工具来生成自签的根证书，服务端证书以及客户端证书

##### CA证书

ca私钥 ca-key.pem
ca证书 ca.pem

```bash
#生成ca私钥，输入密码，这里为xinyue
openssl genrsa -passout pass:xinyue -aes256 -out ca-key.pem 4096
#使用rsa加密方式，-aes256加密，4096长度

#根据ca私钥生成ca证书ca.pem
openssl req -passin pass:xinyue -new -x509 -days 365 -key ca-key.pem -sha256 -out ca.pem -subj /C=CN/ST=henan/L=zhengzhou/O=gainet/OU=pcp/CN=dockerapi.pcp.com
```

##### 服务端证书

server私钥 server-key.pem
server服务端证书请求 server.csr
server证书 server-cert.pem


```bash
#生成服务端私钥
openssl genrsa -out server-key.pem 4096

#根据服务端私钥生成服务端证书请求
openssl req -new -sha256 -key server-key.pem -out server.csr -subj "/CN=dockerapi.pcp.com"

#加入服务端IP
echo subjectAltName = IP:127.0.0.1,DNS:dockerapi.pcp.com > extfile.cnf
#生成服务端证书
openssl x509 -passin pass:xinyue -req -days 3650 -sha256 -in server.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out server-cert.pem -extfile extfile.cnf

```

##### 客户端证书

客户端私钥 key.pem
客户端证书请求 client.csr
客户端证书 cert.pem

```bash
#客户端私钥
openssl genrsa -out key.pem 4096

#根据客户端私钥生成客户端证书请求文件
openssl req -subj '/CN=client' -new -key key.pem -out client.csr

#生成客户端配置文件
echo extendedKeyUsage = clientAuth > extfile.cnf

#使用根证书签发客户端证书
openssl x509 -passin pass:xinyue -req -days 3650 -sha256 -in client.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out cert.pem -extfile extfile.cnf

```

##### 权限配置

```bash
rm -f client.csr server.csr ca.srl extfile.cnf

# 转移目录
mkdir client server
cp {ca,cert,key}.pem client
cp {ca,server-cert,server-key}.pem server
rm {cert,key,server-cert,server-key}.pem

# 设置私钥权限为只读
chmod -f 0400 ca-key.pem server/server-key.pem client/key.pem
```

##### 一键脚本

```bash
#/bin/bash
# @author: anyesu

if [ $# != 1 ] ; then 
echo "USAGE: $0 [HOST_IP]" 
exit 1; 
fi 

#============================================#
#   下面为证书密钥及相关信息配置，注意修改   #
#============================================#
PASSWORD="8#QBD2$!EmED&QxK"
COUNTRY=CN
PROVINCE=yourprovince
CITY=yourcity
ORGANIZATION=yourorganization
GROUP=yourgroup
NAME=yourname
#HOST hostname
HOST=$1
SUBJ="/C=$COUNTRY/ST=$PROVINCE/L=$CITY/O=$ORGANIZATION/OU=$GROUP/CN=$HOST"

echo "your host is: $1"

# 1.生成根证书RSA私钥，PASSWORD作为私钥文件的密码
openssl genrsa -passout pass:$PASSWORD -aes256 -out ca-key.pem 4096

# 2.用根证书RSA私钥生成自签名的根证书
openssl req -passin pass:$PASSWORD -new -x509 -days 3650 -key ca-key.pem -sha256 -out ca.pem -subj $SUBJ

#============================================#
#          用根证书签发server端证书          #
#============================================#

# 3.生成服务端私钥
openssl genrsa -out server-key.pem 4096

# 4.生成服务端证书请求文件
openssl req -new -sha256 -key server-key.pem -out server.csr -subj "/CN=$HOST"

# 5.使tls连接能通过ip地址方式，绑定IP
echo subjectAltName = IP:127.0.0.1,DNS:$HOST > extfile.cnf

# 6.使用根证书签发服务端证书
openssl x509 -passin pass:$PASSWORD -req -days 3650 -sha256 -in server.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out server-cert.pem -extfile extfile.cnf


#============================================#
#          用根证书签发client端证书          #
#============================================#

# 7.生成客户端私钥
openssl genrsa -out key.pem 4096

# 8.生成客户端证书请求文件
openssl req -subj '/CN=client' -new -key key.pem -out client.csr

# 9.客户端证书配置文件
echo extendedKeyUsage = clientAuth > extfile.cnf

# 22.使用根证书签发客户端证书
openssl x509 -passin pass:$PASSWORD -req -days 3650 -sha256 -in client.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out cert.pem -extfile extfile.cnf

#============================================#
#                    清理                    #
#============================================#
# 删除中间文件
rm -f client.csr server.csr ca.srl extfile.cnf

# 转移目录
mkdir client server
cp {ca,cert,key}.pem client
cp {ca,server-cert,server-key}.pem server
rm {cert,key,server-cert,server-key}.pem

# 设置私钥权限为只读
chmod -f 0400 ca-key.pem server/server-key.pem client/key.pem

```

### docker加入证书配置

文件/etc/docker/daemon.json 

```json
{
  "registry-mirrors": ["https://jngj6il2.mirror.aliyuncs.com"],
  "tlsverify": true,
  "tlscert": "/data/ssl/server/server-cert.pem",
  "tlskey": "/data/ssl/server/server-key.pem",
  "tlscacert": "/data/ssl/ca.pem",
  "hosts": [
    "tcp://0.0.0.0:3650",
    "unix:///var/run/docker.sock"
  ]
}
```

重启docker服务

```bash
systemctl daemon-reload
systemctl restart docker
```

## 客户端配置

将客户端证书复制到客户端机器上，此为/data/ssl/client

### 加入解析域名

应为上面我们使用了hostname来作为链接，因此需要加入dns解析

文件/etc/hosts
```bash
echo "192.168.6.60 dockerapi.pcp.com" >> /etc/hosts
```

windows的hosts文件位置`c:\Windows\System32\Drivers\etc\hosts`


### 客户端加入认证

下面均在`/data/ssl/client`目录内进行

#### docker客户端

```bash
docker --tlsverify --tlscacert=ca.pem \
  --tlscert=cert.pem \
  --tlskey=key.pem \
  -H tcp://dockerapi.pcp.com:3650 ps
```

#### curl使用

```bash
curl https://dockerapi.pcp.com:3650/images/json \
  --cert ./cert.pem \
  --key ./key.pem \
  --cacert ./ca.pem
```



