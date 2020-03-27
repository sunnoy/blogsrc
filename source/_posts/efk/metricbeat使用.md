---
title: metricbeat使用
date: 2018-11-05 12:12:28
tags:
- elk
- metricbeat
---

# metricbeat使用

## ELK

elk是日志收集分析处理可视化工具，是三个软件的缩写，分别是

### Elasticsearch

是个开源分布式搜索引擎，提供搜集、分析、存储数据三大功能。它的特点有：分布式，零配置，自动发现，索引自动分片，索引副本机制，restful风格接口，多数据源，自动搜索负载等

### Logstash

主要是用来日志的搜集、分析、过滤日志的工具，支持大量的数据获取方式。一般工作方式为c/s架构，client端安装在需要收集日志的主机上，server端负责将收到的各节点日志进行过滤、修改等操作在一并发往elasticsearch上去。

### Kibana

Kibana可以为 Logstash 和 ElasticSearch 提供的日志分析友好的 Web 界面

### Beats

Beats是一个代理平台，其监控数据主要发往Logstash和Elasticsearch
<!-- more -->
## elk组件调用

Beats在机器上收集监控数据发往Logstash和Elasticsearch，Logstash收集的日志也会发往Elasticsearch。

Elasticsearch将接收到的数据进行分析存储，通过Elasticsearch搜索和搜集后在Kibana进行展示。

![elk调用](https://qiniu.li-rui.top/elk调用.png)

## Beats

Beats是一个代理/(采集)平台，它可以将不同类型的数据发送到elasticsearch，也可以通过Logstash发送到elasticsearch。

![beates架构](https://qiniu.li-rui.top/beates架构.png)

Beats有多个采集组件来对应采集不同类型的数据，组件包含elk官网组件和社区组件

### 官方组件

#### Auditbeat

系统审计组件

#### Filebeat

日志文件采集组件，可采集日志对象有服务器，虚拟机，容器等

#### Heartbeat

对一个服务或者应用通过URL进行心跳检测，判断是否在alive状态

#### Metricbeat

提供对系统和应用的状态统计，比如处理器和内存以及nginx和redis应用的统计

#### Packetbeat

针对网络数据包的采集

#### Winlogbeat

针对Windows系统的窗口事件采集

### 社区组件

社区组件[列表](https://www.elastic.co/guide/en/beats/libbeat/6.3/community-beats.html)

## metricbeat


### Metricbeat如何工作

#### module和metricsets
Metricbeat包含多个module(modules)和metricsets。modules中定义了对一个服务进行数据采集的全部内容，包含如何链接服务，对哪个参数进行收集以及收集的频率。一个modules包含多个metricsets，他是对收集服务参数的进一步细化，比如redismodule中的info metricsets就是获取redis的服务器信息

![redisinfo](https://qiniu.li-rui.top/redisinfo.png)

module会在固定的事件间隔去轮询所采集的服务，当远端服务无法采集时就会返回错误，随着轮询的进行会持续返回错误。

#### module事件

每当Metricbeat进行一次轮询就会产生一次事件(event)，事件的返回时一个json，事件的返回时异步的，如果output无效那么事件就会丢失。

示例，其中rtt(Round trip time)为采集开始到获取数据所花费的时间，单位为微秒(10的6次方微妙为1秒)

```json
{
  "@timestamp": "2016-06-22T22:05:53.291Z",
  "beat": {
    "hostname": "host.example.com",
    "name": "host.example.com"
  },
  "metricset": {
    "module": "system",
    "name": "process",
    "rtt": 7419
    //以上为必须字段
  },
  .
  .
  .

  "type": "metricsets"
}
```

更多module的事件返回字段[在此](https://www.elastic.co/guide/en/beats/metricbeat/6.3/exported-fields.html)

当远程服务不可用的时候就会返回错误事件，错误事件是在正常事件的基础上增加error字段，告知错误的返回信息

```json
  "error": {
    "message": "Get http://127.0.0.1/server-status?auto: dial tcp 127.0.0.1:80: getsockopt: connection refused",
  },
```

### module

metricbeat主要用来采集和统计系统和应用信息，这些不同的采集类型是由module(Modules)来完成的，metricbeat官方现在所包含的module[列表](https://www.elastic.co/guide/en/beats/metricbeat/6.3/metricbeat-modules.html)

### 安装

#### rpm
```bash
curl -L -O https://artifacts.elastic.co/downloads/beats/metricbeat/metricbeat-6.3.2-x86_64.rpm
sudo rpm -vi metricbeat-6.3.2-x86_64.rpm
```

#### docker

```bash
docker pull docker.elastic.co/beats/metricbeat:6.3.2
```

### 配置

metricbeat配置文件为yml格式，rpm包安装在/etc/metricbeat/metricbeat.yml，源码编译在go/src/github.com/elastic/beats/metricbeat/metricbeat.yml

启动时可以使用`metricbeat run -c file`来指定配置文件

#### module的启用和禁用

主要使用命令`metricbeat modules`

```bash
#列出module
metricbeat modules list
#启用module
metricbeat modules enable apache mysql
#禁用module
metricbeat modules disable apache mysql
```

#### metricbeat输出(output)

metricbeat所采集的数据流向配置，从上面知道metricbeat采集的数据可以发送到Elasticsearch和Logstash，还可以发送的地方有：

- Elasticsearch
- Logstash
- Kafka
- Redis
- File
- Console
- Cloud

详细的配置[在此]，配置文件为metricbeat.yml(https://www.elastic.co/guide/en/beats/metricbeat/6.3/configuring-output.html)

### 输出为Elasticsearch索引模板

当输出为Elasticsearch时需要对接Elasticsearch的模板系统，需要为Elasticsearch进行索引模板创建，索引模板就是将采集的数据进行格式化处理，对查询数据进行类型化处理的目的是。Elasticsearch根据索引模板会存储数据，Kibana会通过Elasticsearch 的DSL语法进行数据查询和展示.

模板文件为go/src/github.com/elastic/beats/metricbeat/fields.yml

导入模板文件需要先禁用输出到logstash，然后连接到Elasticsearch进行模板导入

```bash
metricbeat setup --template -E output.logstash.enabled=false -E 'output.elasticsearch.hosts=["173.16.1.110:9200"]'
```

### module配置

针对某个module的配置文件在go/src/github.com/elastic/beats/metricbeat/modules.d

示例System module

```yml
# Module: system
# Docs: https://www.elastic.co/guide/en/beats/metricbeat/master/metricbeat-module-system.html

- module: system
  period: 10s
  metricsets:
    - cpu
    - load
    - memory
    - network
    - process
    - process_summary
    #- core
    #- diskio
    #- socket
  process.include_top_n:
    by_cpu: 5      # include top 5 processes by CPU
    by_memory: 5   # include top 5 processes by memory

- module: system
  period: 1m
  metricsets:
    - filesystem
    - fsstat
  processors:
  - drop_event.when.regexp:
      system.filesystem.mount_point: '^/(sys|cgroup|proc|dev|etc|host|lib)($|/)'

```
针对某个metricsets的配置在go/src/github.com/elastic/beats/metricbeat/modules.d的module配置文件内，下面示例cpu配置

```yml
metricbeat.modules:
- module: system
  metricsets: [cpu]
  #可取值为percentages/normalized_percentages/ticks
  cpu.metrics: [percentages, normalized_percentages, ticks]
```














