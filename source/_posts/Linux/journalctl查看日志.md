---
title: journalctl查看日志
date: 2020-01-13 12:12:28
tags:
- Linux
---

# journalctl介绍

journalctl使用来查看日志的工具，主要是查看systemd的日志

<!--more-->

# 常用命令

## 最新的在前面

```bash
journalctl -r
```

## 自动换行

```bash
journalctl --no-pager
```

## 持续输出

```bash
journalctl -f
```

## 时间过滤

```bash
journalctl --since "2018-08-30 14:10:10" --until "2018-09-02 12:05:50"
```

## 启动日志

```bash
# 列出
journalctl --list-boots

# 指定某一次
journalctl -b -2
```

## 查看某个服务

```bash
journalctl -u ssh
```

## 查看内核

```bash
journalctl -k
```

## 删除日志

```bash
journalctl --vacuum-size=2G
journalctl --vacuum-size=2G
```

## 查看相关状态

```bash
journalctl -u systemd-journald
```

# 配置

```bash
/etc/systemd/journald.conf
```

