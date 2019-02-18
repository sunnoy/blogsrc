---
title: 关于
date: 2018-11-15 10:19:06
comments: false
---

## 关于博客

博客主要记录工作中的技术点以及工具，还夹杂着一点技术感想。

那么一篇博文如何与您见面的呢？

本地使用vscode写markdown文档，文档中的图片托管到七牛云。完成文档后使用git上传到内网gitlab。

上传完成后push操作会触发gitlab的webhook去向内网测试用服务器发送post请求。

内网测试服务器上由flask写的web服务收到webhook请求后就会执行一个脚本，脚本内容主要是拉取最新文档，然后生成hexo的静态页面，主题使用next主题。

页面生成完成后会同时将静态页面部署到快云的虚拟主机和GitHub的gitpages，通过阿里云dns的线路选择，境外访问会到gitpages，境内访问会到快云虚拟主机。

## 关于作者

大学修过电脑，研究生做过超算，工作后先进行常见WEB容器和常用网络服务的工作，现阶段进行虚拟化相关的工作

## 联系邮箱

<tzhsz.123@gmail.com>

## avenue of poplars in autumn

最后附上副画 

这幅画画于1884年，算是梵高早期的印象派画作，和他后期的画相比在生命力的表达上有将发未发的感觉。

![avenue of poplars in autumn](https://qiniu.li-rui.top/avenue%20of%20poplars%20in%20autumn.jpg)
