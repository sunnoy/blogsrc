---
title: elasticsearch 聚合之metrics
date: 2019-05-16 12:12:28
tags:
- efk
---


elasticsearch中聚合操作，本文介绍metrics聚合

metrics聚合的数据来源有两个

- 文档中的字段值 一般为数字域
- 脚本得出的数字

<!--more-->

# max min sum avg

```json
POST /bank/_search?
{
  "size": 0, 
  "aggs": {
    "masssbalance": {
      "max": {
        "field": "balance"
      }
    }
  }
}
```

## 查询排序聚合

```json
POST /bank/_search?
{
  "size": 2, 
  "query": {
    "match": {
      "age": 24
    }
  },
  "sort": [
    {
      "balance": {
        "order": "desc"
      }
    }
  ],
  "aggs": {
    "max_balance": {
      "max": {
        "field": "balance"
      }
    }
  }
}
```

# 去重计数

查询某个字段的值的数量，例如查询有多少个pod进行了日志输出

```json
//size=0 返回几条数据，只显示聚合结果
POST _search?size=0
{
    "aggs" : {
        "namespace" : {
            "cardinality" : {
                //一定要使用keyword
                "field" : "kubernetes.pod_name.keyword",
                //越大越精确
                "precision_threshold": 200
            }
        }
    }
}
```

# stats 

主要是统计 count max min avg sum 几个值

```json
POST /bank/_search?size=0
{
  "aggs": {
    "age_stats": {
      "stats": {
        "field": "age"
      }
    }
  }
}
```

extended_stats 还会统计平方和、方差、标准差、平均值加/减两个标准差的区间

# percentiles

## 默认占比

字段值所占比例的区间 [ 1, 5, 25, 50, 75, 95, 99 ]

```json
GET latency/_search
{
    "size": 0,
    "aggs" : {
        "load_time_outlier" : {
            "percentiles" : {
                "field" : "load_time" 
            }
        }
    }
}
```

返回

```json
{
    ...

   "aggregations": {
      "load_time_outlier": {
         "values" : {
            "1.0": 5.0,
            "5.0": 25.0,
            "25.0": 165.0,
            "50.0": 445.0,
            "75.0": 725.0,
            "95.0": 945.0,
            "99.0": 985.0
         }
      }
   }
}
```

## 自定义占比

```json
GET latency/_search
{
    "size": 0,
    "aggs" : {
        "load_time_ranks" : {
            "percentile_ranks" : {
                "field" : "load_time", 
                "values" : [500, 600]
            }
        }
    }
}

```

# value_count

含有某个字段值的文档数量

```json
GET _search?size=0
{
  "aggs": {
    "pod": {
      "value_count": {
        "field": "kubernetes.namespace_name.keyword"
      }
    }
  }
}
```

# top hits

[详见](https://blog.csdn.net/ctwy291314/article/details/82773180)











