---
title: redhat6更改为centosyum源
date: 2018-11-04 12:12:28
tags:
- linux
---
#### redhat6更改为centosyum源
- 缘起
Redhat的yum源要订阅才可以使用
- 订阅麻烦死
- so 更改centos yum源

##### 1 卸载Redhat上的yum源

```bash
rpm -aq | grep yum|xargs rpm -e --nodeps
```
<!--more-->
##### 2. 下载新的yum安装包

```bash
wget http://gitlab.gainet.com/lirui/nor_service/raw/master/yum/rpm/yum-metadata-parser-1.1.2-16.el6.x86_64.rpm

wget http://gitlab.gainet.com/lirui/nor_service/raw/master/yum/rpm/yum-3.2.29-30.el6.noarch.rpm

wget http://gitlab.gainet.com/lirui/nor_service/raw/master/yum/rpm/yum-plugin-fastestmirror-1.1.30-40.el6.noarch.rpm
```
##### 3.  安装下载的rpm包
- 不要更改顺序

```bash
rpm -ivh yum-metadata-parser-1.1.2-16.el6.x86_64.rpm yum-3.2.29-30.el6.noarch.rpm yum-plugin-fastestmirror-1.1.30-40.el6.noarch.rpm

```
##### 4. 下载163的repo文件

- 下载到目录/etc/yum.repos.d

```bash
wget http://git.centos8.com/lirui/nor_service/raw/master/yum/repo/CentOS6-Base-163.repo

#163官方源
#7
wget -P /etc/yum.repos.d https://mirrors.163.com/.help/CentOS7-Base-163.repo
#6
wget -P /etc/yum.repos.d https://mirrors.163.com/.help/CentOS6-Base-163.repo
```

- ~~更改repo内的版本号上面链接已经更改为修改后的~~

```bash
sed -i 's#$releasever#7#g' CentOS6-Base-163.repo
```
- 更新一下缓存

```bash
yum clean all

yum makecache
```

- 如果发现修改无效

**清空/etc/yum.repos.d内的所有文件重新下载163源**

- 报错404没有发现

更改主的repo

vi打开

```bash
#打开文件后开始全体替换
vi CentOS6-Base-163.repo
:%s/$releasever/7/g
#更显缓存
yum makecache

```
