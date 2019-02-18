---
title: docker中jdk优化
date: 2018-12-25 12:12:28
tags:
- docker
---

![javadocker](https://qiniu.li-rui.top/javadocker.png)

docker中跑jdk配置不当很容易触发oom，内存超出后被kill掉

# 容器中的资源

我们知道，容器资源隔离基础是Linux中各种namespace对资源的隔离。但是Linux中并不是所有的资源都可以使用namespace来隔离，比如SELinux、time、syslog。namespace的隔离也是不全面的，比如/proc 、/sys 、/dev/sd*没有完全隔离。

也就是说在容器中执行一些抓取上面没有完全隔离的目录时，获取的是宿主机上的信息。比如命令free，free主要读取/proc或/sys目录

<!--more-->

## 读取容器中cgroup资源

docker在1.8版本以后就会将分配给容器的cgroup资源挂载到容器内

```bash
#mount -n
cgroup on /sys/fs/cgroup/cpuset type cgroup (ro,nosuid,nodev,noexec,relatime,cpuset)
cgroup on /sys/fs/cgroup/cpu,cpuacct type cgroup (ro,nosuid,nodev,noexec,relatime,cpuacct,cpu)
cgroup on /sys/fs/cgroup/memory type cgroup (ro,nosuid,nodev,noexec,relatime,memory)
cgroup on /sys/fs/cgroup/devices type cgroup (ro,nosuid,nodev,noexec,relatime,devices)
cgroup on /sys/fs/cgroup/freezer type cgroup (ro,nosuid,nodev,noexec,relatime,freezer)
cgroup on /sys/fs/cgroup/net_cls type cgroup (ro,nosuid,nodev,noexec,relatime,net_cls)
cgroup on /sys/fs/cgroup/blkio type cgroup (ro,nosuid,nodev,noexec,relatime,blkio)
cgroup on /sys/fs/cgroup/perf_event type cgroup (ro,nosuid,nodev,noexec,relatime,perf_event)
cgroup on /sys/fs/cgroup/hugetlb type cgroup (ro,nosuid,nodev,noexec,relatime,hugetlb)
mqueue on /dev/mqueue type mqueue (rw,nosuid,nodev,noexec,relatime,seclabel)

```

在容器中查看一些资源

```bash
#查看容器核数，除100000
cat  /sys/fs/cgroup/cpu/cpu.cfs_quota_us 

#获取内存使用
cat /sys/fs/cgroup/memory/memory.usage_in_bytes 
cat /sys/fs/cgroup/memory/memory.limit_in_bytes 

#查看容器是否设置oom，oom_kill_disable默认为0表示开启
cat /sys/fs/cgroup/memory/memory.oom_control 

#获取磁盘io
cat /sys/fs/cgroup/blkio/blkio.throttle.io_service_bytes

#获取网卡出入流量
cat /sys/class/net/eth0/statistics/rx_bytes 
cat /sys/class/net/eth0/statistics/tx_bytes 
```

# 容器中的应用

一些应用如果读取namespace之外的信息来作为自身获取资源的方式，那么就会造成给容器配置资源限制起不到作用。

比如java，jdk默认使用主机内存的4分之一来作为堆的大小，jdk容器获取的是主机的内存信息，因此即便设定jdk容器的内存限制也不没有作用。

随着jdk的版本迭代，这种情况出现了转机。

## 环境变量设置

JavaSE8(<8u131)，可以使用环境变量方式

```bash
docker run -d -m 800M -e JAVA_OPTIONS='-Xmx400m' openjdk:8-jdk-alpine
```

### 手动指定

JavaSE8(>8u131)以及JavaSE9中内置了相关实验性功能，需要添加一些环境变量开启

```bash
docker run -d -m 800M -e JAVA_OPTIONS='-XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap -XX:MaxRAMFraction=1' openjdk:8-jdk-alpine
```

JavaSE10则默认开启该实验性功能

```bash
docker run -d -m 800M  openjdk:10-jdk-alpine
```

jdk相关版本对docker兼容的具体测试请参考[这篇博文](https://royvanrijn.com/blog/2018/05/java-and-docker-memory-limits/)



