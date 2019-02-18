---
title: tcp和udp端口检查
date: 2018-11-04 12:12:28
tags:
- Linux
---
## tcp和udp端口检查

### tcp

```bash
yum install telnet -y
telnet 127.0.0.3 25
```
<!--more-->
### udp

```bash
yum install nmap -y
nmap -sU -p 3478 192.168.1.25
```