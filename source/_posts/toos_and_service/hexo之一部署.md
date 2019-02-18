---
title: hexo之一部署
date: 2018-7-9 12:14:28
tags:
- hexo
- next

---

![hexologo](https://qiniu.li-rui.top/hexologo.png)

### 1. 环境准备

- nodejs

```bash
#centos
yum install epel-release
yum -y install nodejs npm 
```

- 中国镜像

```bash
#使用cnpm安装
npm install -g cnpm --registry=https://registry.npm.taobao.org
#直接配置
npm config set registry https://registry.npm.taobao.org
```

<!--more-->

### 2. hexo安装

应用安装

```bash
#全局安装
npm install hexo-cli -g
```

初始化博客

```bash
#文件夹内一定要空
hexo init blog

```
插件安装，`--save`为生成package.json内依赖

```bash
#git
npm install hexo-deployer-git --save
#ftp
npm install hexo-deployer-ftpsync --save
#本地搜索
npm install hexo-generator-searchdb --save
```

配置github推送，自定义域名

```
[root@ansible public]# cat CNAME
www.li-rui.top
[root@ansible public]# pwd
/data/blog/public
[root@ansible public]#
```

### 3. next主题安装

```bash
cd themes
wget https://github.com/theme-next/hexo-theme-next.git

```

### 添加标签页面

```bash
hexo new page tags
```

加入属性

```bash
[root@ansible blog]# hexo new page tags
INFO  Created: /data/blog/source/tags/index.md
```
文件/data/blog/source/tags/index.md加入

```yml
---
title: 所有标签
date: 2018-11-15 10:06:40
type: "tags"
comments: false
---
```

### 添加关于页面

```bash
hexo new page about
```
并修改

```yml
---
title: 关于
date: 2018-11-15 10:19:06
comments: false
---

```


### 4. gitpages

- 配置

```bash
vi _config.yml
deploy:
  type: git
  repo: git@github.com:sunnoy/sunnoy.github.io.git
  branch: master
```

- 部署

```bash
#生成静态文件
#hexo generate
hexo g

#部署到GitHub
#hexo deploy
hexo d

deploy:
  type: git
  repo: git@github.com:sunnoy/sunnoy.github.io.git
  branch: master
  message: '站点更新:{{now("YYYY-MM-DD HH:mm:ss")}}'

#部署到ftp
deploy:
  type: ftpsync
  host: **********
  user: webmas*******
  pass: QWZb*********
  remote: /WEB/

```

## hexo镜像

```Dockerfile
FROM node:latest
MAINTAINER lirui

# install hexo
RUN npm install -g cnpm --registry=https://registry.npm.taobao.org && \
cnpm install hexo-cli -g

# init
WORKDIR _init
RUN hexo init && cnpm install

#&& cnpm install hexo-deployer-git --save && hexo-deployer-ftpsync --save

# create data volume
VOLUME /blog
WORKDIR /blog

# hexo default port
EXPOSE 4000

# set entrypoint
COPY entrypoint.sh /root/
RUN chmod +x /root/entrypoint.sh

# run hexo server
ENTRYPOINT ["/root/entrypoint.sh"]

```

entrypoine.sh

```bash
#!/bin/bash

# init the BLOG if it's empty
if [ -z "`ls`" ];then
    echo "directory is empty, init ... "
    cp -ar /_init/* .
fi

# just call hexo directly
hexo $@
```

## 相册

```bash
C:\Users\lirui\AppData\Local\Programs\Python\Python37-32\Lib\site-packages
```

