---
title: Linux使用sshkey认证免密码登录
date: 2018-08-02 12:12:28
tags:
- Linux
---

## ssh认证方式

ssh常见的有两种认证方式，密码认证和sshkey认证

![ssh认证](https://qiniu.li-rui.top/ssh认证.png)

<!--more-->

### 配置sshkey认证

#### 生成key

命令为`ssh-keygen`

```bash
[root@node1 .ssh]# ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
bb:d3:8e:78:8f:57:29:71:72:aa:fe:aa:a4:53:27:01 root@node1
The key's randomart image is:
+--[ RSA 2048]----+
|                 |
|    E            |
|     .           |
|      .   o o    |
|       .S  * .   |
|      o ..o o    |
|     ..o.o o     |
|    .o .++o      |
|    ..o+*B+      |
+-----------------+

```

然后会在出现两个文件id_rsa和id_rsa.pub

```bash
[root@node1 .ssh]# ll
total 16
-rw-------. 1 root root  792 Jul 31 14:29 authorized_keys
-rw-------. 1 root root 1675 Aug  2 12:34 id_rsa
-rw-r--r--. 1 root root  392 Aug  2 12:34 id_rsa.pub
-rw-r--r--. 1 root root  177 Nov 15  2015 known_hosts

```

#### 导入key

```bash
[root@node1 .ssh]# ssh-copy-id 172.17.10.8
The authenticity of host '172.17.10.8 (172.17.10.8)' can't be established.
ECDSA key fingerprint is d2:50:63:f3:82:05:37:6e:17:3e:bc:de:d6:26:16:38.
Are you sure you want to continue connecting (yes/no)? yes
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
root@172.17.10.8's password:

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh '172.17.10.8'"
and check to make sure that only the key(s) you wanted were added.

```

#### authorized_keys

该文件就是执行过`ssh-copy-id`之后，公钥`id_rsa.pub`存放的位置

#### known_hosts

该文件是本地登录其他机器的“指纹”

## 不是默认端口

```bash
ssh -p 1234 user@host, 
ssh-copy-id "-p 1234 user@host" 
scp -P 1234 user@host
```