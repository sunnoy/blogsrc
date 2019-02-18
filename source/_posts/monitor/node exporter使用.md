---
title: node exporter使用
date: 2018-11-26 12:14:28
tags:
- prometheus
- monitor
---

# node exporter使用

node exporter主要用来监控主机[项目地址](https://github.com/prometheus/node_exporter)

<!--more-->

## 安装

```bash
docker run -d \
  --net="host" \
  --pid="host" \
  -v "/:/host:ro,rslave" \
  quay.io/prometheus/node-exporter \
  --path.rootfs /host
```

## 选项

重点来看一下相关选项参数

```bash
# 磁盘统计项目忽略的设备
--collector.diskstats.ignored-devices="^(ram|loop|fd|(h|s|v|xv)d[a-z]|nvme\\d+n\\d+p)\\d+$"
# 文件系统项目忽略的挂载点
--collector.filesystem.ignored-mount-points="^/(dev|proc|sys|var/lib/docker/.+)($|/)"
# 文件系统项目中忽略的文件系统类型
--collector.filesystem.ignored-fs-types="^(autofs|binfmt_misc|bpf|cgroup2?|configfs|debugfs|devpts|devtmpfs|fusectl|hugetlbfs|mqueue|nsfs|overlay|proc|procfs|pstore|rpc_pipefs|securityfs|selinuxfs|squashfs|sysfs|tracefs)$"
#网络类别的忽略
--collector.netclass.ignored-devices="^$"
#忽略完了设备类型              
--collector.netdev.ignored-devices="^$"
#忽略的网络字段统计信息
--collector.netstat.fields="^(.*_(InErrors|InErrs)|Ip_Forwarding|Ip(6|Ext)_(InOctets|OutOctets)|Icmp6?_(InMsgs|OutMsgs)|TcpExt_(Listen.*|Syncookies.*)|Tcp_(ActiveOpens|PassiveOpens|RetransSegs|CurrEstab)|Udp6?_(InDatagrams|OutDatagrams|NoPorts))$"
# supervisord 服务配置
--collector.supervisord.url="http://localhost:9001/RPC2"
# systemd 监控白名单                    
--collector.systemd.unit-whitelist=".+"
# systemd 监控黑名单    
--collector.systemd.unit-blacklist=".+\\.scope"
# 指定textfile的路径
--collector.textfile.directory=""
# 执行vmstat命令统计的字段             
--collector.vmstat.fields="^(oom_kill|pgpg|pswp|pg.*fault).*"

```

##　使用实例

```bash
./node_exporter \
--collector.systemd \
--collector.mountstats \
--collector.processes \
--collector.interrupts \
--collector.textfile.directory="/data/textfile_collector" \
--collector.filesystem.ignored-fs-types="^(autofs|binfmt_misc|cgroup|configfs|debugfs|devpts|devtmpfs|fusectl|hugetlbfs|mqueue|overlay|proc|procfs|pstore|rpc_pipefs|securityfs|sysfs|tracefs)$" \
--collector.diskstats.ignored-devices="^(ram|loop|fd|(h|s|v|xv)d[a-z]|nvme\\d+n\\d+p)\\d+$" \
--collector.filesystem.ignored-mount-points="^/(dev|proc|sys|var/lib/docker/.+)($|/)" \
--collector.systemd.unit-whitelist="^ceph.*$"

```

## Linux中统计

### mount stats

```bash
cat /proc/mounts
```

### filesystem

```bash
cat /proc/filesystems
```

## textfile collector

node exporter 支持从主机文件内读取监控信息

首先需要指定读取文件路径

```bash
--collector.textfile.directory="/data/textfile_collector"
```

下面任务是查询给出目录的剩余空间和可用空间

```bash
#!/bin/bash
for i in $@; do
    filesystem=`df $i | awk 'NR==2{print $1}'`
    used=`df $i | awk 'NR==2{print $3}'`
    available=`df $i | awk 'NR==2{print $4}'`
    echo "node_directory_used_size_bytes{filesystem=\"$filesystem\", directory=\"$i\"} $used" >> /data/textfile_collector/directory_size.prom.ss
    echo "node_directory_available_size_bytes{filesystem=\"$filesystem\",directory=\"$i\"} $available" >> /data/textfile_collector/directory_size.prom.ss
done
mv /data/textfile_collector/directory_size.prom.ss /data/textfile_collector/directory_size.prom
```

形成数据为

```bash
# HELP node_directory_available_size_bytes Metric read from /usr/directory_size.prom
# TYPE node_directory_available_size_bytes untyped
node_directory_available_size_bytes{directory="/boot",filesystem="/dev/sda1"} 236432
node_directory_available_size_bytes{directory="/data",filesystem="/dev/mapper/vg_system-lv_root"} 5.4593092e+07
# HELP node_directory_used_size_bytes Metric read from /usr/directory_size.prom
# TYPE node_directory_used_size_bytes untyped
node_directory_used_size_bytes{directory="/boot",filesystem="/dev/sda1"} 272148
node_directory_used_size_bytes{directory="/data",filesystem="/dev/mapper/vg_system-lv_root"} 4.3228532e+07
```

