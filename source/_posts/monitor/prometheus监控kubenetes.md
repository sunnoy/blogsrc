---
title: prometheus监控kubenetes
date: 2019-4-19 12:14:28
tags:
- prometheus
- monitor
---

# 服务发现

服务发现就是服务提供者和服务使用者之间的服务代理。服务提供者去服务代理注册自己的服务，服务使用者去服务代理拿到可用的服务提供者列表，然后通过某种机制去选择一个服务提供者。

![服务发现](https://qiniu.li-rui.top/服务发现.jpg)

<!--more-->



# kubernetes提供的服务发现

提供的服务发现主要是通过API rest 来提供，包含下面几个role

- node
- service
- pod
- endpoints
- ingress

本质上是kubernetes中内置了一些exporer，prometheus可以通过http请求抓取数据。
prometheus会通过不同的role的服务发现来获取抓取数据的连接。

抓取到的数据是time series格式，用于存放到prometheus中的时间序列数据库。抓取到的数据中包含大量以 __meta 开头的标签，用于提供相关对象中的字段信息

下面分别说明这些role提供的数据类型，

## node

- __meta_kubernetes_node_name
- __meta_kubernetes_node_label_<labelname>
- __meta_kubernetes_node_labelpresent_<labelname>
- __meta_kubernetes_node_annotation_<annotationname>
- __meta_kubernetes_node_annotationpresent_<annotationname>
- __meta_kubernetes_node_address_<address_type>

通过看后面可发现，node对象是集群资源，不是namespace资源，因此node类型的服务发现中没有 __meta_kubernetes_namespace 标签

## service

- __meta_kubernetes_namespace
- __meta_kubernetes_service_annotation_<annotationname>
- __meta_kubernetes_service_annotationpresent_<annotationname>
- __meta_kubernetes_service_cluster_ip
- __meta_kubernetes_service_external_name
- __meta_kubernetes_service_label_<labelname>
- __meta_kubernetes_service_labelpresent_<labelname>
- __meta_kubernetes_service_name
- __meta_kubernetes_service_port_name
- __meta_kubernetes_service_port_number
- __meta_kubernetes_service_port_protocol

## pod

- __meta_kubernetes_namespace
- __meta_kubernetes_pod_name
- __meta_kubernetes_pod_ip
- __meta_kubernetes_pod_label_<labelname>
- __meta_kubernetes_pod_labelpresent_<labelname>
- __meta_kubernetes_pod_annotation_<annotationname>
- __meta_kubernetes_pod_annotationpresent_<annotationname>
- __meta_kubernetes_pod_container_name
- __meta_kubernetes_pod_container_port_name
- __meta_kubernetes_pod_container_port_number
- __meta_kubernetes_pod_container_port_protocol
- __meta_kubernetes_pod_ready
- __meta_kubernetes_pod_phase
- __meta_kubernetes_pod_node_name
- __meta_kubernetes_pod_host_ip
- __meta_kubernetes_pod_uid
- __meta_kubernetes_pod_controller_kind
- __meta_kubernetes_pod_controller_name

## endpoints

- __meta_kubernetes_namespace
- __meta_kubernetes_endpoints_name
- __meta_kubernetes_endpoint_ready
- __meta_kubernetes_endpoint_port_name
- __meta_kubernetes_endpoint_port_protocol
- __meta_kubernetes_endpoint_address_target_kind
- __meta_kubernetes_endpoint_address_target_name

如果一个endpoint属于某个service内，这个endpoint所属的service的标签也会被附到样本数据库上抓取过来

如果这个endpoint的后端是pod，相应的pod的标签也会被抓取到

也就是说，如果endpoint标签有四种类型

- endpoint后端为外部服务，仅仅创建endpoint对象那么标签只有endpoint的
- endpoint后端为外部服务，创建了endpoint和与之相关的service对象，那么标签有endpoint和service的
- endpoint后端为pod，那么endpoint的标签有endpoint和pod的
- endpoint后端为pod，并且创建了service对象，那么endpoint的标签有endpoint和service以及pod的

## ingress

- __meta_kubernetes_namespace
- __meta_kubernetes_ingress_name
- __meta_kubernetes_ingress_label_<labelname>
- __meta_kubernetes_ingress_labelpresent_<labelname>
- __meta_kubernetes_ingress_annotation_<annotationname>
- __meta_kubernetes_ingress_annotationpresent_<annotationname>
- __meta_kubernetes_ingress_scheme
- __meta_kubernetes_ingress_path

# prometheus中使用

## 配置字段

prometheus中提供了 kubernetes_sd_config 的配置字段，主要的配置字段如下

```yaml
# 留空就是默认在kubernetes中部署
[ api_server: <host> ]

# 通过什么样的role来使用服务发现
role: <role>

# 下面主要是认证信息，用下面的认证信息去访问API server

# http基本认证
basic_auth:
  [ username: <string> ]
  [ password: <secret> ]
  [ password_file: <string> ]

# token信息认证
[ bearer_token: <secret> ]

# 使用token文件认证
[ bearer_token_file: <filename> ]

# 使用代理访问API server
[ proxy_url: <string> ]

# 配置访问API server的ssl证书
tls_config:
  [ <tls_config> ]

# 在哪个namespace中抓取数据，留空就是所有namespace
namespaces:
  names:
    [ - <string> ]
```

# 详细使用

我们基于prometheus提供的[示例配置文件](https://github.com/prometheus/prometheus/blob/release-2.9/documentation/examples/prometheus-kubernetes.yml)以及prometheus chart的[配置文件](https://github.com/helm/charts/blob/master/stable/prometheus/values.yaml)来说明

对接kubernetes的服务发现需要下面几个步骤：

- prometheus获取 API server 的连接信息
- 通过APIserver和相应的role获取服通过务发现得到的采集数据服务地址
- 对采集服务的服务地址做认证配置
- 采集服务收到prometheus经过认证的请求后进行授权过程

通过上面的配置字段我们知道，如果prometheus部署在kubernetes中就不用配置字段 api_server 的值。但是要配置访问API server认证的信息。访问API server中相关对象的信息需要通过授权来完成，在prometheus chart中默认会创建RBAC的授权配置yaml文件。

下面会分别介绍如何在prometheus中通过 relabel_configs 来抓取不同类型资源的监控数据

## API server

### 集群内获取服务

二进制部署中，API server是宿主机的服务，也即是说并没有在集群service对象内。为了让集群内的pod仿访问到API server我们需要将宿主机山的6443端口用endpoint的方式整到集群里面

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  creationTimestamp: "2019-03-23T04:26:10Z"
  name: kubernetes
  namespace: default
  resourceVersion: "8"
subsets:
- addresses:
  - ip: 10.9.1.xxx
  ports:
  - name: https
    port: 6443
    protocol: TCP

---
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: "2019-03-23T04:26:10Z"
  labels:
    component: apiserver
    provider: kubernetes
  name: kubernetes
  namespace: default
spec:
  clusterIP: 10.68.0.1
  ports:
  - name: https
    port: 443
    protocol: TCP
    targetPort: 6443
  sessionAffinity: None
  type: ClusterIP
```

### 进行服务发现

因为我们要获取抓取数据的连接和端口，因此要获取API server需要使用 endpoint role

但又因为kubernetes提供的服务发现中的role都是将所有该类型的服务提供者都获取到，因此我们需要通过 prometheus 中的 relabel_configs 配置字段和适当的正则表达式去筛选出符合要求的服务提供者

```yaml
      - job_name: 'kubernetes-apiservers'
        # 获取endpopit对象
        kubernetes_sd_configs:
          - role: endpoints
        # API server需要https访问
        scheme: https
        # 引入每个pod中的ca证书，用来完成 ssl认证
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
          # 如果是对kubernetes使用的是自签证书，可能需要将跳过安全认证打开
          insecure_skip_verify: true
        # 引入token文件 用来和serviceaccount绑定进行授权对集群内资源信息的获取
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

        relabel_configs:
          #只保留与kubernetes相关的标签
          - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
            action: keep
            #并对标签的值进行过滤，进一步筛选
            regex: default;kubernetes;https
```

## node

kubelet内置的模块会提供prometeus来采集监控数据，因此访问kubelet有两个方法

- 直接访问kubelet的10250 https端口
- 通过APIserver代理去访问 相关认证配置复杂不推荐

### 通过API server代理去获取

同API server代理访问可以避免node上的防火墙阻隔

```yaml
- job_name: 'kubernetes-nodes'
  scheme: https
  tls_config:
    ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
  bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
  
  # 此处获取node role信息
  kubernetes_sd_configs:
  - role: node

  relabel_configs:
    #此处的labelmap用来对(.+)筛选出来的内容做标签映射
    # 标签 __meta_kubernetes_node_label_beta_kubernetes_io_arch="amd64" 会映射为 beta_kubernetes_io_arch="amd64"
  - action: labelmap
    regex: __meta_kubernetes_node_label_(.+)
    #将内置的__address__的标签值替换成kubernetes.default.svc:443
  - target_label: __address__
    replacement: kubernetes.default.svc:443
    #获取到__meta_kubernetes_node_name的值，然后通过正则表达式(.+)获取该值，因为只有一个正则分组，所以 $1 就是标签__meta_kubernetes_node_name的值
  - source_labels: [__meta_kubernetes_node_name]
    regex: (.+)
    # 进行抓取的url路径
    target_label: __metrics_path__
    replacement: /api/v1/nodes/${1}/proxy/metrics
```

## 容器

容器主要通过kubelet的cadvisor模块获取，因此他的方式和node类似。

```yaml
- job_name: 'kubernetes-cadvisor'

  scheme: https

  tls_config:
    ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
  bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

  kubernetes_sd_configs:
  - role: node

  relabel_configs:
  - action: labelmap
    regex: __meta_kubernetes_node_label_(.+)
  - target_label: __address__
    replacement: kubernetes.default.svc:443
  - source_labels: [__meta_kubernetes_node_name]
    regex: (.+)
    target_label: __metrics_path__
    #这里和node不一样
    replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor
```

## service endpoints

### 服务发现配置

如果没有对endpoint前面的服务进行配置，prometheus服务发现并不能发现他们。endpoint要想被服务发现发现自己，需要在前面的service对象中的annotation配置一些信息，这些信息包含

- prometheus.io/scrape 布尔值 是否需允许服务发现找到自己
- prometheus.io/scheme https或者http 默认http
- prometheus.io/path 默认为 /metrics
- prometheus.io/port 默认为service中的port 

看个示例coredns服务

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    prometheus.io/port: "9153"
    prometheus.io/scrape: "true"
```
在进行服务发现的时候，上面添加的注释会映射为相应的服务发现标签

```yaml
__meta_kubernetes_service_annotation_prometheus_io_scrape
__meta_kubernetes_service_annotation_prometheus_io_scheme
__meta_kubernetes_service_annotation_prometheus_io_path
__meta_kubernetes_service_annotation_prometheus_io_port
```

但是在prometeus中以 __meta 开头的标签不会放到样本数据里面去，因此需要我们将 __meta 开头的标签通过 relabel_configs 字段进行标签转换映射等一系列操作。

通过这些操作可以对服务发现的资源进行一些列的处理，来更好的为我们监控使用。

### endpoint配置

```yaml
      - job_name: 'kubernetes-service-endpoints'

        #指定role为endpoint
        kubernetes_sd_configs:
          - role: endpoints

        relabel_configs:
         # 获取service中annotation prometheus.io/scrape信息，
         #只有值为 true 时才将该信息的标签保留
          - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
            action: keep
            regex: true
        #如果是https就将__scheme__加上https
          - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
            action: replace
            target_label: __scheme__
            regex: (https?)
        #抓取路径映射 
          - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
            action: replace
            target_label: __metrics_path__
            regex: (.+)
          # 获取抓取地址和端口
          #引入了两个标签  
          - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
            action: replace
            target_label: __address__
            #这里获取到地址端口号
            regex: ([^:]+)(?::\d+)?;(\d+)
            replacement: $1:$2
          #这里是将标签__meta_kubernetes_service_label_后面的内容map为新的标签，标签的值不变
          - action: labelmap
            regex: __meta_kubernetes_service_label_(.+)
          - source_labels: [__meta_kubernetes_namespace]
            action: replace
            target_label: kubernetes_namespace
          - source_labels: [__meta_kubernetes_service_name]
            action: replace
            target_label: kubernetes_name
          - source_labels: [__meta_kubernetes_pod_node_name]
            action: replace
            target_label: kubernetes_node
```

## service 发现

这个是通过 blackbox exporter来实现的，需要在service注释中添加

- prometheus.io/probe: true

这样就可以通过 service role发现

```yaml
    - job_name: 'kubernetes-services'

        metrics_path: /probe
        #配置module模块
        params:
          module: [http_2xx]

        kubernetes_sd_configs:
          - role: service

        relabel_configs:
          - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_probe]
            action: keep
            regex: true
          - source_labels: [__address__]
            target_label: __param_target
          - target_label: __address__
            replacement: blackbox
          - source_labels: [__param_target]
            target_label: instance
          - action: labelmap
            regex: __meta_kubernetes_service_label_(.+)
          - source_labels: [__meta_kubernetes_namespace]
            target_label: kubernetes_namespace
          - source_labels: [__meta_kubernetes_service_name]
            target_label: kubernetes_name
```

## pod发现

pod发现需要在pod中添加注释

- prometheus.io/scrape
- prometheus.io/path
- prometheus.io/port 默认为9102

```yaml
      - job_name: 'kubernetes-pods'

        kubernetes_sd_configs:
          - role: pod

        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
            action: keep
            regex: true
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
            action: replace
            target_label: __metrics_path__
            regex: (.+)
          - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
            action: replace
            regex: ([^:]+)(?::\d+)?;(\d+)
            replacement: $1:$2
            target_label: __address__
          - action: labelmap
            regex: __meta_kubernetes_pod_label_(.+)
          - source_labels: [__meta_kubernetes_namespace]
            action: replace
            target_label: kubernetes_namespace
          - source_labels: [__meta_kubernetes_pod_name]
            action: replace
            target_label: kubernetes_pod_name
```

## ingress 发现

ingress监控是基于blackbox exporter的



```yaml

- job_name: 'kubernetes-ingresses'

  metrics_path: /probe
  params:
    module: [http_2xx]

  kubernetes_sd_configs:
  - role: ingress

  relabel_configs:

  #通过在ingress中添加定制标签来选择性的采集ingress对象
  - source_labels: [__meta_kubernetes_ingress_annotation_example_io_should_be_probed]
    action: keep
    regex: true


  - source_labels: [__meta_kubernetes_ingress_scheme,__address__,__meta_kubernetes_ingress_path]
    regex: (.+);(.+);(.+)
    replacement: ${1}://${2}${3}
    target_label: __param_target
  - target_label: __address__
    #这里要改为blackbox-exporter服务的服务名称和端口号
    replacement: blackbox-exporter.example.com:9115
  - source_labels: [__param_target]
    target_label: instance
  - action: labelmap
    regex: __meta_kubernetes_ingress_label_(.+)
  - source_labels: [__meta_kubernetes_namespace]
    target_label: kubernetes_namespace
  - source_labels: [__meta_kubernetes_ingress_name]
    target_label: kubernetes_name

```











