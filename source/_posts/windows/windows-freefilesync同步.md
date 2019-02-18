---
title: windows-freefilesync同步
date: 2018-07-30 12:12:28
tags:
- Windows
- freefilesync
- 文件服务

---

## freefilesync-windows

### 软件下载与安装

```bash
https://freefilesync.org/download.php
```
<!-- more -->
### 建立共享文件

在备份的目标机器上建立文件机共享

![同步4](https://qiniu.li-rui.top/同步4.png)

### 数据服务器上加入目标服务器共享

在运行里面

```cmd
\\目标服务器IP
```

### 配置freefielsync

建立初始配置模板

![同步1](https://qiniu.li-rui.top/同步1.png)

保存批处理文件

![同步2](https://qiniu.li-rui.top/同步2.png)

导入批处理文件，并作相关设置

![同步3](https://qiniu.li-rui.top/同步3.png)