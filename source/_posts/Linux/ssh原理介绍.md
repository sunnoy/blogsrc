---
title: ssh原理介绍
tags:
- linux
---

# ssh基本介绍

ssh是一个加密协议，用来加密Linux登陆通信。ssh有很多实现，既有开源也要商业。[openssh](https://www.openssh.com/)是一个使用广泛的ssh协议的开源实现。

<!--more-->

# 加密过程

- 远程主机监听客户端请求，当收到客户端请求后便将自己的公钥发给客户端
- 客户端用收到的公钥对登陆密码加密发给服务端
- 服务端使用自己的公钥解密客户端发送的密码，解密成功后，客户端认证成功

# 密码登陆

客户端首次登陆远程服务器时会有下列提示

```bash
[root@matrix03 ~]# ssh 192.168.6.13
The authenticity of host '192.168.6.13 (192.168.6.13)' can't be established.
ECDSA key fingerprint is MD5:d2:50:63:f3:82:05:37:6e:17:3e:bc:de:d6:26:16:38.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '192.168.6.13' (ECDSA) to the list of known hosts.
root@192.168.6.13's password:
```

这是因为客户端本地`.ssh/known_hosts`没有远程服务端的公钥，该指纹是远程服务端公钥进行md5运算后的值。这个时候客户端就要确定这个指纹到底是不是远程服务端的。因为会存在中间人攻击，即其他远程服务端冒充客户连接的客户端来发送公钥。通过某种方法确定过之后就可以确认，然后会提示输入远程服务端登陆密码。

# 公钥登陆

## 认证过程

公钥登陆可以免去客户端每次登陆输入密码的过程。

用户会将自己的公钥提前放入到远程主机文件`.ssh/authorized_keys`内，登陆过程如下

- 客户端向远程主机发起登陆请求后，远程主机向客户端发送随机字符串
- 客户端将该字符串使用自己的私钥加密传给服务端
- 服务端用客户端提前上传的公钥解密自己发送的字符串

## 使用过程

### 远程主机配置

文件/etc/ssh/sshd_config

```bash
　　RSAAuthentication yes
　　PubkeyAuthentication yes
　　AuthorizedKeysFile .ssh/authorized_keys
```

### 创建公钥

```bash
ssh-keygen

#常用参数
-t type:指定要生成的密钥类型，有rsa1(SSH1),dsa(SSH2),ecdsa(SSH2),rsa(SSH2)等类型，较为常用的是rsa类型
-C comment：提供一个新的注释
-b bits：指定要生成的密钥长度 (单位:bit)，对于RSA类型的密钥，最小长度768bits,默认长度为2048bits。DSA密钥必须是1024bits
```

该命令执行过后默认会在目录`.ssh`下面创建文件，主要私钥权限600

```bash
#私钥
id_rsa  
#公钥
id_rsa.pub

-rw-------  1 root root 1679 Jan 28 14:26 id_rsa
-rw-r--r--  1 root root  395 Jan 28 14:26 id_rsa.pub
```

### 上传公钥

上传公钥其实是将客户端文件`.ssh/id_rsa.pub`上传到服务端`.ssh/authorized_keys`中

命令

```bash
ssh-copy-id user@host

#或者
ssh user@host 'mkdir -p .ssh && cat >> .ssh/authorized_keys' < ~/.ssh/id_rsa.pub
```

### 登陆

使用公钥登陆时ssh默认会找私钥，位置为`.ssh/id_rsa`，如果不是使用默认私钥需要使用下面命令

```bash
ssh -i file user@host
```

要确保私钥文件file权限为600，远程主机上`.ssh/authorized_keys`也应该是600

