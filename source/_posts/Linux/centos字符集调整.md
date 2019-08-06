---
title: centos字符集调整
date: 2019-08-06 12:12:28
tags:
- linux
---

# 语言环境

Linux支持很多的语言环境，不同的语言环境通过不同的编码方式来实现，现在通用的utf8编码

<!--more-->

# localedef

localedef是一个调整语言环境的工具

## 加入中文支持

```bash
localedef -c -f UTF-8 -i zh_CN en_US.utf8
#localedef -c (强制执行) -f 字符集 -i locale定义文件 生成的locale的名称
```

# 配置文件

## 查看当前的字符集

```bash
locale
LANG=en_US.UTF-8
LC_CTYPE="en_US.UTF-8"
LC_NUMERIC="en_US.UTF-8"
LC_TIME="en_US.UTF-8"
LC_COLLATE="en_US.UTF-8"
LC_MONETARY="en_US.UTF-8"
LC_MESSAGES="en_US.UTF-8"
LC_PAPER="en_US.UTF-8"
LC_NAME="en_US.UTF-8"
LC_ADDRESS="en_US.UTF-8"
LC_TELEPHONE="en_US.UTF-8"
LC_MEASUREMENT="en_US.UTF-8"
LC_IDENTIFICATION="en_US.UTF-8"
LC_ALL=
```

里面有几大类

```bash
语言符号及其分类(LC_CTYPE)
数字(LC_NUMERIC)
比较和排序习惯(LC_COLLATE)
时间显示格式(LC_TIME)
货币单位(LC_MONETARY)
信息主要是提示信息,错误信息, 状态信息, 标题, 标签, 按钮和菜单等(LC_MESSAGES)
姓名书写方式(LC_NAME)
地址书写方式(LC_ADDRESS)
电话号码书写方式(LC_TELEPHONE)
度量衡表达方式(LC_MEASUREMENT)
默认纸张尺寸大小(LC_PAPER)
对locale自身包含信息的概述(LC_IDENTIFICATION)。
```

## 查看系统的字符集

```bash
locale -a
#安装中文支持
yum install kde-l10n-Chinese -y
```

## 设定

```bash
vi /etc/locale.conf
##加下面内容到第一行，设置中文
LANG=zh_CN.UTF8
```

# 时区

```bash
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```

