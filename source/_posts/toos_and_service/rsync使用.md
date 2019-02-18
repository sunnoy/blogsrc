---
title: rsync使用
date: 2018-11-04 12:12:28
tags:
- linux
---
## rsync使用

### 1. 同步到远程主机

同步到远程主机，并添加定时任务

```bash
45 2 * * * /usr/bin/rsync -avzu --progress -e 'ssh -p 23'  /www/backup/*.dmp 1.192.218.185:/home/backup >/www/backup/sync.log 2>&1

45 2 * * * /usr/bin/rsync -avzu --progress -e 'ssh -p 23'  /master/backup/* 1.192.218.185:/home/backup/datarzzl >> /www/backup/sync.log 2>&1

0 22 * * * /usr/bin/rsync -avzP --timeout=20 --bwlimit=2048 --stats -e 'ssh -p 23' /data/* 1.192.218.185:/rzzl/rzzlbackup/ >> /opt/sync.log 2>&1
0 7 * * * /usr/bin/pkill rsync

```
<!--more-->
### 2. 相关参数

```bash
#rsync  限速100KB/s
  rsync -avzP --stats --dry-run --bwlimit=100 本地文件 远程文件

#  v：详细提示 
#  a：以archive模式操作，复制目录、符号连接
#　z：压缩
#  P：是综合了--partial --progress两个参数，
#  stats：显示同步情况，有哪些文件会同步
#  dry-run：测试同步，有哪些文件会同步，但是不实际运行

```

### 3. windows系统使用

```bash
cwrsync
```

## lsync

实时同步增加

```bash
sysctl fs.inotify.max_user_watches=200000
```

### 本地之间同步

```bash
settings {

    logfile = "/var/log/lsyncd/lsyncd.log",
    statusFile = "/var/log/lsyncd/lsyncd.status",
    inotifyMode = "CloseWrite",
    maxProcesses = 8,

    }

sync {
    default.rsync,
    source = "/data1",
    target = "/data",
    delete = false,
    rsync = {
        binary = "/usr/bin/rsync",
        archive = true,
        compress = true

        }
    }

```

### 异地之间同步

```bash
settings {
   logfile    = "/tmp/lsyncd.log",
   statusFile = "/tmp/lsyncd.status",
   statusInterval = 1,
   inotifyMode = CloseWrite,
}

sync {
   default.rsyncssh,
   source="{{ src_dir }}",
   host="{{ tar_host }}",
   targetdir="{{ tar_dir }}",
   delete=false
   rsync = {
     archive = true,
     compress = true,
     whole_file = false,
     _extra    = {"--bwlimit={{ bw_limit }}"}
   },
   ssh = {
     port = {{ ssh_port }}
   }
}
```