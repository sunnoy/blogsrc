---
title: metricbeat-system-module使用
date: 2018-11-07 12:12:28
tags:
- elk
- metricbeat
---

# metricbeat-system-module使用

system module主要用来采集系统的基本信息，包含cpu、磁盘、文件系统等方面的metricsets，本文主要关注磁盘方面的内容

<!--more-->

## system默认metricsets

### 开启控制台输出

编辑文件metricbeat.yml将输出调整为控制台

```yml
#================================ Outputs =====================================


#-------------------------- Elasticsearch output ------------------------------

output.console:
  pretty: true
```

### 仅使用system

为了方便调试，减少输出干扰仅仅开启system模块，模块的使用官方[文档](https://www.elastic.co/guide/en/beats/metricbeat/6.3/metricbeat-module-system.html#_metricsets_29)

查看已经启用的模块

```bash
cd go/src/github.com/elastic/beats/metricbeat
#查看模块，保证只有system启用
 ./metricbeat modules list
Enabled:
system
```
通过下面命令来启用或者禁用模块

```bash
metricbeat modules enable system
metricbeat modules disable system
```

## metricsets查看

### 禁用所有metricsets

我们把system中metricsets都取消掉，查看默认输出

#### 模块配置

进入单个模块配置路径go/src/github.com/elastic/beats/metricbeat/modules.d，在这个目录内模块启用其拓展名就是`yml`结尾，如果模块禁用就是以`disabled`结尾。

```bash
aerospike.yml.disabled  system.yml
```
#### system中metricsets配置

接着把system.yml内metricsets都取消掉，文件system.yml，period为采集间隔时间

```yml
# Module: system
# Docs: https://www.elastic.co/guide/en/beats/metricbeat/master/metricbeat-module-system.html

- module: system
  period: 100s
  metricsets:
#    - cpu
#    - load
#    - memory
#    - network
#    - process
#    - process_summary
#    - core
#    - diskio
#    - socket
```

#### 启动metricbeat

启动metricbeat来查看输出，控制台会输出system模块采集的所有metricsets信息

```bash
./metricbeat -c system.yml run
```

接下来查看关于磁盘的几个metricsets

### diskio

采集系统磁盘的读写信息和读写等待时间

#### 配置信息system.yml

```yml
# Module: system
# Docs: https://www.elastic.co/guide/en/beats/metricbeat/master/metricbeat-module-system.html

- module: system
  period: 2s
  metricsets:
    - diskio
  #还可以指定监控特定磁盘的读写
  diskio.include_devices: ["vda1"]
```

#### 查看返回

```json
{
  "@timestamp": "2018-11-07T02:13:06.306Z",
  "@metadata": {
    "beat": "metricbeat",
    "type": "doc",
    "version": "7.0.0-alpha1"
  },
  "metricset": {
    "module": "system",
    "rtt": 1458,
    "name": "diskio"
  },
  "system": {
    "diskio": {
      "write": {
        "count": 2051,
        "time": 383,
        "bytes": 2098176
      },
      "io": {
        "time": 662
      },
      "iostat": {
        "write": {
          "request": {
            "merges_per_sec": 0,
            "per_sec": 0
          },
          "per_sec": {
            "bytes": 0
          }
        },
        "queue": {
          "avg_size": 0
        },
        "request": {
          "avg_size": 0
        },
        "await": 0,
        "service_time": 0,
        "busy": 0,
        "read": {
          "request": {
            "merges_per_sec": 0,
            "per_sec": 0
          },
          "per_sec": {
            "bytes": 0
          }
        }
      },
      "name": "vda1",
      "read": {
        "count": 135,
        "time": 344,
        "bytes": 6480384
      }
    }
  },
  "beat": {
    "name": "localhost.localdomain",
    "hostname": "localhost.localdomain",
    "version": "7.0.0-alpha1"
  },
  "host": {
    "name": "localhost.localdomain"
  }
}
```

### filesystem

查看文件系统的挂载点和挂载容量

#### 配置信息

`filesystem.ignore_types`没有配置该字段会忽略所有虚拟设备，就是在/proc/filesystems中标记为nodev的设备都不会被统计

```yml
metricbeat.modules:
  - module: system
    period: 30s
    metricsets: ["filesystem"]
    #排除的文件系统在/proc/filesystems列表
    #filesystem.ignore_types: [nfs, smbfs, autofs]
```

#### filesystem.ignore_types

上面的配置中没有启用`filesystem.ignore_types`，首先看本机的文件系统

```bash
cat /proc/filesystems
nodev   sysfs
nodev   rootfs
nodev   ramfs
nodev   bdev
nodev   proc
nodev   cgroup
nodev   cpuset
nodev   tmpfs
nodev   devtmpfs
nodev   debugfs
nodev   securityfs
nodev   sockfs
nodev   dax
nodev   pipefs
nodev   anon_inodefs
nodev   configfs
nodev   devpts
nodev   hugetlbfs
nodev   autofs
nodev   pstore
nodev   mqueue
        xfs
nodev   rpc_pipefs
```

可见只有xfs不是虚拟文件系统，那么统计的时候只会统计xfs的文件系统的信息，是根据挂载点来分的

配置为下面是采集所有的/proc/filesystems内的文件系统

```yml
# Module: system
# Docs: https://www.elastic.co/guide/en/beats/metricbeat/master/metricbeat-module-system.html

- module: system
  period: 2000s
  metricsets:
    - filesystem
  filesystem.ignore_types: [ ]
```

下面为排除pstore、autofs的采集数据

```yml
# Module: system
# Docs: https://www.elastic.co/guide/en/beats/metricbeat/master/metricbeat-module-system.html

- module: system
  period: 2000s
  metricsets:
    - filesystem
  filesystem.ignore_types: [pstore, autofs]
```


排除模板

```yml
filesystem.ignore_types: [sysfs,rootfs,ramfs,bdev,proc,cpuset,tmpfs,devtmpfs,debugfs,securityfs,sockfs,dax,pipefs,anon_inodefs,configfs,devpts,hugetlbfs,autofs,pstore,mqueue,xfs,cgroup,rpc_pipefs,binfmt_misc]
```

#### 返回信息

大小单位为bytes，换算为GB需要/(1024*3)

```json
{
  "@timestamp": "2018-11-07T02:51:55.748Z",
  "@metadata": {
    "beat": "metricbeat",
    "type": "doc",
    "version": "7.0.0-alpha1"
  },
  "beat": {
    "hostname": "localhost.localdomain",
    "version": "7.0.0-alpha1",
    "name": "localhost.localdomain"
  },
  "host": {
    "name": "localhost.localdomain"
  },
  "system": {
    "filesystem": {
      "device_name": "/dev/mapper/centos-root",
      "type": "xfs",
      "available": 24947838976,
      "mount_point": "/",
      "free": 24947838976,
      "files": 40370176,
      "free_files": 40157132,
      "total": 41328574464,
      "used": {
        "bytes": 16380735488,
        "pct": 0.3964
      }
    }
  },
  "metricset": {
    "name": "filesystem",
    "module": "system",
    "rtt": 508
  }
}
{
  "@timestamp": "2018-11-07T02:51:55.748Z",
  "@metadata": {
    "beat": "metricbeat",
    "type": "doc",
    "version": "7.0.0-alpha1"
  },
  "system": {
    "filesystem": {
      "device_name": "/dev/vda1",
      "free": 317898752,
      "total": 520794112,
      "available": 317898752,
      "type": "xfs",
      "free_files": 511659,
      "mount_point": "/boot",
      "used": {
        "bytes": 202895360,
        "pct": 0.3896
      },
      "files": 512000
    }
  },
  "metricset": {
    "name": "filesystem",
    "module": "system",
    "rtt": 550
  },
  "beat": {
    "name": "localhost.localdomain",
    "hostname": "localhost.localdomain",
    "version": "7.0.0-alpha1"
  },
  "host": {
    "name": "localhost.localdomain"
  }
}
```

可见采集出了两个挂载点`/`和`/boot`以及相应的挂载设备

查看本机的挂载情况

```bash
lsblk
NAME            MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
vda             252:0    0   40G  0 disk
├─vda1          252:1    0  500M  0 part /boot
├─vda2          252:2    0  9.5G  0 part
│ ├─centos-root 253:0    0 38.5G  0 lvm  /
│ └─centos-swap 253:1    0    1G  0 lvm  [SWAP]
└─vda3          252:3    0   30G  0 part
  └─centos-root 253:0    0 38.5G  0 lvm  /
```

### fsstat

用来对文件系统进行统计，是该文件系统的总量统计

#### 配置

`filesystem.ignore_types`没有配置该字段会忽略所有虚拟设备，就是在/proc/filesystems中标记为nodev的设备都不会被统计

```yml
metricbeat.modules:
  - module: system
    period: 30s
    metricsets: ["fsstat"]
    #排除的文件系统在/proc/filesystems列表
    #filesystem.ignore_types: [nfs, smbfs, autofs]
```


#### 返回

```json
{
  "@timestamp": "2018-11-07T05:51:40.335Z",
  "@metadata": {
    "beat": "metricbeat",
    "type": "doc",
    "version": "7.0.0-alpha1"
  },
  "beat": {
    "hostname": "localhost.localdomain",
    "version": "7.0.0-alpha1",
    "name": "localhost.localdomain"
  },
  "host": {
    "name": "localhost.localdomain"
  },
  "metricset": {
    "name": "fsstat",
    "module": "system",
    "rtt": 636
  },
  "system": {
    "fsstat": {
      "count": 2,
      "total_files": 40882176,
      "total_size": {
        "free": 25173913600,
        "used": 16675454976,
        "total": 41849368576
      }
    }
  }
}
```


