---
title: Prometheus server使用
date: 2018-11-12 12:14:28
tags:
- prometheus
- monitor
---

# Prometheus server使用

Prometheus是一个开源监控和报警平台，为[Cloud Native Computing Foundation](https://www.cncf.io/)在Kubernetes之后的第二个毕业项目。

Prometheus包含组件有：

- Prometheus server 用来向远程主机pull和存储数据
- 信息采集包含多种方式
- alertmanager报警模块
- 服务发现
- 通过PromQL查询语言来展示数据

![architecturprometheuse](https://qiniu.li-rui.top/architecturprometheuse.png)

<!--more-->
远程主机上的CLIENT LIBRARIES和EXPORTERS来采集主机，其中EXPORTERS为特定应用或者服务打包好的相关工具包，里面的监控项目较为固定，比如MySQL server exporter。CLIENT LIBRARIES则是更灵活定义所监控的项目，需要自行开发继承到自己的项目中。

Prometheus server通过pull的方式向远程采集数据后存储，如果远程服务器生命周期比较短就通过push操作把数据放到push gateway进行缓存，push gateway等待Prometheus server来pull数据。远程主机也可以使用服务发现来配置。

Prometheus server发现有报警情况就会将事件push到alertmanager模块，alertmanager模块会依据配置进行相关通知

web界面和其他的向获取数据的模块则通过查询语言PromQL来向Prometheus server请求相关数据

## Prometheus server

### 数据如何存储

使用时间序列数据库，存储形式

```bash
<--------------- metric ---------------------><-timestamp -><-value->
http_request_total{status="200", method="GET"}@1434417560938 => 94355
http_request_total{status="200", method="GET"}@1434417561287 => 94334

http_request_total{status="404", method="GET"}@1434417560938 => 38473
http_request_total{status="404", method="GET"}@1434417561287 => 38544

```

## 基本使用

我们将会对一台宿主机上的libvirt、ceph、宿主机等进行监控

### libvirt

Prometheus和libvirt均没有为对方提供接口进行对接，这里使用[Kumina](https://www.kumina.nl)提供的开源[libvirt_exporter](https://github.com/kumina/libvirt_exporter)，libvirt_exporter使用了libvirt的go版本api。

在容器内编译完成后使用`docker cp`把编译好的文件拉出来，对接libvirt服务api有很多方式[详见](https://libvirt.org/remote.html)。这里使用本地sock方式

```bash
nohup ./libvirt_exporter --libvirt.uri="qemu+unix:///system?socket=/var/run/libvirt/libvirt-sock" &
```

### prometheus server

#### 定义配置文件

```bash
global:
  scrape_interval:     5s # By default, scrape targets every 15 seconds.

  # Attach these labels to any time series or alerts when communicating with
  # external systems (federation, remote storage, Alertmanager).
  external_labels:
    monitor: 'codelab-monitor'
alerting:
  alertmanagers:
  - scheme: http
    static_configs:
    - targets:
      - "192.168.66.112:9093"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s

    static_configs:
      - targets: ['192.168.66.112:9090']

  - job_name: 'node'

    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s

    static_configs:
      - targets: ['192.168.66.112:9100']

  - job_name: 'libvirt'

    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s

    static_configs:
      - targets: ['192.168.66.112:9177']

  - job_name: 'ceph'

    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s

    static_configs:
      - targets: ['192.168.66.112:9283']
```

#### 启动prometheus服务端容器

Prometheus默认以账户`nobody`运行，这里强制使用root账户运行

```bash
docker run -u root:root --name prom --rm \
-v /home/prom/data:/home \
--net="host" \
prom/prometheus \
--web.enable-lifecycle \
--storage.tsdb.path=/home \
--storage.tsdb.retention=180d \
--config.file=/home/prometheus.yml
```

#### 直接二进制启动

```bash
prometheus --web.listen-address=0.0.0.0:9091 \ --config.file=/root/test/prom/prometheus.yml \
--storage.tsdb.path=/root/test/prom-data \
--web.enable-lifecycle \
--storage.tsdb.retention=180d \
--web.console.libraries=/root/prometheus-2.5.0.linux-amd64/console_libraries \
--web.console.templates=/root/prometheus-2.5.0.linux-amd64/consoles
```

### grafana

使用grafana进行图表展示

```bash
docker run -d --name grafana -p 3000:3000 --net="host" grafana/grafana
```

### node-exporter

监控宿主机上的系统参数

```bash
docker run --rm --name node  \
-v "/proc:/host/proc:ro" \
-v "/sys:/host/sys:ro" \
-v "/:/rootfs:ro" \
--net="host"   prom/node-exporter \
--collector.mountstats \
--collector.filesystem.ignored-fs-types="^(autofs|binfmt_misc|cgroup|configfs|debugfs|devpts|devtmpfs|fusectl|hugetlbfs|mqueue|overlay|proc|procfs|pstore|rpc_pipefs|securityfs|sysfs|tracefs)$" \
--collector.processes \


#二进制启动
./node_exporter --collector.mountstats \
--collector.filesystem.ignored-fs-types="^(autofs|binfmt_misc|cgroup|configfs|debugfs|devpts|devtmpfs|fusectl|hugetlbfs|mqueue|overlay|proc|procfs|pstore|rpc_pipefs|securityfs|sysfs|tracefs)$" \
--collector.diskstats 

```


