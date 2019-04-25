---
title: kubernetes中dns查询分析
date: 2019-4-25 12:12:28
tags:
- kubernetes
---

在pod运行的容器大部分都是没有抓包工具的，因此在容器内抓包就显得很难。不过容器的网络namespace都可以在宿主机直接进入，就可以通过在宿主机上进入要抓包容器的网络namespace内进行抓包


<!--more-->

# nsenter

nsenter通过获取容器内的主进程PID来获取容器的各种namespace参数，然后进入不同的namespace

## 安装工具nsenter

通过官方的[GitHub主页](https://github.com/jpetazzo/nsenter)介绍来安装

```bash
docker run --rm -v /usr/local/bin:/target jpetazzo/nsenter
```

## 基本使用

```bash
用法：
 nsenter [options] <program> [<argument>...]

Run a program with namespaces of other processes.

选项：
 -t, --target <pid>     要获取名字空间的目标进程
 -m, --mount[=<file>]   enter mount namespace
 -u, --uts[=<file>]     enter UTS namespace (hostname etc)
 -i, --ipc[=<file>]     enter System V IPC namespace
 -n, --net[=<file>]     enter network namespace
 -p, --pid[=<file>]     enter pid namespace
 -U, --user[=<file>]    enter user namespace
 -S, --setuid <uid>     set uid in entered namespace
 -G, --setgid <gid>     set gid in entered namespace
     --preserve-credentials do not touch uids or gids
 -r, --root[=<dir>]     set the root directory
 -w, --wd[=<dir>]       set the working directory
 -F, --no-fork          执行 <程序> 前不 fork
 -Z, --follow-context   set SELinux context according to --target PID

 -h, --help     显示此帮助并退出
 -V, --version  输出版本信息并退出

```

# 获取容器的id

这里获取coredns容器的ID

```bash
#选择容器
kubectl get pod -n kube-system coredns-66798c86bd-zrvz6 -o wide
NAME                       READY   STATUS    RESTARTS   AGE   IP             NODE         NOMINATED NODE   READINESS GATES
coredns-66798c86bd-zrvz6   1/1     Running   0          21h   172.77.2.118   10.9.1.174   <none>           <none>

#获取详情
kubectl describe pod -n kube-system coredns-66798c86bd-zrvz6

#找到
Containers:
  coredns:
    Container ID:  docker://7bb53d8940764e6daee1d5f4544190f6658e21cb950cf19fa02e4dfe5c03bdf1
#只需要
7bb53d894076
```

# 获取容器内PID

```bash
docker inspect --format {{.State.Pid}} 7bb53d894076
25842
```

# 抓包

## 进入网络namespace

```bash
nsenter -t 25842 -n

#查看网卡
ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN qlen 1
    link/ipip 0.0.0.0 brd 0.0.0.0
4: eth0@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP
    link/ether 62:34:e5:54:f2:21 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.77.2.118/24 scope global eth0
       valid_lft forever preferred_lft forever
```

## 容器内dns解析

### 容器内dns配置

- default namespace

```bash
cat /etc/resolv.conf
nameserver 10.68.0.2
search default.svc.cluster.local. svc.cluster.local. cluster.local.
options ndots:5
```

- sre namespace

```bash
cat /etc/resolv.conf
nameserver 10.68.0.2
search sre.svc.cluster.local. svc.cluster.local. cluster.local.
options ndots:5
```

也可以在pod中更改配置

```yaml
  dnsConfig:
    nameservers:
      - 1.2.3.4
    searches:
      - ns1.svc.cluster.local
      - my.dns.search.suffix
    options:
      - name: ndots
        value: "2"
```

### 解析流程

在容器内执行

```bash
nslookup www.li-rui.top
```

容器内执行的查询流程为

- 找到dns服务器 10.68.0.2
- 依次将域名 www.li-rui.top 带入到 search 后面的多个域，向dns服务器进行查询，没有就继续查询
    - www.li-rui.top.default.svc.cluster.local 先查询A 然后是AAAA
    - www.li-rui.top.svc.cluster.local
    - www.li-rui.top.cluster.local
    - www.li-rui.top 到上游进行递归查询

### 一个问题

我们要查询 www.li-rui.top 为什么有这么多查询，直接查询 www.li-rui.top 不就完了。答案也很简单，为了使用service的时候方便。在同一个namespace内只需要在调用服务的时候直接写上service name就行了

如果要避免多余的查询有两个方法

- 使用绝对域名 比如 www.li-rui.top. 也就是域名后面加个 .
- 调整 ndots 值，默认为5，也就是说非绝对域名内有小于5个 . 就会进行search查找，>= 5 就直接查找域名

## 抓包

```bash
#172.77.2.120为发送dns查询请求的容器IP
tcpdump -vvv -n -i eth0 udp dst port 53 and host 172.77.2.120 -w dns
```

![dns查询](https://qiniu.li-rui.top/dns查询.png)
