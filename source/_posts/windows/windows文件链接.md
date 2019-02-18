---
title: windows文件链接
date: 2018-11-04 12:12:28
tags:
- Windows
---
## windows文件链接

### 内存虚拟硬盘

- 工具

```cmd
#创建文件连接
mklink /D "C:\Users\Administrator\AppData\Local\Google\Chrome\User Data\Default\Cache" "Z:\Cache"
```
<!--more-->
### 转发

```bash
 nohup ./brook_linux_386 relay -l 0.0.0.0:1085 -r 203.137.183.99:21028 &

```

