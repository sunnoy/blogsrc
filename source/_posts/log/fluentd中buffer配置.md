---
title: fluentd中buffer配置
date: 2018-11-29 12:12:28
tags:
- efk
---

# 为何需要buffer

fluentd向es进行日志传送的过程中，如果es挂了，日志就会被丢弃。常见的做法是在fluentd和es之间加一个kafka作为buffer。

问题虽然解决了，但是日志量不大的时候使用kafka未免有些大材小用。fluentd本身就提供了buffer，可以在一定程度上替代kafka

<!--more-->

# fluentd日志流向

![fluentd-v0.14-plugin-api-overview](https://qiniu.li-rui.top/fluentd-v0.14-plugin-api-overview.png)

这里只关注出口，从图上可以看出，日志发送有两中方式：

- 直接同步发送
- 通过buffer发送

## 通过buffer发送

整个buffer机制将日志传送分成了两个阶段，stage和queue。既然是buffer，多条日志会累计到一定的数量再发往目的地，这一定数量的日志就是chunk。这里就提现两个阶段的用意了：

- stage 在chunk还没有填满的的时候，一旦chunk被填满就会送往queue阶段
- queue 移交至queue阶段的chunk并不会立即发送目的地，因为有目的地挂掉的情况

既然目的地存在挂掉的情况，本来送往目的地的日志还在queue里面，因此就有重试发往目的地的机制。

# 文件和内存

buffer的存储后端模式有文件和内存两种，很显然文件没有内存存储效率快，内存没有文件存储体积大。这里仅讨论文件情况的配置。

值得注意的是fluentd官方不并不推荐使用远程文件系统如nfs等。

# buffer配置参数

buffer字段必须被包含在match字段里面

## path

指定buffer文件的位置

```xml
  <buffer>
    @type file
    path /var/log/fluent/foo
  </buffer>
```

## chunk key

chunk key就是buffer文件的命名规范

```xml
<buffer ARGUMENT_CHUNK_KEYS>
  # ...
</buffer>
```

可选值

```xml
  <buffer tag>
    # ...
  </buffer>

  <buffer time>
    timekey      1h # chunks per hours ("3600" also available)
    timekey_wait 5m # 5mins delay for flush ("300" also available)
  </buffer>  

```

### buffer key为time时

- timekey 必须要配置该值，chunk 间隔
- timekey_wait 当下一个timekey到的时候，再推迟timekey_wait事件写入，默认为600s
- timekey_use_utc 是否使用utc时间，默认 false
- timekey_zone 使用的时区，默认为系统时区，格式`-0700 or Asia/Tokyo`


# chunk相关调整

- chunk_limit_size 每个chunk的最大值 默认: 8MB (memory) / 256MB (file)
- chunk_limit_records  每个chunk中的最大日志条数 无默认值
- total_limit_size buffer能够存储的最大日志大小，默认：512MB (memory) / 64GB (file)
- chunk_full_threshold output插件在total_limit_size*chunk_full_threshold大小的时候就会向目的地发送日志
- queued_chunks_limit_size 队列中chunk数量的最大值，默认为1
- compress 是否压缩节省空间，默认为text不压缩，可选gzip

# flush相关调整

flush就是将queue中的chunk发往到目的地，主要是调整一些延迟和吞吐量

- flush_at_shutdown 在fulentd关掉的时候将所有的缓存都发送完，默认：false for file，true for mem
- flush_interval 发送间隔，默认为60s 
- flush_mode 发送模式，默认为interval，如果chunk为time的时候就是lazy
    - lazy 每timekey周期发送一次
    - interval 每个flush_interval发送一次 
    - immediate 立即发送
- flush_thread_count 发送线程数量，默认：1
- flush_thread_interval 当没有chunk需要发送的时候，线程到下一次重试发送的间隔，默认 1.0s
- flush_thread_burst_interval 持续发送chunk的过程中下次发送的间隔，默认 1.0s
- delayed_commit_timeout 该参数不是很理解
- overflow_action buffer queue满了以后，应该怎么做
    - throw_exception 抛出异常，输出错误日志，默认选项
    - block 阻止input插件向buffer发送日志
    - drop_oldest_chunk 丢弃老得chunk

# 重试机制调整

output插件发送到目的地的重试机制，当目的地不可用的时候

- retry_timeout 最大重试超时，默认 72h
- retry_max_times 重试最大次数
- retry_forever 是否永远重试，默认false，设置为true时会忽略retry_max_times和retry_timeout
- retry_secondary_threshold 重试retry_timeout*retry_secondary_threshold时间间隔后使用备用存储，默认为0.8
- retry_type 重试间隔类型
    - exponential_backoff 指数级间隔时间增长，retry_exponential_backoff_base来确定起始间隔，默认为2
    - periodic 恒定的间隔重试，间隔时间通过retry_wait来配置
- retry_max_interval 对于指数级重试类型，最大的间隔秒数，无默认值
- retry_randomize 是否启动随机重试
- disable_chunk_backup 是否备份没有送到的chunk，默认为false



