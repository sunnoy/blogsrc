---
title: drf初始化
date: 2019-11-16 12:12:28
tags:
- python
---

# drf介绍

Django REST framework is a powerful and flexible toolkit for building Web APIs

详细介绍请参考

```bash
https://zhuanlan.zhihu.com/p/53957464
```

<!--more-->

# 虚拟环境准备

```bash
python3 -m venv drf
source drf/bin/activate

pip3 install django
pip3 install djangorestframework
# 代码高亮
pip3 install pygments  
```

# 创建工程

## 初始化项目

```bash
django-admin startproject tutorial
cd tutorial
python manage.py startapp snippets
```

## 添加app

让django去加载我们的app

```python
//tutorial/settings.py

INSTALLED_APPS = [
    ...
    'rest_framework',
    'snippets.apps.SnippetsConfig',
]
```