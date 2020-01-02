---
title: wireshark远程抓包
date: 2020-01-02 12:12:28
tags:
- linux
---

# 原理

通过ssh向远程主机执行命令返回到本机的标准输出，以此作为Wireshark的标准输入

<!--more-->

# 关键命令

```bash
ssh root@xxxxxxx 'tcpdump -i eth0 -s 0 -l -w -' | wireshark -k -i -
```