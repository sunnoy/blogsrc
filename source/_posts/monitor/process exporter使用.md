---
title: process exporter使用
date: 2018-11-26 12:14:28
tags:
- prometheus
- monitor
---

# process exporter使用

process exporter主要用来监控主机中的进程，[项目地址](https://github.com/ncabatoff/process-exporter)

<!--more-->

## 安装

```bash
docker run -d --rm -p 9256:9256 --privileged -v /proc:/host/proc -v `pwd`:/config ncabatoff/process-exporter --procfs /host/proc -config.path /config/filename.yml
```

## 配置

使用配置文件来确定要监控的进程，它提供了很多匹配规则类型

```yml
process_names:
  # 匹配原始执行程序
  - comm: 
    - chromium-browse
    - bash
    - prometheus
    - gvim
  # 匹配执行程序名称
  - exe: 
    - /sbin/password

```