---
title: elasticsearch 聚合之Bucket
date: 2019-05-16 12:12:28
tags:
- efk
---


elasticsearch中聚合操作，本文介绍bucket聚合

bucket主要是基于条件来划分文档，搜索结果会返回每个bucket的文档的数量，也可以根据在每个桶里面进行一个agg的操作

<!--more-->

# terms

统计每个namespace中产生文档的数量

```json
POST _search?size=0
{
  "aggs": {
    "namespace_logs_num": {
      "terms": {
        "field": "kubernetes.namespace_name.keyword"
      }
    }
  }
}
```

返回

```json
  //...
  "aggregations" : {
    "namespace_logs_num" : {
      "doc_count_error_upper_bound" : 0,
      "sum_other_doc_count" : 0,
      "buckets" : [
        {
          "key" : "default",
          "doc_count" : 805462
        },
        {
          "key" : "efk",
          "doc_count" : 60964
        },
        {
          "key" : "sre",
          "doc_count" : 1834
        },
        {
          "key" : "kube-system",
          "doc_count" : 192
        }
      ]
    }
  }
  //...
```

# filter

查询某个namespace的日志数量

```json
GET _search?size=0
{
  "aggs": {
    "namespace": {
      "filter": {
        "term": {
          "kubernetes.namespace_name": "efk"
        }
      }
    }
  }
}
```
返回

```json
  "aggregations" : {
    "namespace" : {
      "doc_count" : 61043
    }
  }
```

# filters

对每个字段进行过滤

```json
GET logs/_search
{
  "size": 0,
  "aggs" : {
    "messages" : {
      "filters" : {
        "filters" : {
          "errors" :   { "match" : { "body" : "error"   }},
          "warnings" : { "match" : { "body" : "warning" }}
        }
      }
    }
  }
}
```

返回

```json
"aggregations": {
    "messages": {
      "buckets": {
        "errors": {
          "doc_count": 1
        },
        "warnings": {
          "doc_count": 2
        }
      }
    }
}
```

# adjacency_matrix

通过不同的过滤来查询结果，直接看例子吧

```json
GET emails/_search
{
  "size": 0,
  "aggs" : {
    "interactions" : {
      "adjacency_matrix" : {
        "filters" : {
          "grpA" : { "terms" : { "accounts" : ["hillary", "sidney"] }},
          "grpB" : { "terms" : { "accounts" : ["donald", "mitt"] }},
          "grpC" : { "terms" : { "accounts" : ["vladimir", "nigel"] }}
        }
      }
    }
  }
}
```

返回，注意看key哈

```json
  "aggregations": {
    "interactions": {
      "buckets": [
        {
          "key":"grpA",
          "doc_count": 2
        },
        {
          "key":"grpA&grpB",
          "doc_count": 1
        },
        {
          "key":"grpB",
          "doc_count": 2
        },
        {
          "key":"grpB&grpC",
          "doc_count": 1
        },
        {
          "key":"grpC",
          "doc_count": 1
        }
      ]
    }
  }
```

# date聚合

## 自定时间间隔

```json
POST /sales/_search?size=0
{
    "aggs" : {
        "sales_over_time" : {
            "date_histogram" : {
                //指出所要聚合的字段
                "field" : "date",
                //确定时间间隔
                //时间间隔可以是 90m 等
                "interval" : "month",
                //指定时区
                "time_zone": "-01:00",
                //指定偏移量
                "offset": "+6h",
                //
            }
        }
    }
}
```

## 定义时间范围



```json
POST /sales/_search?size=0
{
   "aggs": {
       "range": {
           "date_range": {
               "field": "date",
               //时区
               "time_zone": "CET",
               "ranges": [
                 //确定时间格式
                  { "to": "2016/02/01" }, 
                  //按天来
                  { "from": "2016/02/01", "to" : "now/d" }, 
                  { "from": "now/d" }
              ]
          }
      }
   }
}
```

# nest 聚合

## 嵌套maping

```json
PUT /index
{
  "mappings": {
    "product" : {
        "properties" : {
            "resellers" : { 
                "type" : "nested",
                "properties" : {
                    "name" : { "type" : "text" },
                    "price" : { "type" : "double" }
                }
            }
        }
    }
  }
}
```

## 查询

```json
GET /_search
{
    "query" : {
        "match" : { "name" : "led tv" }
    },
    "aggs" : {
        "resellers" : {
            "nested" : {
                "path" : "resellers"
            },
            "aggs" : {
                "min_price" : { "min" : { "field" : "resellers.price" } }
            }
        }
    }
}
```

# range聚合

```json
GET /_search
{
    "aggs" : {
        "price_ranges" : {
            "range" : {
                "field" : "price",
                "ranges" : [
                  //小于100的
                    { "to" : 100.0 },
                    //100 到 200的
                    { "from" : 100.0, "to" : 200.0 },
                    // 大于200的
                    { "from" : 200.0 }
                ]
            }
        }
    }
}

```

## 自定义key

```bash
GET /_search
{
    "aggs" : {
        "price_ranges" : {
            "range" : {
                "field" : "price",
                "keyed" : true,
                "ranges" : [
                    { "key" : "cheap", "to" : 100 },
                    { "key" : "average", "from" : 100, "to" : 200 },
                    { "key" : "expensive", "from" : 200 }
                ]
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
        "price_ranges" : {
            "buckets": {
              //这里显示自定义的key值
                "cheap": {
                    "to": 100.0,
                    "doc_count": 2
                },
                "average": {
                    "from": 100.0,
                    "to": 200.0,
                    "doc_count": 2
                },
                "expensive": {
                    "from": 200.0,
                    "doc_count": 3
                }
            }
        }
    }
}
```

## 对range进行聚合

```json
GET /_search
{
    "aggs" : {
        "price_ranges" : {
            "range" : {
                "field" : "price",
                "ranges" : [
                    { "to" : 100 },
                    { "from" : 100, "to" : 200 },
                    { "from" : 200 }
                ]
            },
            "aggs" : {
                "price_stats" : {
                  //这里不用指定，本身会自己继承
                    "stats" : {} 
                }
            }
        }
    }
}
```

# histogram

创建相关直方图

```json
POST /sales/_search?size=0
{
    "aggs" : {
        "prices" : {
            "histogram" : {
                "field" : "price",
                "interval" : 50,
                //间隔包含的文档数量最少为1个
                "min_doc_count" : 1,
                //直方图的划分范围
                "extended_bounds" : {
                    "min" : 0,
                    "max" : 500
                },
                //添加key
                "keyed" : true,
                //匹配不到的就给定值为0
                "missing": 0 
            }
        }
    }
}
```

# ip

## 掩码

```json
GET /ip_addresses/_search
{
    "size": 0,
    "aggs" : {
        "ip_ranges" : {
            "ip_range" : {
                "field" : "ip",
                "ranges" : [
                    { "mask" : "22.0.0.0/25" },
                    { "mask" : "22.0.0.127/25" }
                ]
            }
        }
    }
}
```

## IP段

```json
GET /ip_addresses/_search
{
    "size": 10,
    "aggs" : {
        "ip_ranges" : {
            "ip_range" : {
                "field" : "ip",
                "ranges" : [
                    { "to" : "22.0.0.5" },
                    { "from" : "22.0.0.5" }
                ]
            }
        }
    }
}
```

# composite

多种聚合联合使用

```json
GET /_search
{
    "aggs" : {
        "my_buckets": {
            "composite" : {
                "sources" : [
                    { "shop": { "terms": {"field": "shop" } } },
                    { "product": { "terms": { "field": "product" } } },
                    { "date": { "date_histogram": { "field": "timestamp", "interval": "1d" } } }
                ]
            }
        }
    }
}
```

# global

优化查询的一种方法，本文不详细说了。

```json
POST /sales/_search?size=0
{
    "query" : {
        "match" : { "type" : "t-shirt" }
    },
    "aggs" : {
        "all_products" : {
            "global" : {}, 
            "aggs" : { 
                "avg_price" : { "avg" : { "field" : "price" } }
            }
        },
        "t_shirts": { "avg" : { "field" : "price" } }
    }
}
```

[详见](https://blog.csdn.net/zwgdft/article/details/83215977)



