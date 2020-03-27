---
title: Prometheus PromQL查询
date: 2018-11-20 12:14:28
tags:
- prometheus
---

# Prometheus PromQL查询

## Metric

指标是多种监控项

### 表示

```bash
<metric name>  {<label name>=<label value>, ...}  value
#例如 
node_cpu_seconds_total{cpu="1",mode="idle"} 1.17808959e+06
```

标签(label)反映了当前样本的特征维度，通过这些维度Promtheus可以对样本数据进行过滤，聚合等

每次采集一条数据就是一个样本

<!--more-->

### 类型

Prometheus总结了4种样本类型

#### Counter

只增不减的计数器`rate(http_requests_total[5m])`

#### Gauge

就是当前采集的测量值`node_memory_MemFree`

#### Histogram柱状图

主要用来计算在一定范围内的分布情况。

示例

```bash
# HELP prometheus_tsdb_compaction_chunk_size_bytes Final size of chunks on their first compaction
# TYPE prometheus_tsdb_compaction_chunk_size_bytes histogram
prometheus_tsdb_compaction_chunk_size_bytes_bucket{le="32"} 4
prometheus_tsdb_compaction_chunk_size_bytes_bucket{le="48"} 558827
prometheus_tsdb_compaction_chunk_size_bytes_bucket{le="72"} 958126
prometheus_tsdb_compaction_chunk_size_bytes_bucket{le="108"} 1.4479e+06
prometheus_tsdb_compaction_chunk_size_bytes_bucket{le="162"} 1.552101e+06
prometheus_tsdb_compaction_chunk_size_bytes_bucket{le="243"} 1.712441e+06
prometheus_tsdb_compaction_chunk_size_bytes_bucket{le="364.5"} 1.785153e+06
prometheus_tsdb_compaction_chunk_size_bytes_bucket{le="546.75"} 1.840005e+06
prometheus_tsdb_compaction_chunk_size_bytes_bucket{le="820.125"} 1.948824e+06
prometheus_tsdb_compaction_chunk_size_bytes_bucket{le="1230.1875"} 1.975387e+06
prometheus_tsdb_compaction_chunk_size_bytes_bucket{le="1845.28125"} 1.975387e+06
prometheus_tsdb_compaction_chunk_size_bytes_bucket{le="2767.921875"} 1.975387e+06
prometheus_tsdb_compaction_chunk_size_bytes_bucket{le="+Inf"} 1.975387e+06
prometheus_tsdb_compaction_chunk_size_bytes_sum 2.83911803e+08
prometheus_tsdb_compaction_chunk_size_bytes_count 1.975387e+06
```

柱状图会提供三种metric
- 观察桶累计计数器，命名为<basename>_bucket={le="<upper inclusive bound>"}
- 观察值的总和，命名为<basename>_sum
- 已经观察到的事件数量，命名为<basename>_count，等同于<basename>_bucket{le="+Inf"}

聚合使用:可以使用`histogram_quantile()`函数，从histogram或者是histogram聚合，来计算百分比数目。比如我们想知道90%的http请求低于某个延迟数，可以使用`histogram_quantile(0.9, histogram)`。

#### Summary概要图

作用和Histogram类似，主要提供3中metric

示例

```bash
# HELP prometheus_tsdb_head_gc_duration_seconds Runtime of garbage collection in the head block.
# TYPE prometheus_tsdb_head_gc_duration_seconds summary
prometheus_tsdb_head_gc_duration_seconds{quantile="0.5"} NaN
prometheus_tsdb_head_gc_duration_seconds{quantile="0.9"} NaN
prometheus_tsdb_head_gc_duration_seconds{quantile="0.99"} NaN
prometheus_tsdb_head_gc_duration_seconds_sum 0.6487833710000002
prometheus_tsdb_head_gc_duration_seconds_count 58
```

- 观察时间的φ-quantiles (0 ≤ φ ≤ 1), 显示为[basename]{quantile="[φ]"}
- [basename]_sum，是指所有观察值的总和
- [basename]_count, 是指已观察到的事件计数值

#### Histogram和Summary区别

- 都包含 <basename>_sum，<basename>_count
- Histogram 需要通过 <basename>_bucket 计算 quantile, 而 Summary 直接存储了 quantile 的值

## 查询

### 四种类型

Prometheus的查询语句会查出四种数据类型

- Instant vector 瞬时向量(既有时间又有数值)，查询单个metric的时间序列图`node_disk_read_bytes_total{device="sdc"}`
- Range vector 区间向量，查询单个metric在一段时间内的数据
查询单个metric在一段时间内的数据`node_disk_read_bytes_total{device="sdc"}[1s]`
查询2s前的数据`node_disk_read_bytes_total{device="sdc"}offset 2s`
查询聚合`sum(node_disk_read_bytes_total{device="sdc"}offset 2s)`
时间单位为：`s/m/h/d/w/y`
- Scalar 标量(只有数值)，仅仅是数字
- String 字符串，仅仅是字符串

### PromQL表达式格式

合法的表达式

```bash
node_disk_read_bytes_total # 合法
node_disk_read_bytes_total{}
node_disk_read_bytes_total{device="sdc"} # 合法
#查询所有关于sdc的数据
{device="sdc"} # 合法

#也可以使用__name__来指定metric
{__name__="node_disk_read_bytes_total"} # 合法
{__name__=~"node_disk_read_bytes_total"} # 合法
{__name__=~"node_disk_bytes_read|node_disk_bytes_written"} # 合法
```

### 数据筛选

#### 筛选途径

数据筛选有三种途径

- 对metric名称进行筛选，主要使用数据内建标签`__name__`
- 对数据条目中label进行筛选，主要使用`{}`内的标签
- 对metric值进行筛选，主要使用操作符

#### 正则表达式语法

对label筛选主要使用正则表达式，Prometheus使用的是[RE2](https://github.com/google/re2/wiki/Syntax)表达式

### 两元操作符

#### 数学运算

支持的运算符号
- \+ (加法)
- \- (减法)
- \* (乘法)
- / (除法)
- % (求余)
- ^ (幂运算)

示例

- 瞬时向量和标量

```bash
#主机剩余内存，由byte换算到GB
node_memory_MemFree_bytes/1024/1024/1024
```

- 瞬时向量和瞬时向量

依次找到与左边向量元素匹配（标签完全一致）的右边向量元素进行运算，如果没找到匹配元素，则直接丢弃。同时新的时间序列将不会包含指标名称

```bash
node_disk_written_bytes_total + node_disk_read_bytes_total
```

![相加](https://qiniu.li-rui.top/相加.png)

#### 布尔运算

支持的运算符号

- == (相等)
- != (不相等)
- \> (大于)
- \< (小于)
- \>= (大于等于)
- \<= (小于等于)

示例

```bash
#内存使用率小于90%的主机
(node_memory_MemTotal_bytes - node_memory_MemFree_bytes)/node_memory_MemTotal_bytes < 0.9
```

布尔修饰

```bash
(node_memory_MemTotal_bytes - node_memory_MemFree_bytes)/node_memory_MemTotal_bytes < bool  0.9

#小于0.9就返回1，大于0.9就返回0

```

#### 集合类

支持的运算符

- and (并且) 会产生一个由vector1的元素组成的新的向量。该向量包含vector1中完全匹配vector2中的元素组成
- or (或者) 会产生一个新的向量，该向量包含vector1中所有的样本数据，以及vector2中没有与vector1匹配到的样本数据。
- unless (排除) 会产生一个新的向量，新向量中的元素由vector1中没有与vector2匹配的元素组成。

#### 操作符优先级

`^`最高
- ^
- *, /, %
- +, -
- ==, !=, <=, <, >=, >
- and, unless
- or

#### 标签匹配

进行操作的时候会出现操作符两边数据内的标签不完全一样情况，进行操作的时候需要进行标签筛选。该匹配规则主要用于二元操作符


- 一对一匹配

两边标签完全一致的时候

```bash
vector1 <bin-op> vector2
```

当两边标签不一致的时候，可以使用关键字`on`和`ignoring`来选择匹配的标签，最终让两边标签一致进行两元操作符的运算

##on##用来表示仅仅匹配的标签列表，on所指定标签值在任何一边都应该是唯一的
##ignoring##用来标签仅仅忽略的标签列表

```bash
#样本，ignoring忽略code标签后就会使用两边标签一致
method_code:http_errors:rate5m{method="post", code="500"} 6
method_code:http_errors:rate5m{method="post", code="404"} 21
​
method:http_requests:rate5m{method="get"}  600
method:http_requests:rate5m{method="del"}  34
method:http_requests:rate5m{method="post"} 120

#一对一匹配
method_code:http_errors:rate5m{code="500"} / ignoring(code) method:http_requests:rate5m
```

- 一对多和多对一

多对一和一对多两种匹配模式指的是“一”侧的每一个向量元素可以与"多"侧的多个元素匹配的情况。在这种情况下，必须使用group修饰符：group_left或者group_right来确定结果中全部显示那边的标签。多对一和一对多两种模式一定是出现在操作符两侧表达式返回的向量标签不一致的情况。因此需要使用ignoring和on修饰符来排除或者限定匹配的标签列表。

`group_left<labela>`是结果显示等号左侧的全部标签，labela则是在结果中也加上一些右边(right)的标签

```bash
<vector expr> <bin-op> ignoring(<label list>) group_left(<label list>) <vector expr>
<vector expr> <bin-op> ignoring(<label list>) group_right(<label list>) <vector expr>
<vector expr> <bin-op> on(<label list>) group_left(<label list>) <vector expr>
<vector expr> <bin-op> on(<label list>) group_right(<label list>) <vector expr>

```

- 示例

ceph_osd_metadata包含标签为下。值为`1`

```bash
{ceph_daemon="osd.0",ceph_version="ceph version 12.2.8 (ae699615bac534ea496ee965ac6192cb7e0e07c0) luminous (stable)",cluster="machine_room",cluster_addr="173.20.1.103",device_class="hdd",hostname="matrix_03",instance="192.168.66.112:9283",job="ceph",public_addr="173.20.1.103"}
```

ceph_osd_in包含标签为

```bash
{ceph_daemon="osd.0",cluster="machine_room",instance="192.168.66.112:9283",job="ceph"}
```

现在需要获取的标签为

```bash
{ceph_daemon="osd.0",cluster="machine_room",device_class="hdd",instance="192.168.66.112:9283",job="ceph"}
```

语句

```bash
{ceph_daemon="osd.0",cluster="machine_room",device_class="hdd",hostname="matrix_03",instance="192.168.66.112:9283",job="ceph"}
```

`on`后面接的标签在每个指标中不可以为一样，以左边的标签(ceph_osd_in)为主然后加上右边(ceph_osd_metadata)标签列表中的(device_class和hostname)

### 聚合操作

- sum (求和)
- min (最小值)
- max (最大值)
- avg (平均值)
- stddev (标准差)
- stdvar (标准差异)
- count (计数)
- count_values (对value进行计数)
- bottomk (后n条时序)
- topk (前n条时序)
- quantile (分布统计)

聚合操作语法

```bash
<aggr-op>([parameter,] <vector expression>) [without|by (<label list>)]
sum (node_cpu_seconds_total)  by (job)
```

without用来屏蔽聚合后的标签，by用来显示聚合后的标签

### 内置函数

#### 增长率

- increase(node_cpu[2m]) / 120 增长率
- rate(node_cpu[2m]) 增长率和`increase(node_cpu[2m]) / 120`一样
- irate(node_cpu[2m]) 灵敏度更高的增长率

####　预测趋势

基于2小时的样本数据，来预测主机可用磁盘空间的是否在4个小时候被占满
```bash
predict_linear(node_filesystem_free{job="node"}[2h], 4 * 3600) < 0
```

[更多内置函数](https://prometheus.io/docs/prometheus/latest/querying/functions/)

## HTTP API

访问点`/api/v1`返回代码
- 2**为正常
- 422 语句执行失败

返回格式

```json
{
  "status": "success" | "error",
  "data": <data>,

  // Only set if status is "error". The data field may still hold
  // additional data.
  "errorType": "<string>",
  "error": "<string>"
}
```

### 及时查询

```bash
GET /api/v1/query
```

传递参数：
- query=：PromQL表达式。
- time=：用于指定用于计算PromQL的时间戳。可选参数，默认情况下使用当前系统时间。
- timeout=：超时设置。可选参数，默认情况下使用-query,timeout的全局设置。

示例

```bash
$ curl 'http://localhost:9090/api/v1/query?query=up&time=2015-07-01T20:10:51.781Z&timeout=3s'
```
响应数据类型

```bash
{ #矩阵matrix
  "resultType": "matrix" | "vector" | "scalar" | "string",
  "result": <value>
}
```

### 区间数据查询

```bash
GET /api/v1/query_range
```

请求参数
- query=: PromQL表达式。
- start=: 起始时间。
- end=: 结束时间。
- step=: 查询步长。
- timeout=: 超时设置。可选参数，默认情况下使用-query,timeout的全局设置。

相应格式

```bash
{
  "resultType": "matrix",
  "result": <value>
}
```

示例

```bash
$ curl 'http://localhost:9090/api/v1/query_range?query=up&start=2015-07-01T20:10:30.781Z&end=2015-07-01T20:11:00.781Z&step=15s'
```
[更多使用](https://prometheus.io/docs/prometheus/latest/querying/api/)


## 那么问题来了，如何快速入门PromQL语法？

[强烈推荐参见](https://prometheus.io/docs/visualization/grafana/)

导入完成后看到漂亮的界面就点击

![edit](https://qiniu.li-rui.top/edit.png)

然后开启语法学习之旅把

![pql](https://qiniu.li-rui.top/pql.png)












