---
title: alertmanager信息梳理
date: 2019-7-25 12:14:28
tags:
- prometheus
---

# 概览

prom会根据规则去触发报警消息，然后将消息发送到alermanager

<!--more-->

# 两个循环

![twoloop](https://qiniu.li-rui.top/twoloop.png)

- 采集循环和评估循环是独立计时的
- 评估对象为上一个采集周期的值
- 对于评估效果不符合条件和没有值是一样的

# 信息流

## 总流程

![报警流程](https://qiniu.li-rui.top/报警流程.png)

## alertmanger

![alert-art](https://qiniu.li-rui.top/alert-art.png)

## 关于分组

以标签分组的意思是以标签的值分组，比如以标签a分组：

- a=b 为一组
- a=c 为一组

## repeat_interval

- 针对每条报警信息
- 每个报警会做哈希计算，再次发送的间隔

## group_wait group_interval

- 针对group
- group_wait 为初始发送缓存
- group_interval 改组已经发送过报警了，新来报警发送间隔

# 报警抑制

分为两种

- inhibition
- Silence

## inhibition

目的是“关联抑制”：一个node挂了，上面的服务所产生的报警都被抑制

在配置文件里面进行配置，对于分布式alertmanger每个实例可以分别配置。

```yaml
- source_match:
# node挂掉报警触发
    alertname: NodeDown
    severity: critical
# node上的服务的报警就会与之相关联，都会被抑制
  target_match:
    severity: critical
# source_match和target_match所匹配的报警条目中必须含有同样的node标签，并且其值相同
  equal:
    - node
```

## Silence

直接在alertmanger进行可视化操作，没有“抑制关联”。页面示例

![silencd](https://qiniu.li-rui.top/silencd.png)

# 参考资料

```bash
https://pracucci.com/prometheus-understanding-the-delays-on-alerting.html
https://www.robustperception.io/whats-the-difference-between-group_interval-group_wait-and-repeat_interval
https://yeya24.github.io/post/am-1/
https://github.com/prometheus/alertmanager
https://yunlzheng.gitbook.io/prometheus-book/parti-prometheus-ji-chu/alert/alert-manager-inhibit
```

