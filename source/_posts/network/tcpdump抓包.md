---
title: tcpdump抓包
date: 2018-07-30 12:12:28
tags:
- net
---
## tcpdump抓包

### 安装

```bash
yum install tcpdump -y
```

<!--more-->

### 使用

```bash
#tcp和主机
tcpdump tcp port 23 and host 222.27.48.1
#网卡
tcpdump -i eth0
#输出文件使用wirshack分析
tcpdump -w data
```