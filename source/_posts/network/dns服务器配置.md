---
title: dns服务器配置
date: 2018-07-30 12:12:28
tags:
- net
---

### 1. DNS
- 域
![dns_dot](https://qiniu.li-rui.top/dns_dot.gif)
- 查询图
![dns_search](https://qiniu.li-rui.top/dns_search.gif)
- 正解与反解
正解：主机名到IP地址
反解：IP地址到主机名
<!-- more -->
- zone
DNS数据库中每个要解析的`domin`就称为一个`zone`

### 2. DNS服务架设

- 安装包
```bash
yum install bind bind-chroot caching-nameserver –y
#下面命令会安装dns查询工具，nslookup/dig等
yum install bind-utils
```
- 配置文件
- `/etc/named.conf`为主配置文件
```bash
#正解格式resource record, RR
[domain] IN [[RR type] [RR data]]
主机名. IN A IPv4 的IP 地址
主机名. IN AAAA IPv6 的IP 地址
领域名. IN NS 管理这个领域名的服务器主机名字.
领域名. IN SOA 管理这个领域名的七个重要参数(容后说明)
领域名. IN MX 顺序数字接收邮件的服务器主机名字
主机别名. IN CNAME 实际代表这个主机别名的主机名字.
#反解格式.in-addr.arpa
```
- `/etc/sysconfig/named`是否启动chroot等额外参数

### 3. 配置案例
- 规划
假设主域名为`rac.cc`

- 主配置文件配置`/etc/named.conf`
下面为`any`
```bash
listen-on port 53 { any; };
listen-on-v6 port 53 { any; };
allow-query     { any; };

```

- 区域配置文件为`cat /etc/named.rfc1912.zones`
```bash
zone "rac.cc" IN {              
        type master;                  
        file "named.rac.cc"; 
        #这是配置主从服务器使用的
        allow-update { none; };
};
zone "159.168.192.in-addr.arpa" IN {
        type master;
        file "named.192.168.159";
        allow-update { none; };
};
```

- 正解
文件`/var/named/named.rac.cc`内容
SOA两个作用，一是负责与上级域通讯，而是标明在自己域内的众多DNS服务器中自己是主的服务器
NS记录则是指定本域中的所有DNS服务器，但是与父域发生关系的服务器需要SOA来指定。
还需要一条A记录来制定SOA服务器的IP
所以SOA指定的服务器需要有三条记录SOA，NS，A
@表示域本身
```bash
$TTL    86400
#@代表zone，在此地方为rac.cc
@      IN    SOA  ns.rac.cc.  root.rac.cc. (
                                        42              ; serial (d.adams)
                                         3H              ; refresh
                                        15M             ; retry
                                        1W              ; expiry
                                        1D)            ; minimum
#######此处存在省略@，因为和第一条名称一样都是@
#@         IN     NS      ns.rac.cc.
           IN     NS      ns.rac.cc.
ns         IN     A       192.168.159.130

rac-scan   IN     A       192.168.159.30
rac-scan   IN     A       192.168.159.31
rac-scan   IN     A       192.168.159.32

rac1       IN     A       192.168.159.130
rac2       IN     A       192.168.159.131

```

- 反解
文件为`named.192.168.159`
```bash
$TTL    86400
@     IN SOA  ns.rac.cc. root.rac.cc. (
                                      1997022700 ; Serial
                                      28800      ; Refresh
                                      14400      ; Retry
                                      3600000    ; Expire
                                      86400)    ; Minimum

      IN      NS     ns.rac.cc.
130   IN      PTR    ns.rac.cc.

30    IN      PTR    rac-scan.rac.cc.
31    IN      PTR    rac-scan.rac.cc.
32    IN      PTR    rac-scan.rac.cc.

10    IN      PTR    rac1.rac.cc
11    IN      PTR    rac2.rac.cc

```
- 启动服务
```bash
#开机启动
systemctl enable named
#启动服务
systemctl a named
```

### 4.  使用

- 在节点文件`/etc/resolv.conf`
```bash
domain  rac.cc 
nameserver 192.168.159.130  
options rotate  
options timeout:2  
options attempts:5  
```
### 5. 主从
- 主192.168.159.130
```bash
zone "rac.cc" IN {              
        type master;                  
        file "named.rac.cc"; 
        #这是配置主从服务器使用的
        allow-update { 192.168.159.131; };
};
zone "159.168.192.in-addr.arpa" IN {
        type master;
        file "named.192.168.159";
        allow-update { 192.168.159.131; };
};
########正解添加
@          IN     NS      ns.rac.cc.
ns         IN     A       192.168.159.130
#添加从的解析
@          IN     NS      slave.rac.cc.
ns         IN     A       192.168.159.131

########反解添加
@     IN      NS     ns.rac.cc.
130   IN      PTR    ns.rac.cc.

#添加反解
@     IN      NS     slave.rac.cc.
131   IN      PTR    slave.rac.cc.

```
- 从

```bash
zone "rac.cc" IN {              
        type slave;                  
        file "slave/named.rac.cc"; 
        #配置主
        masters { 192.168.159.130; };
};
zone "159.168.192.in-addr.arpa" IN {
        type slave;
        file "slave/named.192.168.159";
        master { 192.168.159.130; };
};
```
### 6. 子域配置
- 父域

```bash
#子dns服务器为132
child.rac.cc         IN     NS      dns.child.rac.cc.
dns.child.rac.cc.    IN     A       192.168.159.132 
```
- 子域
```bash
zone "child.rac.cc" IN {              
        type master;                  
        file "named.child.rac.cc";         
};

####正解
$TTL    86400
#@代表zone，在此地方为rac.cc
@          IN    SOA  dns.child.rac.cc.  root.child.rac.cc. (
                                        42 ; serial (d.adams)
                                         3H ; refresh
                                        15M ; retry
                                        1W  ; expiry
                                        1D) ; minimum

            IN     NS      dns.child.rac.cc.
dns         IN     A       192.168.159.130
```

### 5. windows server配置

- 概念区别

**正向查找区域** 用于把域名转化为ip地址 这个是DNS的主要功能 就是负责诸如吧 www.xxx.com转换为123.123.123.123的ip地址的形式然后返回给客户计算机(这只是个举例哈..不是说真正有一个叫xxx的网站的ip地址是4个123)

**反向查找区域** 用于把ip地址转换为域名(就是正向查找区域的逆向功能) 这个...初初看上去好像没乍的作用,但在使用诸如nslookup的时候其实用的就是反向查找区域

**信任点** 如果你的dns需要跟其他dns或者域配合使用,则dns之间要互相信任(比如a域里的dns能给b域里的计算机使用)则需要再此添加信任

**条件转发器** 可以设置根据某个条件把符合该条件的dns解析转交给另外一台dns服务器 比如把abc.com结尾的转发给dns服务器a解析而把cba.com结尾的转发给dns服务器b解析





