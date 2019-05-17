---
title: elasticsearch dsl语法之组合查询
date: 2019-05-16 12:12:28
tags:
- efk
---

一些dsl语法介绍，本文主要介绍组合查询。

组合查询会对多个条件进行查询

<!--more-->

# constant_score


对符合的字段查询给一个得分值

```json
GET /_search
{
  "query": {
    "constant_score": {
      "filter": {
        "term": {
          "kubernetes.namespace_name": "efk"
        }
      },
      "boost": 1
    }
  }
}
```

# bool

bool查询为设定一个满足类型，看是否满足这些类型。

包含的类型有

- must 文档中必须包含，增加文档得分
- filter 文档中必须包含，但是文档没有得分
- should 文档中应该有查询的条件，可以通过minimum_should_match来设定应该包含的条目
- must_not 文档中一定不包含查询的条件

```json
POST _search
{
  "query": {
    "bool" : {
      "must" : {
        "term" : { "user" : "kimchy" }
      },
      "filter": {
        "term" : { "tag" : "tech" }
      },
      "must_not" : {
        "range" : {
          "age" : { "gte" : 10, "lte" : 20 }
        }
      },
      "should" : [
        { "term" : { "tag" : "wow" } },
        { "term" : { "tag" : "elasticsearch" } }
      ],
      "minimum_should_match" : 1,
      "boost" : 1.0
    }
  }
}

```

# dis_max

该查询返回子查询结果的并集，多子查询的中的最高的那个得分子查询就是这个总查询的得分

```json
GET /_search
{
    "query": {
        "dis_max" : {
            "tie_breaker" : 0.7,
            "boost" : 1.2,
            "queries" : [
              //假如得分为1
                {
                    "term" : { "age" : 34 }
                },
              //假如得分为2  
                {
                    "term" : { "age" : 35 }
                }
            ]
        }
    }
}
//总得分为2
```

# function_score

该查询用来自定分数，相当于在默认的相关度上进行定制化匹配。这里不再详细介绍，[详细请看](https://www.jianshu.com/p/d6e7c39d9dcd)

```json
GET /_search
{
    "query": {
        "function_score": {
          "query": { "match_all": {} },
          "boost": "5", 
          "functions": [
              {
                  "filter": { "match": { "test": "bar" } },
                  "random_score": {}, 
                  "weight": 23
              },
              {
                  "filter": { "match": { "test": "cat" } },
                  "weight": 42
              }
          ],
          "max_boost": 42,
          "score_mode": "max",
          "boost_mode": "multiply",
          "min_score" : 42
        }
    }
}
```

# boosting

该查询用来判定优先级或者降低优先级

```json
GET /_search
{
    "query": {
        "boosting" : {
            "positive" : {
                "term" : {
                    "field1" : "value1"
                }
            },
            "negative" : {
                 "term" : {
                     "field2" : "value2"
                }
            },
            "negative_boost" : 0.2
        }
    }
}
```

