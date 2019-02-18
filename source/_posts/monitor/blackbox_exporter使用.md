---
title: blackbox_exporter使用
date: 2018-11-23 12:14:28
tags:
- prometheus
- monitor
---

# blackbox_exporter使用

blackbox_exporter是用来监控`http, tcp, dns, icmp`协议的工具

## 安装

直接下载二进制或者docker

```bash
#到此处下载二进制文件
https://github.com/prometheus/blackbox_exporter/releases
#docker
docker run -d -p 9115:9115 --name blackbox_exporter -v `pwd`:/config blackbox_exporter --config.file=/config/blackbox.yml
```

<!--more-->

## 使用

```bash
./blackbox_exporter -h
usage: blackbox_exporter [<flags>]

Flags:
  -h, --help                Show context-sensitive help (also try --help-long and --help-man).
      --config.file="blackbox.yml"
                            Blackbox exporter configuration file.
      --web.listen-address=":9115"
                            The address to listen on for HTTP requests.
      --timeout-offset=0.5  Offset to subtract from timeout in seconds.
      --log.level=info      Only log messages with the given severity or above. One of: [debug, info, warn, error]
      --version             Show application version.

```

## 配置

```yml
modules:
  #自定义模块名称
  http_2xx_example:
    #定义检测类型，可选http, tcp, dns, icmp
    prober: <prober_string>
    #定义检测超时
    [ timeout: <duration> ]
    #对不同的协议进行单独配
    [ http: <http_probe> ]
    [ tcp: <tcp_probe> ]
    [ dns: <dns_probe> ]
    [ icmp: <icmp_probe> ]
```

### http_probe

```yml
modules:
  http_2xx_example:
    prober: http
    timeout: 5s
    http:
    # 接收的状态码，
    [ valid_status_codes: <int>, ... | default = 2xx ]
    #配置http版本["HTTP/1.1", "HTTP/2"]
    [ valid_http_versions: <string>, ... ]
    #请求方法
    [ method: <string> | default = "GET" ]
    # The HTTP headers配置
    headers:
        [ <string>: <string> ... ]
    #是否允许重定向
    [ no_follow_redirects: <boolean> | default = false ]
    #发现ssl配置就识别
    [ fail_if_ssl: <boolean> | default = false ]
    # 发现ssl没有配置就失败
    [ fail_if_not_ssl: <boolean> | default = false ]
    # 响应符合正则就失败
    fail_if_matches_regexp:
        [ - <regex>, ... ]
    # 相应失败如果不符合正则表达式
    fail_if_not_matches_regexp:
        [ - <regex>, ... ]
    # 配置ssl
    tls_config:
        [ <tls_config> ]
    # 配置基本认证
    basic_auth:
        [ username: <string> ]
        [ password: <secret> ]
    [ bearer_token: <secret> ]
    [ bearer_token_file: <filename> ]
    #配置http代理
    [ proxy_url: <string> ]
    # 配置IP协议
    [ preferred_ip_protocol: <string> | default = "ip6" ]
    #探针请求体
    body: [ <string> ]

```

### tcp_probe

```yml
modules:
  tcp_example:
    prober: tcp
    timeout: 5s
    tcp:
    #IP协议
    [ preferred_ip_protocol: <string> | default = "ip6" ]
    #
    [ source_ip_address: <string> ]

    #发送请求，期待回复
    query_response:
    [ - [ [ expect: <string> ],
            [ send: <string> ],
            #始终使用ssl
            [ starttls: <boolean | default = false> ]
        ], ...
    ]

    #是否使用ssl
    [ tls: <boolean | default = false> ]

    #启动ssl后进行ssl配置
    tls_config:
    [ <tls_config> ]
```

### dns

```yml
modules:
  dns_example:
    prober: dns
    timeout: 5s
    dns:
    #ip协议
    [ preferred_ip_protocol: <string> | default = "ip6" ]
    #源IP地址
    [ source_ip_address: <string> ]
    #使用tcp或udp
    [ transport_protocol: <string> | default = "udp" ] # udp, tcp
    #请求域名
    query_name: <string>
    #请求类型，可选A SOA CNAME TXT等记录类型
    [ query_type: <string> | default = "ANY" ]
    #验证记录
    valid_rcodes:
    [ - <string> ... | default = "NOERROR" ]
    #Resource Records  rrs 资源记录
    #Answer RRs(资源记录数)
    validate_answer_rrs:
    fail_if_matches_regexp:
        [ - <regex>, ... ]
    fail_if_not_matches_regexp:
        [ - <regex>, ... ]
    #Authority RRs(授权资源记录数)
    validate_authority_rrs:
    fail_if_matches_regexp:
        [ - <regex>, ... ]
    fail_if_not_matches_regexp:
        [ - <regex>, ... ]
    #Additional RRs(额外资源记录数)，
    validate_additional_rrs:
    fail_if_matches_regexp:
        [ - <regex>, ... ]
    fail_if_not_matches_regexp:
        [ - <regex>, ... ]
```

### icmp_probe

```yml
modules:
  icmp_example:
    prober: icmp
    timeout: 5s
    icmp:
    #IP协议
    [ preferred_ip_protocol: <string> | default = "ip6" ]

    #源IP地址
    [ source_ip_address: <string> ]

    # Set the DF-bit in the IP-header. Only works with ip4 and on *nix systems.
    [ dont_fragment: <boolean> | default = false ]

    #发送包大小
    [ payload_size: <int> ]
```

[官方配置实例](https://github.com/prometheus/blackbox_exporter/blob/master/example.yml)

## 配置重载

发送POST请求到`/-/reload`

## Prometheus配置

blackbox exporter需要将`targets`作为参数传入

```yml
scrape_configs:
  - job_name: 'blackboxi_http'
    metrics_path: /probe
    params:
      module: [http_get_2xx]
    static_configs:
      - targets:
         - http://192.168.66.112:3000
         - http://192.168.66.112:9093

    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        #blackbox exporter 所在节点
        replacement: 192.168.66.112:9115
#需要同时使用两个模块
  - job_name: 'blackboxi_icmp'
    metrics_path: /probe
    params:
      module: [icmp]
    static_configs:
      - targets:
         - 192.168.66.112
         - 192.168.6.92
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        #blackbox exporter 所在节点
        replacement: 192.168.66.112:9115

```

## black export

配置

```yml
modules:
  http_get_2xx:
    prober: http
    http:
      valid_http_versions: ["HTTP/1.1", "HTTP/2"]
      method: GET
  icmp:
    prober: icmp
```

启动

```yml
./blackbox_exporter --config.file="blackbox.yml"
```

查看信息，主要查看为`probe_success`，成功为`1`，失败为`0`.

![black](https://qiniu.li-rui.top/black.png)

