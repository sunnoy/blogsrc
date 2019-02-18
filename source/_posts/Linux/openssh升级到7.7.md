---
title: openssh升级到7.7
date: 2018-07-30 12:12:28
tags:
- ssh
---

### 1. 安装telnet

为了防止配置ssh出问题

- 安装

```bash
yum install telnet telnet-server -y
```

- 允许root登录

```bash
vi /etc/xinetd.d/telnet

service telnet
{
        flags           = REUSE
        socket_type     = stream
        wait            = no
        user            = root
        server          = /usr/sbin/in.telnetd
        log_on_failure  += USERID
        disable         = no     #将语句 disable = yes 改成 disable = no 保存退出。激活 telnet 服务
}
```
<!-- more -->
- 启动

```bash
/etc/init.d/xinetd start
```

### 2. 自带openssh卸载

```bash
systemctl stop sshd 或者service sshd stop


rpm -qa | grep openssh
rpm -e openssh --nodeps

rpm -e openssh-server --nodeps

rpm -e openssh-clients --nodeps
#验证
rpm -qa | grep openssh
```


### 3. 依赖安装

- 编译环境

```bash
yum -y install gcc make zlib-devel openssl-devel
```

- 依赖包

```bash
  mkdir /usr/local/sshtools
  wget http://www.zlib.net/zlib-1.2.11.tar.gz

  wget https://www.openssl.org/source/openssl-1.0.2o.tar.gz

  wget https://ftp.openbsd.org/pub/OpenBSD/OpenSSH/portable/openssh-7.7p1.tar.gz

  tar -xvf zlib-1.2.11.tar.gz
  tar -xvf openssl-1.0.2o.tar.gz
  tar -xvf openssh-7.7p1.tar.gz
```

### 4. 依赖包编译

```bash

cd zlib-1.2.11
./configure --prefix=/usr/local/zlib

make -j2&&make install

vi /etc/ld.so.conf.d/zlib.conf
/usr/local/zlib/lib
ldconfig -v


cd openssl-1.0.2o
./config shared zlib
make -j2

make test

make install

mv /usr/bin/openssl /usr/bin/openssl.bak
mv /usr/include/openssl /usr/include/openssl.bak

ln -s /usr/local/ssl/bin/openssl /usr/bin/openssl

ln -s /usr/local/ssl/include/openssl /usr/include/openssl

vi /etc/ld.so.conf.d/ssl.conf
/usr/local/ssl/lib
ldconfig -v
```

### 5. 编译配置openssh7.7

```bash
#编译
mv /etc/ssh /etc/ssh.bak
cd openssh-7.7p1
./configure --prefix=/usr/local/openssh --sysconfdir=/etc/ssh --with-ssl-dir=/usr/local/ssl --mandir=/usr/share/man --with-zlib=/usr/local/zlib

#配置
cp contrib/redhat/sshd.init /etc/init.d/sshd
chmod u+x /etc/init.d/sshd
chkconfig --add sshd
chkconfig --list | grep sshd

cp sshd_config /etc/ssh/sshd_config

#允许root登录以及sftp相关
vi /etc/ssh/sshd_config

Subsystem sftp /usr/local/openssh/libexec/sftp-server
PasswordAuthentication yes
PermitRootLogin yes

#复制二进制文件到环境变量目录
cp /usr/local/openssh/sbin/sshd /usr/sbin/sshd
cp /usr/local/openssh/bin/ssh /usr/bin/
cp /usr/local/openssh/bin/ssh-keygen /usr/bin/ssh-keygen

#启动服务
service sshd restart
```

### 6. 安全相关

```bash
vi /etc/ssh/sshd_config

X11Forwarding no
Ciphers aes128-ctr,aes192-ctr,aes256-ctr,aes128-cbc,3des-cbc
KexAlgorithms curve25519-sha256@libssh.org,diffie-hellman-group-exchange-sha256

```

### 7. 远程主机端生成key

在远程主机端生成key需要,对自己生成公钥注册

```bash
cp id_rsa.pub authorized_keys
```

### 8. 提示key不存在，无法启动

```bash
sudo ssh-keygen -t rsa -b 2048 -f /etc/ssh/ssh_host_rsa_key
sudo ssh-keygen -t dsa -f /etc/ssh/ssh_host_dsa_key
sudo ssh-keygen -t ecdsa -f /etc/ssh/ssh_host_ecdsa_key
sudo ssh-keygen -t ecdsa -f /etc/ssh/ssh_host_ed25519_key
```
