---
title: yum下载包而不安装
date: 2018-08-1 12:12:28
tags:
- Linux
- yum
---

## 下载包

### centos7

#### 下载包

```bash
yum install --downloadonly --downloaddir=/data/ceph-rpm ceph
```

<!--more-->
#### 查看下载包的数量

```bash
ls -l | grep "^-" | wc -l
```

#### 压缩下载的包

```bash
tar -cvf ceph-rpm.tar.gz ceph-rpm
```

### centos6

centos6需要安装一个插件`yum-plugin-downloadonly`

```bash
 yum install yum-plugin-downloadonly  -y
```
