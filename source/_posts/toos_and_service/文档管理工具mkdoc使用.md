---
title: 文档管理工具mkdoc使用
date: 2018-11-04 12:12:28
tags:
- mkdoc
---
## 文档管理工具mkdoc使用

### 安装

安装pip

```bash
pip install mkdocs
```
<!--more-->
### 管理

```bash
#启动内置服务器
mkdocs serve -a 0.0.0.0:8000
```

### docker

```bash
#安装
docker pull squidfunk/mkdocs-material
#启动
docker run --rm -it --name doc -d -p 8000:8000 -v /data/doc_src:/docs squidfunk/mkdocs-material
```

