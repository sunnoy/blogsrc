---
title: 开启swap
date: 2018-12-18 12:12:28
tags:
- Linux
---

swap可以将一部分硬盘当作内存使用，在某些应用下十分必要。比如hexo中文章多了渲染所需要的内存不够就会被kill掉

<!--more-->

# 文件方式

```bash
#创建4g的交换分区
dd if=/dev/zero of=/swapfile bs=1M count=4096
#权限要给600
chmod 600 /swapfile
#格式化swap分区格式
mkswap /swapfile
#激活分区
swapon /swapfile
#加入/etc/fstab开启启动
echo "LABEL=SWAP-sda /swapfile swap swap defaults 0 0" >> /etc/fstab
```

# 分区方式

```bash
#选择一个分区/dev/sda2
#格式化分区
mkswap /dev/sda2
#激活分区
swapon /dev/sda2
#挂载分区
echo "LABEL=SWAP-sda /dev/sda2 swap swap defaults 0 0" >> /etc/fstab
```

