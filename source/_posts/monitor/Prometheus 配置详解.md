---
title: Prometheus 配置详解
date: 2018-11-12 12:14:28
tags:
- prometheus
- monitor
---

# Prometheus 配置详解

Prometheus配置方式有两种

- 命令行，用来配置不可变命令参数，主要是Prometheus运行参数，比如数据存储位置
- 配置文件，用来配置Prometheus应用参数，比如数据采集，报警对接

不重启进程配置生效方式也有两种

- 对进程发送信号`SIGHUP`
- HTTP POST请求，需要开启`--web.enable-lifecycle`选项`curl -X POST http://192.168.66.112:9091/-/reload`

<!--more-->

## 命令行

命令行可用配置可通过`prometheus -h`来查看

```bash

  -h, --help                     Show context-sensitive help (also try --help-long and --help-man).
      --version                  Show application version.
      --config.file="prometheus.yml"
                                 Prometheus configuration file path.
      --web.listen-address="0.0.0.0:9090"
                                 Address to listen on for UI, API, and telemetry.
      --web.read-timeout=5m      Maximum duration before timing out read of the request, and closing idle
                                 connections.
      --web.max-connections=512  Maximum number of simultaneous connections.
      --web.route-prefix=<path>  Prefix for the internal routes of web endpoints. Defaults to path of
                                 --web.external-url.
      --web.user-assets=<path>   Path to static asset directory, available at /user.
      --web.enable-lifecycle     Enable shutdown and reload via HTTP request.

```

## 配置文件

配置文件使用`yml`格式，配置文件中一级配置项

```yml
＃全局配置
global:
＃规则配置主要是配置报警规则
rule_files:
＃抓取配置，主要配置抓取客户端相关
scrape_configs:
＃报警配置
alerting:
＃用于远程存储写配置
remote_write:
＃用于远程读配置
remote_read:
```

配置文件中通用字段值格式

- `<boolean>`: 布尔类型值为`true`和`false`
- `<scheme>`: 协议方式包含`http`和`https`

### global字段

#### scrape_interval

全局默认的数据拉取间隔

```yml
[ scrape_interval: <duration> | default = 1m ]
```

#### scrape_timeout

全局默认的单次数据拉取超时，当报`context deadline exceeded`错误时需要在特定的job下配置该字段

```yml
[ scrape_timeout: <duration> | default = 10s ]
```

#### evaluation_interval

全局默认的规则(主要是报警规则)拉取间隔

```yml
[ evaluation_interval: <duration> | default = 1m ]
```

#### external_labels

该服务端在与其他系统对接所携带的标签

```yml
[ <labelname>: <labelvalue> ... ]
```

### rule_files

这里可以引用两种规则

- `recording rules`用来优化查询
- `alerting rules`用来处理报警

这里只介绍recording rules

#### 规则分组rule_group

不论是`recording rules`还是`alerting rules`都要在组里面

```yml
groups:
  #groups的名称
  - name: example
    #该组下的规则
    rules:
      [ - <rule> ... ]
```

#### 定义Recording rules

有一些监控的数据查询时很耗时的，还有一些数据查询所使用的查询语句很繁琐。Recording rules可以把一些很耗时的查询或者很繁琐的查询进行提前查询好，然后在需要数据的时候就可以很快拉出数据。

```yml
# 指出规则类型record 后面接名称
record: <string>

# 写入PromQL表达式查询语句
#expr: sum(http_inprogress_requests) by (job)
expr: <string>

# 在存储数据之前加上标签
labels:
  [ <labelname>: <labelvalue> ]
```

示例

```yml
groups:
  - name: example
    rules:
    - record: job:http_inprogress_requests:sum
      expr: sum(http_inprogress_requests) by (job)
```

#### 规则检查

```bash
#打镜像后使用
FROM golang:1.10

RUN GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go get -u github.com/prometheus/prometheus/cmd/promtool

FROM alpine:latest  

COPY --from=0 /go/bin/promtool /bin
ENTRYPOINT ["/bin/promtool"]  

# 编译
docker build -t promtool:0.1 .
#使用
docker run --rm -v /root/test/prom:/opt promtool:0.1 check rules /opt/rule.yml
#返回
Checking /opt/rule.yml
  SUCCESS: 1 rules found

```

### scrape_configs字段

拉取数据配置，在配置字段内可以配置拉取数据的对象(Targets)，job以及实例

#### job_name

定义job名称，是一个拉取单元。每个job_name都会自动引入默认配置如

- scrape_interval 依赖全局配置
- scrape_timeout 依赖全局配置
- metrics_path 默认为'/metrics'
- scheme 默认为'http'

这些也可以在单独的job中自定义

```yml
[ scrape_interval: <duration> | default = <global_config.scrape_interval> ]
[ scrape_timeout: <duration> | default = <global_config.scrape_timeout> ]
[ metrics_path: <path> | default = /metrics ]
```

#### honor_labels

服务端拉取过来的数据也会存在标签，配置文件中也会有标签，这样就可能发生冲突

- true就是以抓取数据中的标签为准
- false就会重新命名抓取数据中的标签为“exported”形式，然后添加配置文件中的标签

```yml
[ honor_labels: <boolean> | default = false ]
```

#### scheme

切换抓取数据所用的协议

```yml
[ scheme: <scheme> | default = http ]
```

#### params

定义可选的url参数

```yml
[ <string>: [<string>, ...] ]
```

#### 抓取认证类

每次抓取数据请求的认证信息

##### basic_auth

password和password_file互斥只可以选择其一

```yml
basic_auth:
  [ username: <string> ]
  [ password: <secret> ]
  [ password_file: <string> ]
```

##### bearer_token

bearer_token和bearer_token_file互斥只可以选择其一

```yml
[ bearer_token: <secret> ]
[ bearer_token_file: /path/to/bearer/token/file ]
```

#### tls_config

抓取ssl请求时证书配置

```yml
tls_config:
  [ ca_file: <filename> ]
  [ cert_file: <filename> ]
  [ key_file: <filename> ]
  [ server_name: <string> ]
  #禁用证书验证
  [ insecure_skip_verify: <boolean> ]
```

#### proxy_url

通过代理去主去数据

```yml
[ proxy_url: <string> ]
```

#### 服务发现类

Prometheus支持多种服务现工具，详细配置这里不再展开

```yml
#sd就是service discovery的缩写
azure_sd_configs: 
consul_sd_configs:
dns_sd_configs:
ec2_sd_configs:
openstack_sd_configs:
file_sd_configs:
gce_sd_configs:
kubernetes_sd_configs:
marathon_sd_configs:
nerve_sd_configs:
serverset_sd_configs:
triton_sd_configs:
```

#### static_configs

服务发现来获取抓取目标为动态配置，这个配置项目为静态配置，静态配置为典型的targets配置，在改配置字段可以直接添加标签

```yml
- targets:
    [ - '<host>' ]
  labels:
    [ <labelname>: <labelvalue> ... ]
```

采集器所采集的数据都会带有`label`，当使用服务发现时，比如`consul`所携带的label如下

```yml
__meta_consul_address: consul地址
__meta_consul_dc: consul中服务所在的数据中心
__meta_consul_metadata_: 服务的metadata
__meta_consul_node: 服务所在consul节点的信息
__meta_consul_service_address: 服务访问地址
__meta_consul_service_id: 服务ID
__meta_consul_service_port: 服务端口
__meta_consul_service: 服务名称
__meta_consul_tags: 服务包含的标签信息
```

这些lable是数据筛选与聚合计算的基础

#### 数据过滤类

抓取数据很繁杂，尤其是通过服务发现添加的target。所以过滤就显得尤为重要，我们知道抓取数据就是抓取target的一些列metrics，Prometheus过滤是通过对标签操作操现的，在字段`relabel_configs`和`metric_relabel_configs`里面配置，两者的配置都需要`relabel_config`字段。该字段需要配置项如下

```yaml
[ source_labels: '[' <labelname> [, ...] ']' ]

[ separator: <string> | default = ; ]

[ target_label: <labelname> ]

[ regex: <regex> | default = (.*) ]

[ modulus: <uint64> ]

[ replacement: <string> | default = $1 ]

#action除了默认动作还有keep、drop、hashmod、labelmap、labeldrop、labelkeep
[ action: <relabel_action> | default = replace ]
```

##### target配置示例

```yml
relabel_configs:
  - source_labels: [job]
    regex:         (.*)some-[regex]
    action:        drop
  - source_labels: [__address__]
    modulus:       8
    target_label:  __tmp_hash
    action:        hashmod
```

##### target中metric示例

```yml
- job_name: cadvisor
  ...
  metric_relabel_configs:
  - source_labels: [id]
    regex: '/system.slice/var-lib-docker-containers.*-shm.mount'
    action: drop
  - source_labels: [container_label_JenkinsId]
    regex: '.+'
    action: drop
```

##### 使用示例

由以上可知当使用服务发现consul会带入标签`__meta_consul_dc`，现在为了表示方便需要将该标签变为`dc`

需要做如下配置，这里面action使用的`replacement`

```yml
scrape_configs:
  - job_name: consul_sd
    relabel_configs:
    - source_labels:  ["__meta_consul_dc"]
      regex: "(.*)"
      replacement: $1
      action: replace
      target_label: "dc"

#或者
- source_labels:  ["__meta_consul_dc"]
  target_label: "dc"
```

过滤采集target

```yml
relabel_configs:
- source_labels: ["__meta_consul_tags"]
  regex: ".*,development,.*"
  action: keep
```

#### sample_limit

为了防止Prometheus服务过载，使用该字段限制经过relabel之后的数据采集数量，超过该数字拉取的数据就会被忽略

```yml
[ sample_limit: <int> | default = 0 ]
```

### alerting 字段

该字段配置与Alertmanager进行对接的配置

#### alert_relabel_configs

此项配置和`scrape_configs`字段中`relabel_configs`配置一样，用于对需要报警的数据进行过滤后发向`Alertmanager`

#### alertmanagers

该项目主要用来配置不同的`alertmanagers`服务，以及Prometheus服务和他们的链接参数。`alertmanagers`服务可以静态配置也可以使用服务发现配置。Prometheus以pushing 的方式向alertmanager传递数据。

alertmanager 服务配置和target配置一样，可用字段如下

```yml
[ timeout: <duration> | default = 10s ]
[ path_prefix: <path> | default = / ]
[ scheme: <scheme> | default = http ]
basic_auth:
  [ username: <string> ]
  [ password: <string> ]
  [ password_file: <string> ]
[ bearer_token: <string> ]
[ bearer_token_file: /path/to/bearer/token/file ]
tls_config:
  [ <tls_config> ]
[ proxy_url: <string> ]
azure_sd_configs:
  [ - <azure_sd_config> ... ]
consul_sd_configs:
  [ - <consul_sd_config> ... ]
dns_sd_configs:
  [ - <dns_sd_config> ... ]
ec2_sd_configs:
  [ - <ec2_sd_config> ... ]
file_sd_configs:
  [ - <file_sd_config> ... ]
gce_sd_configs:
  [ - <gce_sd_config> ... ]
kubernetes_sd_configs:
  [ - <kubernetes_sd_config> ... ]
marathon_sd_configs:
  [ - <marathon_sd_config> ... ]
nerve_sd_configs:
  [ - <nerve_sd_config> ... ]
serverset_sd_configs:
  [ - <serverset_sd_config> ... ]
triton_sd_configs:
  [ - <triton_sd_config> ... ]
static_configs:
  [ - <static_config> ... ]
relabel_configs:
  [ - <relabel_config> ... ]
```

### 远程读写

Prometheus可以进行远程读/写数据。字段`remote_read`和`remote_write`

#### remote_read

```yml
#远程读取的url
url: <string>

#通过标签来过滤读取的数据
required_matchers:
  [ <labelname>: <labelvalue> ... ]

[ remote_timeout: <duration> | default = 1m ]

#当远端不是存储的时候激活该项
[ read_recent: <boolean> | default = false ]

basic_auth:
  [ username: <string> ]
  [ password: <string> ]
  [ password_file: <string> ]
[ bearer_token: <string> ]
[ bearer_token_file: /path/to/bearer/token/file ]
tls_config:
  [ <tls_config> ]
[ proxy_url: <string> ]
```

#### remote_write

```yml
url: <string>

[ remote_timeout: <duration> | default = 30s ]

#写入数据时候进行标签过滤
write_relabel_configs:
  [ - <relabel_config> ... ]

basic_auth:
  [ username: <string> ]
  [ password: <string> ]
  [ password_file: <string> ]

[ bearer_token: <string> ]

[ bearer_token_file: /path/to/bearer/token/file ]

tls_config:
  [ <tls_config> ]

[ proxy_url: <string> ]

#远端写细粒度配置，这里暂时仅仅列出官方注释
queue_config:
  # Number of samples to buffer per shard before we start dropping them.
  [ capacity: <int> | default = 10000 ]
  # Maximum number of shards, i.e. amount of concurrency.
  [ max_shards: <int> | default = 1000 ]
  # Maximum number of samples per send.
  [ max_samples_per_send: <int> | default = 100]
  # Maximum time a sample will wait in buffer.
  [ batch_send_deadline: <duration> | default = 5s ]
  # Maximum number of times to retry a batch on recoverable errors.
  [ max_retries: <int> | default = 3 ]
  # Initial retry delay. Gets doubled for every retry.
  [ min_backoff: <duration> | default = 30ms ]
  # Maximum retry delay.
  [ max_backoff: <duration> | default = 100ms ]
```

















