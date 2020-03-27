---
title: loki配置详解
date: 2019-12-20 08:12:28
tags:
- efk
---

# 一致性哈希

![hashing](https://qiniu.li-rui.top/hashing.png)

详细[请参考](https://juejin.im/post/5ae1476ef265da0b8d419ef2)

<!--more-->


# 主要字段

```yaml
# loki的运行模式，通过不同的字段来让loki运行不同的角色
# all, querier, table-manager, ingester, distributor
[target: <string> | default = "all"]

# Enables authentication through the X-Scope-OrgID header, which must be present
# if true. If false, the OrgID will always be set to "fake".
[auth_enabled: <boolean> | default = true]

# 所选模块的详细配
[server: <server_config>]

# distributor 是客户端直接连接的组件
[distributor: <distributor_config>]

# 用来提供查询服务的组件
[querier: <querier_config>]

# Configures how the distributor will connect to ingesters. Only appropriate
# when running all modules, the distributor, or the querier.
[ingester_client: <ingester_client_config>]

# Configures the ingester and how the ingester will register itself to a
# key value store.
# ingester连接到存储
[ingester: <ingester_config>]

# Configures where Loki will store data.
[storage_config: <storage_config>]

# Configures how Loki will store data in the specific store.
[chunk_store_config: <chunk_store_config>]

# Configures the chunk index schema and where it is stored.
[schema_config: <schema_config>]

# Configures limits per-tenant or globally
# 全局限制
[limits_config: <limits_config>]

# Configures the table manager for retention
# 管理数据分组和循环删除
[table_manager: <table_manager_config>]
```

# server 配置

```yaml
# http监听

[http_listen_address: <string>]
[http_listen_port: <int> | default = 80]
[http_server_read_timeout: <duration> | default = 30s]
[http_server_write_timeout: <duration> | default = 30s]
[http_server_idle_timeout: <duration> | default = 120s]
[http_path_prefix: <string>]

# gprc监听

[grpc_listen_address: <string>]
[grpc_listen_port: <int> | default = 9095]
[grpc_server_max_recv_msg_size: <int> | default = 4194304]
[grpc_server_max_send_msg_size: <int> | default = 4194304]
[grpc_server_max_concurrent_streams: <int> | default = 100]

# Register instrumentation handlers (/metrics, etc.)
# 开启prome监控
[register_instrumentation: <boolean> | default = true]

# Timeout for graceful shutdowns
[graceful_shutdown_timeout: <duration> | default = 30s]

# Log [debug,info, warn, error]
[log_level: <string> | default = "info"]
```

# distributor 配置

```yaml
# ingestion 限制重载间隔
[limiter_reload_period: <duration> | default = 5m]
```

# querier配置

```yaml
# 向 ingesters or storage 的查询超时
[query_timeout: <duration> | default = 1m]

# Limit of the duration for which live tailing requests should be
# served.
[tail_max_duration: <duration> | default = 1h]

# Configuration options for the LogQL engine.
engine:
  # 语句执行超时
  [timeout: <duration> | default = 3m]

  # The maximum amount of time to look back for log lines. Only
  # applicable for instant log queries.
  [max_look_back_period: <duration> | default = 30s]
```

# ingester配置

ingester用来将日志放入持久化存储

```yaml
# ingester的注册和发现
[lifecycler: <lifecycler_config>]

# Number of times to try and transfer chunks when leaving before
# falling back to flushing to the store. Zero = no transfers are done.
[max_transfer_retries: <int> | default = 10]

# 并发存储
[concurrent_flushes: <int> | default = 16]

# 发送到存储间隔
[flush_check_period: <duration> | default = 30s]

# flush超时
[flush_op_timeout: <duration> | default = 10s]

# How long chunks should be retained in-memory after they've
# been flushed.
[chunk_retain_period: <duration> | default = 15m]

# How long chunks should sit in-memory with no updates before
# being flushed if they don't hit the max block size. This means
# that half-empty chunks will still be flushed after a certain
# period as long as they receieve no further activity.
[chunk_idle_period: <duration> | default = 30m]

# The targeted _uncompressed_ size in bytes of a chunk block
# When this threshold is exceeded the head block will be cut and compressed inside the chunk
[chunk_block_size: <int> | default = 262144]

# A target _compressed_ size in bytes for chunks.
# This is a desired size not an exact size, chunks may be slightly bigger
# or significantly smaller if they get flushed for other reasons (e.g. chunk_idle_period)
# The default value of 0 for this will create chunks with a fixed 10 blocks,
# A non zero value will create chunks with a variable number of blocks to meet the target size.
[chunk_target_size: <int> | default = 0]

# The compression algorithm to use for chunks. (supported: gzip, lz4, snappy)
# You should choose your algorithm depending on your need:
# - `gzip` highest compression ratio but also slowest decompression speed. (144 kB per chunk)
# - `lz4` fastest compression speed (188 kB per chunk)
# - `snappy` fast and popular compression algorithm (272 kB per chunk)
[chunk_encoding: <string> | default = gzip]

# Parameters used to synchronize ingesters to cut chunks at the same moment.
# Sync period is used to roll over incoming entry to a new chunk. If chunk's utilization
# isn't high enough (eg. less than 50% when sync_min_utilization is set to 0.5), then
# this chunk rollover doesn't happen.
[sync_period: <duration> | default = 0]
[sync_min_utilization: <float> | Default = 0]
```

## lifecycler_config配置

```yaml
# 哈希环配置
[ring: <ring_config>]

# The number of tokens the lifecycler will generate and put into the ring if
# it joined without transferring tokens from another lifecycler.
[num_tokens: <int> | default = 128]

# Period at which to heartbeat to the underlying ring.
[heartbeat_period: <duration> | default = 5s]

# How long to wait to claim tokens and chunks from another member when
# that member is leaving. Will join automatically after the duration expires.
[join_after: <duration> | default = 0s]

# Minimum duration to wait before becoming ready. This is to work around race
# conditions with ingesters exiting and updating the ring.
[min_ready_duration: <duration> | default = 1m]

# Store tokens in a normalised fashion to reduce the number of allocations.
[normalise_tokens: <boolean> | default = false]

# Name of network interfaces to read addresses from.
interface_names:
  - [<string> ... | default = ["eth0", "en0"]]

# Duration to sleep before exiting to ensure metrics are scraped.
[final_sleep: <duration> | default = 30s]
```

### ring_config配置

哈希环的配置，存储位置等

```yaml
kvstore:
  # The backend storage to use for the ring. Supported values are
  # consul, etcd, inmemory
  store: <string>

  # The prefix for the keys in the store. Should end with a /.
  [prefix: <string> | default = "collectors/"]

  # Configuration for a Consul client. Only applies if store
  # is "consul"
  consul:
    # The hostname and port of Consul.
    [host: <string> | duration = "localhost:8500"]

    # The ACL Token used to interact with Consul.
    [acltoken: <string>]

    # The HTTP timeout when communicating with Consul
    [httpclienttimeout: <duration> | default = 20s]

    # Whether or not consistent reads to Consul are enabled.
    [consistentreads: <boolean> | default = true]

  # Configuration for an ETCD v3 client. Only applies if
  # store is "etcd"
  etcd:
    # The ETCD endpoints to connect to.
    endpoints:
      - <string>

    # The Dial timeout for the ETCD connection.
    [dial_tmeout: <duration> | default = 10s]

    # The maximum number of retries to do for failed ops to ETCD.
    [max_retries: <int> | default = 10]

# The heartbeat timeout after which ingesters are skipped for
# reading and writing.
[heartbeat_timeout: <duration> | default = 1m]

# The number of ingesters to write to and read from. Must be at least
# 1.
[replication_factor: <int> | default = 3]
```

# storage_config后端存储配置

如果使用文件系统存储

```yaml
# Configures storing index in BoltDB. Required fields only
# required when boltdb is present in config.
boltdb:
  # Location of BoltDB index files.
  directory: <string>

# Configures storing the chunks on the local filesystem. Required
# fields only required when filesystem is present in config.
filesystem:
  # Directory to store chunks in.
  directory: <string>
```

# schema_config

用来说明日志信息和后端的存储的关系

```yaml
# The date of the first day that index buckets should be created. Use
# a date in the past if this is your only period_config, otherwise
# use a date when you want the schema to switch over.
[from: <daytime>]

# store and object_store below affect which <storage_config> key is
# used.

# Which store to use for the index. Either cassandra, bigtable, dynamodb, or
# boltdb
store: <string>

# Which store to use for the chunks. Either gcs, s3, inmemory, filesystem,
# cassandra. If omitted, defaults to same value as store.
[object_store: <string>]

# The schema to use. Set to v9 or v22.
schema: <string>

# Configures how the index is updated and stored.
index:
  # Table prefix for all period tables.
  prefix: <string>
  # Table period.
  [period: <duration> | default = 168h]
  # A map to be added to all managed tables.
  tags:
    [<string>: <string> ...]

# Configured how the chunks are updated and stored.
chunks:
  # Table prefix for all period tables.
  prefix: <string>
  # Table period.
  [period: <duration> | default = 168h]
  # A map to be added to all managed tables.
  tags:
    [<string>: <string> ...]

# How many shards will be created. Only used if schema is v22.
[row_shards: <int> | default = 16]
```

todo 更多配置实践