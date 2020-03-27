---
title: elasticsearch 聚合之Pipeline
date: 2019-05-16 12:12:28
tags:
- efk
---


elasticsearch中聚合操作，本文介绍Pipeline聚合

pipeline聚合的源头不再是文档，而是聚合。分为两类

- 父聚合 聚合是串行的
- 同级聚合 聚合是并行的

因为一般聚合过后就是数字，因此pipeline聚合中的聚合种类和metrics中类似

<!--more-->


# 相关参数

## buckets_path

输入聚合的地方

"my_bucket>my_stats.avg" 

输入聚合 my_bucket
metric my_stats
运算符 avg

## _count

也可以在同一个查询中使用_count来替代

## gap_policy

当聚合中有空数据的时候处理 

- skip 跳过
- insert_zeros 差入零值

# avg min max sum

```json
POST /_search
{
  "size": 0,
  "aggs": {
    //创建初始聚合
    "sales_per_month": {
      "date_histogram": {
        "field": "date",
        "interval": "month"
      },
      //嵌套聚合，创建相关指标
      "aggs": {
        "sales": {
          "sum": {
            "field": "price"
          }
        }
      }
    },
    "avg_monthly_sales": {
      "avg_bucket": {
        //使用pipeline聚合
        "buckets_path": "sales_per_month>sales" 
      }
    }
  }
}
```

# stats extended_stats

```json
POST /sales/_search
{
    "size": 0,
    "aggs" : {
        "sales_per_month" : {
            "date_histogram" : {
                "field" : "date",
                "interval" : "month"
            },
            "aggs": {
                "sales": {
                    "sum": {
                        "field": "price"
                    }
                }
            }
        },
        "stats_monthly_sales": {
            "extended_stats_bucket": {
                "buckets_path": "sales_per_month>sales" 
            }
        }
    }
}
```

# percentiles

```json
POST /sales/_search
{
    "size": 0,
    "aggs" : {
        "sales_per_month" : {
            "date_histogram" : {
                "field" : "date",
                "interval" : "month"
            },
            "aggs": {
                "sales": {
                    "sum": {
                        "field": "price"
                    }
                }
            }
        },
        "percentiles_monthly_sales": {
            "percentiles_bucket": {
                "buckets_path": "sales_per_month>sales", 
                "percents": [ 25.0, 50.0, 75.0 ] 
            }
        }
    }
}
```

# cumulative_sum

对聚合进行累计求和

```json
POST /sales/_search
{
    "size": 0,
    "aggs" : {
        "sales_per_month" : {
            "date_histogram" : {
                "field" : "date",
                "interval" : "month"
            },
            "aggs": {
                "sales": {
                    "sum": {
                        "field": "price"
                    }
                },
                "cumulative_sales": {
                    "cumulative_sum": {
                        "buckets_path": "sales" 
                    }
                }
            }
        }
    }
}
```

返回

```json
   "aggregations": {
      "sales_per_month": {
         "buckets": [
            {
               "key_as_string": "2015/01/01 00:00:00",
               "key": 1420070400000,
               "doc_count": 3,
               "sales": {
                  "value": 550.0
               },
               "cumulative_sales": {
                  "value": 550.0
               }
            },
            {
               "key_as_string": "2015/02/01 00:00:00",
               "key": 1422748800000,
               "doc_count": 2,
               "sales": {
                  "value": 60.0
               },
               "cumulative_sales": {
                  "value": 622.0
               }
            },
            {
               "key_as_string": "2015/03/01 00:00:00",
               "key": 1425168000000,
               "doc_count": 2,
               "sales": {
                  "value": 375.0
               },
               "cumulative_sales": {
                  "value": 985.0
               }
            }
         ]
      }
   }
```



