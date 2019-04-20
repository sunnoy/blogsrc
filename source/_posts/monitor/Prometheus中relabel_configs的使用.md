---
title: Prometheus中relabel_configs的使用
date: 2019-4-16 12:14:28
tags:
- prometheus
- monitor
---

如何使用标签更改，这个在服务发现中使用的很多

<!--more-->

# 关于标签重写

- 内置标签 数据采集到的时候就带的标签 __ 开头
    - __meta_ 为服务发现的内置标签
    - __tmp 为临时存储标签
- 样本标签 出现在查询指标时 {} 内的标签

__ 开头的标签是不会出现在样本标签里面的

系统会默认对 内置标签 __address__ relabel成样本标签 instance 。下面就回提供自定义的relabel


# action

可以用到的action以及对应所需的字段

| 动作 | 所需字段 | 介绍 |
|--|--|--|
| replace | regex source_labels target_label replacement | 根据正则匹配标签的**值**,替换标签target_label必须 |
| keep | regex source_labels  | 根据正则匹配标签的**值**保留数据采集源 |
| drop | regex source_labels  | 根据正则匹配标签的**值**剔除数据采集源 |
| hashmod | source_labels target_label modulus | hash模式 |
| labelmap | regex replacement | 根据正则匹配标签的**名称**进行映射 |
| labeldrop | regex | 根据正则匹配标签的**名称**剔除标签 |
| labelkeep | regex | 根据正则匹配标签的**名称**保留标签 |

action的默认动作是 relace

# replace

下面主要介绍一下 replace 的一些使用

## 初始替换

下面的例子就是简单的替换，将原来的标签 lr 替换成 dc。其中包含一些字段的默认值 action 和 replacement 以及 regex separator

replace 动作流程
- 从 source_labels 中获取需要操作的标签列表，并获取他们的值，这些值是一个列表，使用 separator 分割
- regex 中的正则表达式会对 值 列表的值进行过滤，为了方便需要使用正则分组
- 使用分组就回通过从分组获取的匹配字符串 赋值给 $1 等，如果仅仅有一个分组的话，字段 replacement 的默认值为 $1
- 新的 target_label 的内容会被 replacement 的内容替换掉，也就是说 target_label: replacement

考虑到一些字段的默认值，仅仅写出 source_labels 和 target_label 就可以直接替换标签，这其实就是 labelmap 标签映射

```yaml
    - source_labels:  ["lr"]
      #action: replace
      #regex: "(.*)"
      target_label: "dc"
      #replacement: $1
      #separator: ;

      # 该示例会将 lr=ee 替换成 dc=ee
```
    


## 标签值自定义

将替换的后标签值换为自定义内容

```yaml
    - source_labels:  ["lr"]
      #action: replace
      #regex: "(.*)"
      target_label: "dc"
      replacement: tt
      #separator: ;

      # 该示例会将 lr=ee 替换成 dc=tt
```

如果 source_labels 是多个标签，那么 dc 的值就是 gg 因为 replacement 字段的作用就是给 target_label 赋值

## 看个例子

下面例子获取docker镜像的id

```yaml
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['prometheus:9090']
        labels:
          image: quiq/logspout:20170306
    relabel_configs:
      - source_labels: ['image']
       # 通过 正则表达式来匹配 quiq/logspout:20170306 后，从正则分组(.*)获取到 logspout:20170306
        regex: .*/(.*)
        replacement: $1
        action: replace
        separator: ;
        target_label: "rrrrrrrrrr"

        #结果就是 rrrrrrrrrr: logspout:20170306
```

# labelmap

labelmap 会将有正则匹字段 regex 配到的字符传递给字段 replacement $1 $2 等 作为值。无论怎么映射，映射前后标签的值并不会改变

例子

```yaml
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['prometheus:9090']
        labels:
          dong: dong
          lr: dongdongodng
          dd: dkgkkdg
          __meta_kubernetes_node_label_tt: sdgss
    relabel_configs:
     # 表达式 第一组匹配字符 __meta_ 后面的字符
     # (.?) 匹配 k 传递给 $1
     # (..) 匹配 ub 传递给 $2
     # (.+) 匹配 ernetes_node_label_tt 传递给 $3
      - regex: __meta_(.?)(..)(.+)
        replacement: $2
        action: labelmap
```

上面的例子会将标签 __meta_kubernetes_node_label_tt: sdgss 映射成 ub: sdgss

当然也可以直接给字段 replacement 赋值 

```yaml
      - regex: __meta_(.?)(..)(.+)
        replacement: rrrrr
        action: labelmap
```

那么，标签 __meta_kubernetes_node_label_tt: sdgss 映射成 rrrrr: sdgss

# drop

扔掉不想采集数据的数据源，根据所选标签的不同可以分为
- 扔掉target
- 扔掉target中的metrics

```yaml
#扔掉target
- job_name: cadvisor
  metric_relabel_configs:
  - source_labels: ['label_in_target']
    regex: '(container_tasks_state|container_memory_failures_total)'
    action: drop
#扔掉采集数据的指标
- job_name: cadvisor
  metric_relabel_configs:
  - source_labels: ['label_in_metrics']
    regex: '(container_tasks_state|container_memory_failures_total)'
    action: drop
```

# labeldrop

扔掉不需要的标签

```yaml
- job_name: cadvisor
  metric_relabel_configs:
  - regex: 'container_label_com_amazonaws_ecs_task_arn'
    action: labeldrop
```

