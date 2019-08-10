---
title: fluent-bit使用
date: 2019-8-7 12:12:28
tags:
- efk
---

# fluent-bit介绍

相对于fluentd，fluent-bit占用资源比较少

![fluentdvs](https://qiniu.li-rui.top/fluentdvs.png)

<!--more-->

# 启动

```bash
docker run  fluent/fluent-bit 
```

# 日志流

![fluentbitlog](https://qiniu.li-rui.top/fluentbitlog.png)

- input 输入部分，通过相关插件来完成
- Parser 将原始日志进行解析出元数据
- Filter 将日志进行过滤使用
- Buffer 日志默认会在内存中暂存
- Routing 进行一些路由的处理，路由处理通过tag和Match来完成路由操作
- Output 将日志传递出去


# 配置

fluent-bit的全部配置需要在下面四个部分中配置

- Service 全局配置
- Input
- Filter
- Output
- PARSER

## 引入配置

可以进行配置文件引入

```ini
@INCLUDE input_*.conf
```

## 变量

使用系统变量

```ini
[OUTPUT]
    Name  ${MY_OUTPUT}
    Match *
```

定义变量

```ini
@SET my_input=cpu
@SET my_output=stdout

[INPUT]
    Name ${my_input}
```

## 监控

```ini
[SERVICE]
    HTTP_Server  On
    HTTP_Listen  0.0.0.0
    HTTP_PORT    2020
#curl -s http://127.0.0.1:2020/api/v1/metrics/prometheus
```

## 配置文件位置

```bash
/fluent-bit/etc/fluent-bit.conf
```

# tail

我们常使用的是从原有的文件中读取日志，因此tail input插件这里这种介绍一下

## 相关配置

### 配置示例

```ini
[INPUT]
    Name        tail
    Path        /var/log/syslog

[OUTPUT]
    Name   stdout
    Match  *
```

### 详细参数

- Name 插件name，这里是tail
- Buffer_Chunk_Size chunk的大小 默认 32k
- Buffer_Max_Size 对于每个监测的文件的最大的buffer大小
- Path 读取文件路径，可以使用通配符
- Path_Key 定义一个字符串，导出的日志文档中会是:

```json
{"live": "/data/c.log"}
```

- Exclude_Path 排除一些不搜集的文件如 exclude_path=*.gz,*.zip
- Refresh_Interval 刷新采集文件的间隔秒数 默认 60
- Rotate_Wait 每个发送数据周期后的发送数据间隔秒数 默认 5
- Ignore_Older 忽略多久之前的文件 4m 4h 4d
- Skip_Long_Lines 如果条日志很大，超过了Buffer_Max_Size，那么是否忽略他，继续下面的内容，默认off
- DB 记录监测文件的位置，使用的是SQLite  /var/log/flb_kube.db 
- Mem_Buf_Limit 设定内存buffer上限
- Parser 指定解析方式
- Key 原始日志的jsonkey，默认是log
- Tag 配置标签 kube.<namespace_name>.<pod_name>.<container_name>
- Tag_Regex 通过正则来获取字段 

```bash
(?<pod_name>[a-z0-9]([-a-z0-9]*[a-z0-9])?(\.[a-z0-9]([-a-z0-9]*[a-z0-9])?)*)_(?<namespace_name>[^_]+)_(?<container_name>.+)-
```

## Multiline

### 参数

- Multiline 是否开启多行处理
- Multiline_Flush 多行队列数据处理间隔秒数 默认 4
- Parser_Firstline 日志解析，必选参数

Parser_Firstline 需要使用PARSER部分定义的解析器

# 解析文件

## 使用解析器

解析配置代码块不可以写在fluent-bit.conf配置文件里面，需要写在另外的文件内，SERVICE中的Parsers_File来引入

主配置文件：

```ini
#fluent-bit.conf
[SERVICE]
    Parsers_File parser_docker.conf
[INPUT]
    Name             tail
    Path             java.log
    # Multiline on 的时候，使用Parser_Firstline
    Multiline        on
    Parser_Firstline docker1
    Parser_Firstline docker2
    Parser_Firstline docker3
    # Multiline off (默认情况)使用
    Parser
```
解析配置文件：

```ini
#parser_docker.conf
[PARSER]
    Name        docker
    Format      json
    Time_Key    time
    Time_Format %Y-%m-%dT%H:%M:%S %z
```

## 默认解析器

- Apache
- Nginx
- Docker
- Syslog rfc5424
- Syslog rfc3164

# 解析ngixn中access日志

## 日志格式

```bash
127.0.0.1 50384 - [10/Aug/2019:08:34:54 +0000] "GET / HTTP/1.1" 73 403 159 "-" "curl/7.29.0" "-" conn = 1 conn_req=1 request_time = 0.000 upstream_connect_time=- upstream_header_time=- upstream_response_time = - upstream_addr=- upstream_status=-
```

## fluent bit配置

```ini
    [SERVICE]
        Parsers_File parser-nginx.conf
    [PARSER]
        Name        ddnginx
        Format      regex
        Regex       ^(?<remote_addr>[^ ]*) (?<remote_port>[^ ]*) - \[(?<logtime>[^\]]*)\] "(?<method>\S+)(?: +(?<path>[^\"]*?)(?: +\S*)?)?" (?<request_length>[^ ]*) (?<status>[^ ]*) (?<body_bytes_sent>[^ ]*) "(?<http_referer>[^\"]*)" "(?<http_user_agent>[^\"]*)" "(?<http_x_forwarded_for>[^\"]*)" conn = (?<conn>[^ ]*) conn_req=(?<conn_req>[^ ]*) request_time = (?<request_time>[^ ]*) upstream_connect_time=(?<upstream_connect_time>[^ ]*) upstream_header_time=(?<upstream_header_time>[^ ]*) upstream_response_time = (?<upstream_connect_time>[^ ]*) upstream_addr=(?<upstream_addr>[^ ]*) upstream_status=(?<upstream_status>[^ ]*)$
        Time_Key    logtime
        Time_Format %d/%m/%YT%H:%M:%S %z
    [INPUT]
        Name        tail
        Path        /opt/access.log
        Refresh_Interval 2
        DB          /opt/access.db
        Parser      ddnginx
        Tag         nginx.access
```

## 正则解读

```bash
^(?<remote_addr>[^ ]*)

^ 字符串开头

(?<remote_addr>正则语句) 用来捕获正则语句匹配的字符串，给变量remote_addr

[^ ] 里面有个空格，除了不是空格都匹配，[^"] 除了不是"都匹配

* 可以是任意个字符串
```

## 正则表达式验证

- http://fluentular.herokuapp.com/
- https://rubular.com/

