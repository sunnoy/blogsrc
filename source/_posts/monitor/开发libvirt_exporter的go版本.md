---
title: 开发libvirt_exporter的go版本
date: 2018-11-21 12:14:28
tags:
- prometheus
- monitor
---

# 开发libvirt_exporter的go版本

## 环境准备

所需要用到的材料

- go环境 https://golangtc.com/download
- libvirt api 文档 https://libvirt.org/html/index.html
- libvirt_exporter fork 库 https://github.com/kumina/libvirt_exporter

<!--more-->

### 环境搭建

#### go环境

环境变量设置

```bash
echo "export GOROOT=/home/go" >>  /etc/profile
echo "export PATH=$GOROOT/bin:$PATH" >>  /etc/profile
echo "export GOPATH=/home/libvirt_exporter" >>  /etc/profile
source /etc/profile

#安装libvirt develo包
yum install libvirt-devel -y
```

依赖下载

```bash
cd /home/libvirt_exporter/src/prometheus-libvirt-exporter
#该命令会在目录/home/libvirt_exporter/src内下载所需依赖
go get

go build
```

#### 本地环境

使用[GoLand](https://www.jetbrains.com/go/download/)插件`remote host access`

![remote](https://qiniu.li-rui.top/remote.png)

使用

![chaj](https://qiniu.li-rui.top/chaj.png)

## 调用流程

### main函数

主要实例化`NewLibvirtExporter`

```go
func main() {
	var (
		listenAddress = flag.String("web.listen-address", ":9000", "Address to listen on for web interface and telemetry.")
		metricsPath   = flag.String("web.telemetry-path", "/metrics", "Path under which to expose metrics.")
		libvirtURI    = flag.String("libvirt.uri", "/var/run/libvirt/libvirt-sock", "Libvirt URI from which to extract metrics.")
	)
	flag.Parse()

    //将二进制接收的参数传递给函数NewLibvirtExporter后就传递给prometheus.MustRegister函数
	exporter, err := NewLibvirtExporter(*libvirtURI)
	if err != nil {
		panic(err)
	}
	prometheus.MustRegister(exporter)

	http.Handle(*metricsPath, prometheus.Handler())
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte(`
			<html>
			<head><title>Libvirt Exporter</title></head>
			<body>
			<h1>Libvirt Exporter</h1>
			<p><a href='` + *metricsPath + `'>Metrics</a></p>
			</body>
			</html>`))
	})
	log.Fatal(http.ListenAndServe(*listenAddress, nil))

}
```

### NewLibvirtExporter函数

该函数会返回连接libvirtd的`LibvirtExporter`结构体

```go
func NewLibvirtExporter(uri string) (*LibvirtExporter, error) {
	return &LibvirtExporter{
		uri: uri,
	}, nil
}
```

该结构体会让两个函数使用`Describe`和`Collect`

### Collect函数

Collect函数会调用`CollectFromLibvirt`来从libvirtd获取索要查询的信息

```go
func (e *LibvirtExporter) Collect(ch chan<- prometheus.Metric) {
	CollectFromLibvirt(ch, e.uri)
}

```

### CollectFromLibvirt函数


CollectDomain会去连接libvirtd，然后对每一个domain调用CollectDomain函数

```go
func CollectFromLibvirt(ch chan<- prometheus.Metric, uri string) error {
	conn, err := net.DialTimeout("unix", uri, 5*time.Second)
	if err != nil {
		log.Fatalf("failed to dial libvirt: %v", err)
		return err
	}
	defer conn.Close()

...

	for _, domain := range domains {
		err = CollectDomain(ch, l, &domain)
		//sb code delete
		//l.DomainShutdown(domain)
		//domain.Free()
		if err != nil {
			log.Fatalf("failed to Collect domain: %v", err)
			return err
		}
	}
```

###　CollectDomain函数

该函数用来将从libvirtd中获取的信息加入到`ch`中

```go
	xmlDesc, err := l.DomainGetXMLDesc(*domain, 0)
	if err != nil {
		log.Fatalf("failed to DomainGetXMLDesc: %v", err)
		return err
	}
	var libvirtSchema libvirt_schema.Domain
	err = xml.Unmarshal([]byte(xmlDesc), &libvirtSchema)
	if err != nil {
		log.Fatalf("failed to Unmarshal domains: %v", err)
		return err
	}

	domainName := domain.Name
	instanceName := libvirtSchema.Metadata.NovaInstance.Name
	instanceId := libvirtSchema.UUID
	host, err := l.ConnectGetHostname()
	if err != nil {
		log.Fatalf("failed to get hostname: %v", err)
		return err
	}

	rState, rmaxmem, rmemory, rvirCpu, rcputime, err := l.DomainGetInfo(*domain)
	ch <- prometheus.MustNewConstMetric(
		libvirtDomainState,
		prometheus.GaugeValue,
		float64(rState),
		domainName, instanceName, instanceId, domainState[libvirt_schema.DomainState(rState)], host)

```

### Describe

将metric的元数据进行渲染

```go
ch <- libvirtDomainBlockCapacity
```

## 代码编写

###　首先引入包

```go
import (
	"encoding/xml"
	"log"
	"net/http"
	"os"

	"github.com/libvirt/libvirt-go"
	"gopkg.in/alecthomas/kingpin.v2"
	"github.com/prometheus/client_golang/prometheus"

	"github.com/kumina/libvirt_exporter/libvirt_schema"
)
```

### 定义出metric输出格式

定义出后效果

```bash
# HELP libvirt_domain_info_maximum_memory_bytes Maximum allowed memory of the domain, in bytes.
# TYPE libvirt_domain_info_maximum_memory_bytes gauge
libvirt_domain_info_maximum_memory_bytes{domain="test02"} 1.073741824e+09
```

定义代码

```go
var (
	libvirtUpDesc = prometheus.NewDesc(
		prometheus.BuildFQName("libvirt", "", "up"),
		"Whether scraping libvirt's metrics was successful.",
		nil,
		nil)

	libvirtDomainInfoMaxMemDesc = prometheus.NewDesc(
        //此处定义metric name 结果为 libvirt_domain_info_maximum_memory_bytes
        prometheus.BuildFQName("libvirt", "domain_info", "maximum_memory_bytes"),
        //此处为HELP信息 结果为 
        //# HELP libvirt_domain_info_maximum_memory_bytes Maximum allowed memory of the domain, in bytes.
        "Maximum allowed memory of the domain, in bytes.",
        //此处显示标签
		[]string{"domain"},
        nil)
        
)
```

详细代码[见](https://github.com/sunnoy/prometheus-libvirt-exporter)

