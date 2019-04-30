---
title: Prometheus Alertmanager使用
date: 2018-11-12 12:14:28
tags:
- prometheus
- monitor
---

# Prometheus Alertmanager使用

报警分为两部分：Prometheus servers负责报警规则，Alertmanager负责发送报警信息


## Alertmanager报警特点

- 分组，太多的报警信息来到时，可以分组发送
- 抑制，如果一个报警规则触发以后，后面相同的触发就会被抑制
- 静音，直接将个别报警进行屏蔽
- 高可用，可以组成Alertmanager集群

<!--more-->

# Prometheus对接

Prometheus需要加入两个配置项
- 配置文件引入Alertmanager实例
- 定义出报警规则`alerting rules`

## 直接引入alertmanager

```yml
alerting:
  alertmanagers:
    #一组配置项目就是一个列表
    - scheme: https
      static_configs:
        - targets:
            - "1.2.3.4:9093"
            - "1.2.3.5:9093"
            - "1.2.3.6:9093"
    - scheme: http
      static_configs:
        - targets:
            - "1.2.3.7:9093"
            - "1.2.3.8:9093"
            - "1.2.3.9:9093"
```

## 通过kubernetes服务发现

```yaml
alerting:
  alertmanagers:
  - kubernetes_sd_configs:
      - role: pod
    tls_config:
      ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
    relabel_configs:
    - source_labels: [__meta_kubernetes_namespace]
      regex: default
      action: keep
    - source_labels: [__meta_kubernetes_pod_label_app]
      regex: prometheus
      action: keep
    - source_labels: [__meta_kubernetes_pod_label_component]
      regex: alertmanager
      action: keep
    - source_labels: [__meta_kubernetes_pod_container_port_number]
      regex:
      action: drop
```

## 引入报警规则

因为是Prometheus配置文件内引入报警规则，规则内的字段收到配置文件中的`global`字段内容所控制

```yml
rule_files:
  - /glob/path/to/rule_file
```

## 报警规则编写

###  规则分组rule_group

不论是`recording rules`还是`alerting rules`都要在组里面

```yml
groups:
  #groups的名称
  - name: example
    #该组下的规则
    rules:
      [ - <rule> ... ]
```
## 定义alerting rules

### 规则定义

```yml
# 定义alert类型并加上名称
alert: <string>

# PromQL 表达式
#expr: job:request_latency_seconds:mean5m{job="myjob"} > 0.5
expr: <string>

# 报警间隔，含义就是 当一个报警信息触发以后延迟多少时间再去fire
[ for: <duration> | default = 0s ]


# 报警标签
labels:
  [ <labelname>: <tmpl_string> ]

# 报警中的注释
annotations:
  [ <labelname>: <tmpl_string> ]
  #建议写上summary describipton
  #summary: "High request latency on {{ $labels.instance }}"
  #description: "{{ $labels.instance }} has a median request latency above 1s (current value: {{ $value }}s)"
```

报警信息会显示出标签和注释

![alert](https://qiniu.li-rui.top/alert.png)

### label和annotations模板

label和annotations的值是以模板console templates来渲染的，用到[go模板语言](https://golang.org/pkg/text/template/)，报警注释和标签中常用的是变量`$labels`和`$value`，分别表示一个监控实例和`expr`字段上表达式所拉取的值

示例

```yml
annotations:
  summary: "High request latency on {{ $labels.instance }}"
  description: "{{ $labels.instance }} has a median request latency above 1s (current value: {{ $value }}s)"
```

## 评估时间

规则配置以后，默认对规则评估的时间为1m。每隔1m对指标进行查询，看是否满足报警条件

```yaml
global:

  [ evaluation_interval: <duration> | default = 1m ]
```

## Prometheus发送格式

在一个评估过程完成以后，并且考虑到了延迟时间，字段 for 的时间后，Prometheus会向Alertmanager的`/api/v1/alerts`发送下面格式的信息

```json
[
  {
    "labels": {
      "<labelname>": "<labelvalue>",
      ...
    },
    "annotations": {
      "<labelname>": "<labelvalue>",
    },
    "startsAt": "<rfc3339>",
    "endsAt": "<rfc3339>",
    "generatorURL": "<generator_url>"
  },
  ...
]
```

到此，prometheus这边就完了，又发送的信息可以知道，一条报警信息包含了许多标签和注释以及时间。这些都是后面alertmanager所要处理的原材料

# Alertmanager配置

## 配置概述

Alertmanager可以通过命令行或者配置文件进行配置，官方工具[visual editor](https://prometheus.io/webtools/alerting/routing-tree-editor/)可以将配置文件生成路由树

Alertmanager进程不用重启就可以进行新配置启用，需要：

- 发送信号`SIGHUP`到进程
- 同HTTP方式进行POST到`/-/reload`

## 配置文件解析

配置文件主要有下面几个部分

- global进行全局配置
- templates通知内容模板
- route匹配不同规则到特定的接收器(receivers)
- receivers定义不同的接收器，不同的邮箱和微信
- inhibit_rules静音规则


## global

定义邮箱服务等

```yml
global:
  # 如果在这个事件段内监控报警的问题还没有解决就会报警
  [ resolve_timeout: <duration> | default = 5m ]

  # smtp配置
  [ smtp_from: <tmpl_string> ]
  [ smtp_smarthost: <string> ]
  [ smtp_hello: <string> | default = "localhost" ]
  [ smtp_auth_username: <string> ]
  [ smtp_auth_password: <secret> ]
  [ smtp_auth_identity: <string> ]
  [ smtp_auth_secret: <secret> ]
  [ smtp_require_tls: <bool> | default = true ]
  
  #slack配置和其他的一些配置
  [ slack_api_url: <secret> ]
  [ victorops_api_key: <secret> ]
  [ victorops_api_url: <string> | default = "https://alert.victorops.com/integrations/generic/20131114/alert/" ]
  [ pagerduty_url: <string> | default = "https://events.pagerduty.com/v2/enqueue" ]
  [ opsgenie_api_key: <secret> ]
  [ opsgenie_api_url: <string> | default = "https://api.opsgenie.com/" ]
  [ hipchat_api_url: <string> | default = "https://api.hipchat.com/" ]
  [ hipchat_auth_token: <secret> ]
  [ wechat_api_url: <string> | default = "https://qyapi.weixin.qq.com/cgi-bin/" ]
  [ wechat_api_secret: <secret> ]
  [ wechat_api_corp_id: <string> ]
  #定义http认证等信息
  [ http_config: <http_config> ]
```

## route

route定义了报警信息的分发策略，receivers定义了发送媒介，两者合在一起就是什么样的报警信息以什么样的形式发送。主要流程

- 报警信息从Prometheus过来后就依据label(在Prometheus中alerting rules来定义)来分配属于哪一个route中的哪一个group中的哪一个alert
- 新的Group等待group_wait指定的时间（等待时可能收到同一Group的Alert），根据resolve_timeout判断Alert是否解决，然后发送通知
- 已有的Group等待group_interval指定的时间，判断Alert是否解决，当上次发送通知到现在的间隔大于repeat_interval或者Group有更新时会发送通知

[路由树查看](https://prometheus.io/webtools/alerting/routing-tree-editor/)

### 总路由

总路由的默认配置，总路由一定要包含一个 receiver 来充当默认 receiver

```yml
#总路由，下面的配置都是各个子路由所继承的配置，如果子路由中没有配置就是使用总路由的配置，子路由中可以覆盖总路由的配置
route:
  #定义全路由默认的接受者
  receiver: 'default-receiver'
  #报警是否需要继续向兄弟(同级)节点匹配
  [ continue: <boolean> | default = false ]
  #等待还有可能到达的同组报警信息，初次发送一条警报包含多个信息
  [ group_wait: <duration> | default = 30s ]
  #已经存在的group等待group_interval这个时间段看报警问题是否解决
  [ group_interval: <duration> | default = 5m ]
  #当上次发送完报警信息之后再次发送的间隔，用来重复发送报警信息
  [ repeat_interval: <duration> | default = 4h ]
  #根据不同的标签来分组，这个主要用来筛选报警信息
  group_by: [cluster, alertname]
  #使用标签值精确匹配
  match:
    [ <labelname>: <labelvalue>, ... ]
  #使用正则表达进行范围匹配
  match_re:
    [ <labelname>: <regex>, ... ]

```

### 子路由

子路由下面可以进行多个信息流匹配，子路由中需要考虑关键子 continue 来确定是否让报警信息在同级子路由中传递

```yaml
route:
  # 子路由
  routes:
  - receiver: 'database-pager'
    #以正则表达式匹配标签
    match_re:
      service: mysql|cassandra
    #以标签来匹配
    match:
      service: files
      receiver: team-Y-mails
    

```

### 路由树

先看一下路由配置

```yaml
route:
  receiver: default
  routes:
  - match:
      service: database
    receiver: a
    routes:
    - match:
        owner: team-X
      receiver: a1
    - match:
        owner: team-Y
      receiver: a2
      routes:
      - match:
          owner: ll
        receiver: a2.1
      - match:
          owner: rr
        receiver: a2.2
  - match:
    receiver: b
  - match:
    receiver: c
    routes:
    - match:
      receiver: c1
    - match:
      receiver: c2

```

他的路由树就是

![routetree](https://qiniu.li-rui.top/routetree.png)

从上面可以见到，
- 具有标签 service: database 的报警信息会流向 receiver: a
- 具有标签 service: database owner: team-X 的报警信息会流向 receiver: a1
- 具有标签 service: database owner: team-Y 的报警信息会流向 receiver: a2 
- 具有标签 service: database owner: team-Y owner: ll 的报警信息会流向 receiver: a2.1
- 具有标签 service: database owner: team-Y owner: rr 的报警信息会流向 receiver: a2.2

## inhibit_rule

用来配置将一些报警抑制

```yml
# target 配置
target_match:
  [ <labelname>: <labelvalue>, ... ]
target_match_re:
  [ <labelname>: <regex>, ... ]

# alert匹配
source_match:
  [ <labelname>: <labelvalue>, ... ]
source_match_re:
  [ <labelname>: <regex>, ... ]

# 标签必须完全匹配
[ equal: '[' <labelname>, ... ']' ]
```

## receiver

定义出多种发送器配置

```yml
# 定义一个接收器name
name: <string>

# 多种类型
email_configs:
  [ - <email_config>, ... ]
hipchat_configs:
  [ - <hipchat_config>, ... ]
pagerduty_configs:
  [ - <pagerduty_config>, ... ]
pushover_configs:
  [ - <pushover_config>, ... ]
slack_configs:
  [ - <slack_config>, ... ]
opsgenie_configs:
  [ - <opsgenie_config>, ... ]
webhook_configs:
  [ - <webhook_config>, ... ]
victorops_configs:
  [ - <victorops_config>, ... ]
wechat_configs:
  [ - <wechat_config>, ... ]
```

## webhook_config

着重来看一下webhook_config配置，报警信息会通过HTTP POST方式去向webhook发送信息

```yml
# 已经解决的问题是否发送给webhook
[ send_resolved: <boolean> | default = true ]

# webhook地址
url: <string>

#向webhook发送信息是否需要http相关认证等信息
[ http_config: <http_config> | default = global.http_config ]
```

向webhook推送的信息，该信息是一个json字典在templates可以直接调用

```json
{
  "version": "4",
  "groupKey": <string>,    // key identifying the group of alerts (e.g. to deduplicate)
  "status": "<resolved|firing>",
  "receiver": <string>,
  "groupLabels": <object>,
  "commonLabels": <object>,
  "commonAnnotations": <object>,
  "externalURL": <string>,  // backlink to the Alertmanager.
  "alerts": [
    {
      "status": "<resolved|firing>",
      "labels": <object>,
      "annotations": <object>,
      "startsAt": "<rfc3339>",
      "endsAt": "<rfc3339>",
      "generatorURL": <string> // identifies the entity that caused the alert
    },
    ...
  ]
}
```

附上一个测试webhook的服务器代码，这里用python来搭建一个webhook测试服务器

```python
#/root/app.py
from flask import Flask, request
import json
app = Flask(__name__)

@app.route('/send', methods=['POST'])
def send():
    try:
        data = json.loads(request.data)
        alerts = data['alerts']
        for i in alerts:
            print(str(i))
    except Exception as e:
        print(e)
    return 'ok'

if __name__ == '__main__':
    app.run("0.0.0.0")
```

使用docker运行

```bash
docker run --rm -p 5090:5000 --name app -v /root/app.py:/src/app.py jcdemo/flaskapp:latest
```


## templates

究竟向接收器发送什么信息内容呢，这就需要定制一个信息的模板

```yml
templates:
  [ - <filepath> ... ]
```

使用alertmanager推送的json字典来使用，示例

```bash
{{ define "alert.html" }}
<table>
    <tr><td>报警名</td><td>开始时间</td></tr>
    {{ range $i, $alert := .Alerts }}
        <tr><td>{{ index $alert.Labels "alertname" }}</td><td>{{ $alert.StartsAt }}</td></tr>
    {{ end }}
</table>
{{ end }}

```

模板使用

```yaml
{{ template "alert.html" . }}
```

也可以直接使用

```yaml
    annotations:
      description: '{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 5 minutes.'
      summary: 'Instance {{ $labels.instance }} down'
```

# smtp配置

腾讯企业邮箱配置，需要注意的地方

- 端口使用587 而不是 465 。587是STARTTLS协议的 属于TLS通讯协议 只是他是在STARTTLS命令执行后才对之后的原文进行保护。465端口是为SMTPS（SMTP-over-SSL）协议服务。alert需要使用587端口，走的STARTTLS协议
- smtp_auth_password 需要用到客户端专用密码。这个需要在邮箱里面开启，否则会影响发信服务

```yaml
global:
  smtp_smarthost: 'smtp.exmail.qq.com:587'
  smtp_from: '##'
  smtp_auth_username: '##'
  smtp_auth_password: '##'
  smtp_require_tls: true
  resolve_timeout: 10s
```

如果没有配置客户端专用密码，会报错

```bash
evel=error ts=2019-04-30T06:33:39.542678243Z caller=dispatch.go:280 component=dispatcher msg="Notify for alerts failed" num_alerts=3 err="sending mail from: 501 ϵͳ\ufffd\ufffdǿ\ufffdƿ\ufffd\ufffd\ufffd\ufffdʺ\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffdã\ufffd\ufffd\ufffd\ufffd\ufffd¼exmail.qq.com\ufffd\ufffd\ufffd\ufffd΢\ufffd\ufffd "
level=error ts=2019-04-30T06:33:49.542821223Z caller=notify.go:332 component=dispatcher msg="Error on notify" err="sending mail from: 501 ϵͳ\ufffd\ufffdǿ\ufffdƿ\ufffd\ufffd\ufffd\ufffdʺ\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffdã\ufffd\ufffd\ufffd\ufffd\ufffd¼exmail.qq.com\ufffd\ufffd\ufffd\ufffd΢\ufffd\ufffd "
```

# docker启动

```bash
docker run --name alert --rm -d \
-v /root/test/alert/alertmanager.yml:/etc/alertmanager/alertmanager.yml \
--net=host \
prom/alertmanager
```
