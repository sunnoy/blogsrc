---
title: kibana常用操作
date: 2019-08-08 12:12:28
tags:
- efk
---

# kibana介绍

kibana是一个es查询可视化工具，这里介绍常用的操作

<!--more-->

# 查看文档特定字段值

当我们搜索日志的时候，kibana默认返回的是文档的集合。但是我们只关注文档的日志的输出的字段比如message。

可以通过kibana的table功能来完成，点击过后就会在列表顶部出现我们选出的字段的列

![kibanaz字段](https://qiniu.li-rui.top/kibanaz字段.png)

这样我们就可以看到单独的字段的内容了，想要恢复就点击小叉就行了

![tables](https://qiniu.li-rui.top/tables.png)

当然我们还可以添加其他的列
