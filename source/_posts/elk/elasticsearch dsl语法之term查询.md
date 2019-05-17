---
title: elasticsearch dsl语法之term查询
date: 2019-05-15 12:12:28
tags:
- efk
---

一些dsl语法介绍，本文主要介绍term查询

<!--more-->

# 文档字段类型

- text 
- keyword

下面是文档内容

```json
{
  "name": "washing machine"
}
```

## text 

查询的字段值和文档中的字段值不是精确匹配

### 创建索引

```json
{
  "mappings": {
    "_doc": {
      "properties": {
        "name": {
          "type": "text"
        }
      }
    }
  }
}
```

下面查询可以返回文档

```json
{
  "query": {
    "match": {
      "name": "washing"
    }
  }
}
```

## keyword

查询的字段值和文档中的字段值必须要精确匹配才能查询到文档

```json
{
  "mappings": {
    "_doc": {
      "properties": {
        "name": {
          "type": "keyword"
        }
      }
    }
  }
}
```

下面查询**不会**返回文档

```json
{
  "query": {
    "match": {
      "name": "washing"
    }
  }
}
```

# term查询

term查询并不会使用解析器，主要使用精确查询

json文档中字段(field)的值就是term

## term查询

```json
GET _search
{
  "query": {
    "term": {
      //文档中字段值
      "kubernetes.namespace_name": {
        //该字段值的内容，要精确匹配
        "value": "default",
        //boost会将匹配的文档得分乘以boost值
        "boost": 2
      }
    }
  }
}
```

## terms查询

terms查询用来查询文档中某一字段多个值的情形

k8s中查询namespace为efk和sre的文档

```json
GET _search
{
  "query": {
    "terms": {
      "kubernetes.namespace_name": [
        "efk",
        "sre"
      ]
    }
  }
}
```

## terms_set

文档中字段值是一个数组，数组内有多个元素。索引映射添加一个数字域的字段，然后在文档中规定，查询词必须要包含字段值数组内几个元素才可以被查询到

### 创建映射

```json
PUT /my-index
{
    "mappings": {
        "_doc": {
            "properties": {
                "required_matches": {
                    "type": "long"
                }
            }
        }
    }
}

PUT /my-index/_doc/1?refresh
{
    "codes": ["ghi", "jkl"],
    "required_matches": 2
}

PUT /my-index/_doc/2?refresh
{
    "codes": ["def", "ghi"],
    "required_matches": 2
}
```

### 进行查询

```json
GET /my-index/_search
{
    "query": {
        "terms_set": {
            "codes" : {
                "terms" : ["abc", "def", "ghi"],
                "minimum_should_match_field": "required_matches"
            }
        }
    }
}
```

## 其他的一些查询

### range

```json
GET _search
{
    "query": {
        "range" : {
            "age" : {
                "gte" : 10,
                "lte" : 20,
                "boost" : 2.0
            }
        }
    }
}
```

### prefix

```json
GET /_search
{ "query": {
    "prefix" : { "user" : "ki" }
  }
}
```

### wildcard

```json
GET /_search
{
    "query": {
        "wildcard" : { "user" : "ki*y" }
    }
}
```

### regexp

```json
GET /_search
{
    "query": {
        "regexp":{
            "name.first": "s.*y"
        }
    }
}

```

### fuzzy

当查询的字符和字段值的字符不一致时，也就是输入错误，那么使用fuzzy查询就可以自动纠错

```json
GET /_search
{
    "query": {
        "fuzzy" : {
            //字段值
            "user" : {
                "value": "ki",
                "boost": 1.0,
                //纠错次数，2 就是容许输错两个字母，可选为 0 1 2 AUTO
                "fuzziness": 2,
                //查询字符前几个字母不参与模糊查询 默认为0 减少性能消耗
                "prefix_length": 0,
                //模糊查询最大的terms
                "max_expansions": 100
            }
        }
    }
}
```

### type

```json
GET /_search
{
    "query": {
        "type" : {
            "value" : "_doc"
        }
    }
}
```



