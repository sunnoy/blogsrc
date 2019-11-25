---
title: ss命令
date: 2019-11-25 12:12:28
tags:
- linux
---

# net-tools与iproute2

netstat命令大家都很熟悉，属于net-tools包，同样net-tools还包含其他的命令如route以及ifconfig等命令。不过最新的Linux发行版已经内置了iproute2包，因为net-tools已经不再维护了

<!--more-->

下面是两个包的对比

![iproute2](https://qiniu.li-rui.top/iproute2.png)

# ss

从上面可以看到以往的netstat命令被ss命令替代，本文主要说明ss的使用

## 基本参数

```bash
-h, --help      帮助信息
-V, --version   程序版本信息
-n, --numeric   不解析服务名称
-r, --resolve   解析主机名
-a, --all       显示所有套接字（sockets）
-l, --listening 显示监听状态的套接字（sockets）
-o, --options   显示计时器信息
-e, --extended  显示详细的套接字（sockets）信息
-m, --memory    显示套接字（socket）的内存使用情况
-p, --processes 显示使用套接字（socket）的进程
-i, --info      显示 TCP内部信息
-s, --summary   显示套接字（socket）使用概况
-4, --ipv4      仅显示IPv4的套接字（sockets）
-6, --ipv6      仅显示IPv6的套接字（sockets）
-0, --packet    显示 PACKET 套接字（socket）
-t, --tcp       仅显示 TCP套接字（sockets）
-u, --udp       仅显示 UCP套接字（sockets）
-d, --dccp      仅显示 DCCP套接字（sockets）
-w, --raw       仅显示 RAW套接字（sockets）
-x, --unix      仅显示 Unix套接字（sockets）
-f, --family=FAMILY  显示 FAMILY类型的套接字（sockets），FAMILY可选，支持  unix, inet, inet6, link, netlink
-A, --query=QUERY, --socket=QUERY
      QUERY := {all|inet|tcp|udp|raw|unix|packet|netlink}[,QUERY]
-D, --diag=FILE     将原始TCP套接字（sockets）信息转储到文件
 -F, --filter=FILE  从文件中都去过滤器信息
       FILTER := [ state TCP-STATE ] [ EXPRESSION ]
```

## tcp状态的显示

```bash
ss -4 state listen
# ss -4 state FILTER-NAME-HERE
# ss -6 state FILTER-NAME-HERE
# FILTER-NAME-HERE 可以代表以下任何一个：
# established、 syn-sent、 syn-recv、 fin-wait-1、 fin-wait-2、 time-wait、 closed、 close-wait、 last-ack、 listen、 closing、
# all : 所有以上状态
# connected : 除了listen and closed的所有状态
# synchronized :所有已连接的状态除了syn-sent
# bucket : 显示状态为maintained as minisockets,如：time-wait和syn-recv.
# big : 和bucket相反.
```

## 端口过滤

```bash
ss -at '( dport = :22 or sport = :22 )'
```