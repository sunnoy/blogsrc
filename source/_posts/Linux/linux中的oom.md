---
title: linux中的oom
date: 2019-09-17 12:12:28
tags:
- Linux
---

# oom介绍

程序运行的过程中使用的内存会很大，如果超过了系统的最大内存就会被杀掉

<!--more-->

# 评分机制

Linux对每个进程进行评分，分数高的最先发生oom，下面为评价进程的oom评分

```bash
#!/bin/bash
for proc in $(find /proc -maxdepth 1 -regex '/proc/[0-9]+'); do
    printf "%2d %5d %s\n" \
        "$(cat $proc/oom_score)" \
        "$(basename $proc)" \
        "$(cat $proc/cmdline | tr '\0' ' ' | head -c 50)"
done 2>/dev/null | sort -nr | head -n 10
```

# 发生oom的日志

```bash
dmesg | egrep -i -B100 'killed process'

[5673702.665338] Out of memory: Kill process 29953 (java) score 431 or sacrifice child
[5673702.665338] Killed process 29953, UID 500, (java) total-vm:9805316kB, anon-rss:2344496kB, file-rss:128kB
```

