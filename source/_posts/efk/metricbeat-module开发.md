---
title: metricbeat-module开发
date: 2018-11-05 12:12:28
tags:
- elk
- module
---

# metricbeat-module开发

module主要由metricsets组成，创建新的模块仍然需要命令

```bash
make create-metricset
```

该指令执行后会在_meta目录内创建一些文件，在更改_meta目录内的文件后需要执行命令，该命令会调用beats目录下的Makefile文件中的指令

```bash
cd go/src/github.com/elastic/beats
make update
```
<!-- more -->
## module文件

### config.yml and config.reference.yml

这个两个文件展示了该模块的基本配置以及配置参考，该文件内配置出检查间隔以及是否启动模块，以及模块下所启用的metricset和对所启用的metricset的精细化配置

### dosc.asciidoc

这个为该模块的文档

### fields.yml

该文件是该模块的所有返回字段，用于生成Elasticsearch模板和文档





