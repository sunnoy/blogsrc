---
title: mac django初始化
date: 2019-11-16 12:12:28
tags:
- python
---

# 安装python

```bash
brew install python
```

<!--more-->

# 配置pip中国源

```bash
mkdir ~/.pip
cat .pip/pip.conf

[global]
index-url = https://mirrors.aliyun.com/pypi/simple/
[install]
trusted-host = https://mirrors.aliyun.com/pypi/simple/
```


# 准备虚拟环境

虚拟环境步骤可以省略

```bash
sudo pip3 install virtualenv

# 下面命令会创建venv文件夹
virtualenv --no-site-packages venv

# 进入环境
source venv/bin/activate

# 退出环境
deactivate

# 删除环境
rm -rf venv
```

# 安装mysql5.6

```bash
# 数据库使用5.6
docker run --name mysql -v /data/mysql:/var/lib/mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=test -d mysql:5.6
```

# django mysql准备

## 安装mysql-connector-c

```bash
brew install mysql-connector-c
echo 'export PATH="/usr/local/opt/openssl/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

## mysql-connector-c上mac有bug需要修复

[参考官方文档](https://pypi.org/project/mysqlclient) 

```bash
# 找到mysql_config
which mysql_config

# 获取修改权限
chmod 777 /usr/local/Cellar/mysql-connector-c/6.1.11/bin/mysql_config


vim /usr/local/Cellar/mysql-connector-c/6.1.11/bin/mysql_config

# 如下编辑

# Create options
libs="-L$pkglibdir"
libs="$libs -lmysqlclient -lssl -lcrypto"

# 改回权限
chmod 555 /usr/local/Cellar/mysql-connector-c/6.1.11/bin/mysql_config

# 获取openssl编译变量
brew info openssl
LDFLAGS="-L/usr/local/opt/openssl/lib"
```

## 安装mysqlclient

```bash
sudo LDFLAGS="-L/usr/local/opt/openssl/lib" pip3 install mysqlclient
```


# 安装django以及djangorestframework

```bash
# 2.2.7为长期支持版
sudo pip3 install Django==2.2.7 
sudo pip3 install djangorestframework
sudo pip3 install markdown
sudo pip3 install django-filter
```

# 初始化项目

## 创建项目

```bash
django-admin startproject tutorial
cd tutorial
django-admin startapp quickstart

```

## 配置数据库

### 添加数据库连接信息

在 settings.py 文件中修改

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'django',
        'USER': 'root',
        'PASSWORD': 'test',
        'HOST': 'x.x.x.x',
        'PORT': '3306',
    }
}
```

### 初始化数据库

```bash
python3 manage.py migrate
```

# 启动项目

```bash
python3 manage.py runserver 0.0.0.0:8000
```

![django](https://qiniu.li-rui.top/django.png)