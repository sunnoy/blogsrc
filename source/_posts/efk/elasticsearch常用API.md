---
title: elasticsearch常用API
date: 2020-01-15 12:12:28
tags:
- efk
---

# 介绍常用的api

<!--more-->

```bash
# 节点负载状态
GET _cat/nodes?v

# 查看node容量
GET /_cat/allocation?v

# 查看索引占用的空间
GET /_cat/indices?v

# 查看node属性
GET /_cat/nodeattrs?v

# 查看没有分配的分片
GET _cat/shards?h=index,shard,prirep,state,unassigned.reason

# 查看未分配的状态
GET _cluster/allocation/explain?pretty

# 重新分片
POST /_cluster/reroute?retry_failed=true
```