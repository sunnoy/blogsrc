---
title: ntp时间同步服务配置
date: 2018-07-30 12:12:28
tags:
- Linux
---

### ntp服务器搭建

![20140225103215267](https://qiniu.li-rui.top/20140225103215267.jpg)


#### 时间和时区

时间和时区，当地时间，本地时间。

```bash
#/usr/share/zoneinfo放的有时区文件
#切换中国时间
ln -sf /usr/share/zoneinfo/posix/Asia/Shanghai /etc/localtime
```
<!-- more -->
#### 硬件时间和系统时间

- 系统时间
操作系统的kernel所用来计算时间的时钟. 它从1970年1月1日00:00:00 UTC时间到目前为止秒数总和的值

```bash
#设置系统时间
date -s "dd/mm/yyyy hh:mm:ss"  
#查看系统时间
date
#硬件时间设置系统时间
hwclock --hctosys  
#
```

- 硬件时间
硬件时钟是指嵌在主板上的特殊的电路

```bash
#设置硬件时间
hwclock --set --date="mm/dd/yy hh:mm:ss"  
#查看硬件时间
hwclock --show
#系统时间设置硬件时间
hwclock --systohc   
```

**每次开机这两个时间会自动同步**

#### ntp服务开启之前

先同步上游ntp服务器

- 本地比远程快就不同步了
- 本地和远程间隔大，服务启动同步时间长

```bash
ntpdate time1.aliyun.com
# -u 绕过防火墙使用其他端口通信
# -d debug信息
# -s crond 方式使用输出到message
```
### 服务配置

- 配置文件

```bash
vi /etc/ntp.conf
#========权限控制============
#阿里云可以访问
restrict time1.aliyun.com
#国家授时中心可以访问
restrict 133.100.11.8  
#本地可以访问
restrict 127.0.0.1
#该网段可以使用本ntp服务器
#nomodify：客户端不能更改服务端的时间参数，但是客户端可以通过服务端进行网络校时
restrict 192.168.100.0 mask 255.255.255.0 nomodify 

#=========源服务器===========
server time1.aliyun.com prefer 
#本地同步
server 127.127.1.0 
fudge 127.127.1.1 stratum 10
#=========差异分析===========
driftfile /var/lib/ntp/drift
```

- 端口
```bash
iptables -I INPUT -p udp -m udp --dport 123 -j ACCEPT
```

#### 状态监控

```bash
#查看本机和上层服务器的时间同步结果
ntpq –p 
#可以用來追踪某台时间服务器的时间对应关系
ntptrace     
#查看ntp日志
/var/log/ntp/ntp.log  
# 也可以查看一些同步状态
ntpstat      
```

### 客户端配置

#### 安装包

```bash
yum install ntp crontab -y
echo "*/2 * * * * /usr/sbin/ntpdate -s time1.aliyun.com" >>  /var/spool/cron/root
service crond restart
/usr/sbin/ntpdate -s time1.aliyun.com && /sbin/hwclock -w
```




