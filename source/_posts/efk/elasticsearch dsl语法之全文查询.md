---
title: elasticsearch dsl语法之全文查询
date: 2019-05-16 12:12:28
tags:
- efk
---

一些dsl语法介绍，本文主要介绍全文查询。

全文查询会将查询的**语句**输入分析器，然后对分析器出来的term进行和文档中的terms进行对比查询

<!--more-->

# match

一些参数在其他查询类型中可以使用

- zero_terms_query
- fuzziness

```json
GET /_search
{
  "query": {
    "match": {
      "kubernetes.namespace_name": {
        "query": "name is default",
        //分析器出来的词，查询规则，and 为都必须包含比较精确，or 包含任意一个即可 
        "operator": "or",
        //分析器会默认过滤到一些语句中的助词比如 to the a an 等，通过下面的配置就可以让这些助词不让分析器过滤掉
        "zero_terms_query": "all",
        //通过将to the等助词划分为高频词，剩下的为低频词，为了减少性能低频词为必选子查询，高频词为可选子查询。
        //并且低频词没有被查询到，高频词子查询就不会被执行
        "cutoff_frequency": 0.001,
        //是否查询同义词，例如查询ny city 会查询 (ny OR (new AND york)) city
        "auto_generate_synonyms_phrase_query": "true",
        //模糊查询
        "fuzziness": "AUTO"
        
      }
    }
  }
}
```

# match_phrase

查询一个短语，分析器会将查询语句分析成为一个短语，然后去查询

## 什么是短语？

比如查询与就 quick brown fox，查询的时候，文档中一定要出现三个词quick、brown、fox。并且brown比quick位置大1，fox位置比quick大2

```json
GET /_search
{
  "query": {
    "match_phrase": {
      "message": {
        "query": "READ COMPLETE",
        //指定分析器，默认和文档使用的分析器一样
        "analyzer": "simple",
        //slop实际上是一种短语的模糊匹配，见下面
        "slop": 1
      }
    }
  }
}
```

## slop

看个图吧，如果fox quick查询能够匹配quick brown fox的文档，需要slop的值为3

```bash
            Pos 1         Pos 2         Pos 3
-----------------------------------------------
Doc:        quick         brown         fox
-----------------------------------------------
Query:      fox           quick
Slop 1:     fox|quick  ↵  
Slop 2:     quick      ↳  fox
Slop 3:     quick                 ↳     fox
```

# match_phrase_prefix

短语匹配前缀查询

```json
GET /_search
{
    "query": {
        "match_phrase_prefix" : {
            "message" : {
                "query" : "quick brown f",
                //最大匹配后缀的字符数量，默认为50
                "max_expansions" : 10
            }
        }
    }
}
```

# multi_match

在文档的多个字段中查询内容

当我查询READ COMPLETE在字段message和kubernetes.names*_name分为几种情况

- READ COMPLETE 全部在某一个字段
- READ COMPLETE 两个单词分别在两个字段内
- READ COMPLETE 其中一个在某一个字段内
- READ COMPLETE 都不在两个字段内

## best类型

```json
GET /_search
{
  "query": {
    "multi_match": {
      "query": "READ COMPLETE",
      //字段可以使用正则表达式
      "fields": ["message","kubernetes.names*_name"],
      //最佳匹配字段的得分+tie_breaker * 其他所有字段的得分
      //0 为默认其中一个为最高分
      //1 符合条件的字段查询进行相加
      "tie_breaker": 0.7,
      //查询类型 默认为best，比如

      //best为全部在一个字段
      "type": "best_fields"

      //most_fields 为多个字段中含有同样的text
      //phrase_prefix以及phrase 为在字段中查询短语
    }
  }
  
}
```

## cross

will -> firet_name
smith -> last_name

```json
GET /_search
{
  "query": {
    "multi_match" : {
      "query":      "Will Smith",
      "type":       "cross_fields",
      "fields":     [ "first_name", "last_name" ],
      "operator":   "and"
    }
  }
}
```

# common

common查询是将查询语句分为高频和低频，在common查询中可以对高频和低频的相关划分以及查询过程进行配置

```json
GET /_search
{
    "query": {
        "common": {
            "body": {
                "query": "nelly the elephant as a cartoon",
                //高频切分比例
                "cutoff_frequency": 0.001,
                //低频查询和高频查询关系 and 为低频查询也应该有
                "low_freq_operator": "and",
                //这个定义高频分词过后的最小匹配terms数量
                 "minimum_should_match": 2,
                 //或者进行细致的配置
                "minimum_should_match": {
                    "low_freq" : 2,
                    "high_freq" : 3
                }
            }
        }
    }
}
```

# query_string 和 simple_query_string

看个例子吧，不过过多介绍，暂时用的不多

```json
GET /_search
{
    "query": {
        "query_string" : {
            "default_field" : "content",
            "query" : "this AND that OR thus"
        }
    }
}
```








