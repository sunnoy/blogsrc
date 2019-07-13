---
title: opentrace实现
date: 2019-07-13 12:12:28
tags:
- trace
---

# 服务实现opentrace

主要有两种方法

- 应用内引入相关的client实现
- 通过服务网格实现
<!--more-->
# client实现

这里使用jaeger来实现，语言使用go

![architecture-v1](https://qiniu.li-rui.top/architecture-v1.jpg)

测试服务调用图

![opentrace](https://qiniu.li-rui.top/opentrace.jpg)

## 开启jaeger服务

```bash
docker run -d --name jaeger \
  -e COLLECTOR_ZIPKIN_HTTP_PORT=9411 \
  -p 5775:5775/udp \
  -p 6831:6831/udp \
  -p 6832:6832/udp \
  -p 5778:5778 \
  -p 16686:16686 \
  -p 14268:14268 \
  -p 9411:9411 \
  jaegertracing/all-in-one:1.13
```

## 需要引入的包

- github.com/opentracing/opentracing-go
- github.com/uber/jaeger-client-go

这里只列出关键代码，完成代码见[项目地址](https://github.com/sunnoy/JaegerDemoGo)

## 初始化tracer

### 引入jaeger配置

```go
//JaegerDemoGo/lib/tracing/init.go
	cfg := &config.Configuration{
		Sampler: &config.SamplerConfig{
			Type:  "const",
			Param: 1,
		},
		Reporter: &config.ReporterConfig{
			CollectorEndpoint: "http://172.18.247.201:14268/api/traces",
			LogSpans:          true,
		},
	}
```

### 初始化tracer

```go
//lib/tracing/init.go
tracer, closer, err := cfg.New(service, config.Logger(jaeger.StdLogger))
```

## 初始服务配置

下面代码均在`client/hello.go`里面

### 初始化trace

```go
tracer, closer := tracing.Init("begin")
opentracing.SetGlobalTracer(tracer)
```

### 创建root span

```go
span := tracer.StartSpan("hello-opname")
```

### 创建span context

```go
ctx := opentracing.ContextWithSpan(context.Background(), span)
```

### 创建 子span

```go
span, _ := opentracing.StartSpanFromContext(ctx, "callservice1-opname")
```

### 将uber-trace-id http头部注入到http请求头部

```go 
	span.Tracer().Inject(
		span.Context(),
		opentracing.HTTPHeaders,
		opentracing.HTTPHeadersCarrier(req.Header),
	)
```

## 被调用服务

### 从http请求中获取span信息

```go
//service1/service1.go
spanCtx, _ := tracer.Extract(opentracing.HTTPHeaders, opentracing.HTTPHeadersCarrier(r.Header))

//创建一个span，后面接上一些选项，这个span不是 root span
span := tracer.StartSpan("service1-opname", ext.RPCServerOption(spanCtx))
```

# 服务网格实现

主要通过转发平面工具Envoy来实现：

Istio/Envoy为微服务提供了开箱即用的分布式调用跟踪功能。在安装了Istio和Envoy的微服务系统中，Envoy会拦截服务的入向和出向请求，为微服务的每个调用请求自动生成调用跟踪数据。通过在服务网格中接入一个分布式跟踪的后端系统，例如zipkin或者Jaeger，就可以查看一个分布式请求的详细内容，例如该请求经过了哪些服务，调用了哪个REST接口，每个REST接口所花费的时间等。

需要指出的是，应用代码中需要将收到的上游HTTP请求中的uber-trace-id header拷贝到其向下游发起的HTTP请求的header中，以将调用跟踪上下文传递到下游服务。

# 参考资料

- https://yq.aliyun.com/articles/514488?utm_content=m_43363
- https://developer.qiniu.com/insight/manual/5092/tracingdemo-jaeger-go