---
title: opentrace介绍
date: 2019-07-13 12:12:28
tags:
- trace
---

# 为什么需要分布式跟踪？

单体服务到微服务的演进

![wfu](https://qiniu.li-rui.top/wfu.png)

微服务的复杂性决定了分布式跟踪的必要性

<!--more-->

# opentrace介绍

opentrace是一套标准，[详细见标准规范](https://github.com/opentracing-contrib/opentracing-specification-zh/blob/master/specification.md)

## 基础概念

### Trace

描述一个分布式系统中的端到端事务，例如来自客户端的一个请求。

### Spa

一个具有名称和时间长度的操作，例如一个REST调用或者数据库操作等。Span是分布式调用跟踪的最小跟踪单位，一个Trace由多段Span组成。

### Span context
分布式调用跟踪的上下文信息，包括Trace id，Span id以及其它需要传递到下游服务的内容。一个Opentracing的实现需要将Span context通过某种序列化机制(Wire Protocol)在进程边界上进行传递，以将不同进程中的Span关联到同一个Trace上。这些Wire Protocol可以是基于文本的，例如HTTP header，也可以是二进制协议。

## 数据结构

```bash
––|–––––––|–––––––|–––––––|–––––––|–––––––|–––––––|–––––––|–> time

 [Span A···················································]
   [Span B··············································]
      [Span D··········································]
    [Span C········································]
         [Span E·······]        [Span F··] [Span G··] [Span H··]
```

- name: Span所代表的操作名称，例如REST接口对应的资源名称。

- Start timestamp: Span所代表操作的开始时间

- Finish timestamp: Span所代表的操作的的结束时间

- Tags：一系列标签，每个标签由一个key value键值对组成。该标签可以是任何有利于调用分析的信息，例如方法名，URL等。

- SpanContext：用于跨进程边界传递Span相关信息，在进行传递时需要结合一种序列化协议（Wire Protocol）使用。

- References：该Span引用的其它关联Span，主要有两种引用关系，Childof和FollowsFrom。

    - Childof：最常用的一种引用关系，表示Parent Span和Child Span之间存在直接的依赖关系。例PRC服务端Span和RPC客户端Span，或者数据库SQL插入Span和ORM Save动作Span之间的关系。

    - FollowsFrom：如果Parent Span并不依赖Child Span的执行结果，则可以用FollowsFrom表示。例如网上商店购物付款后会向用户发一个邮件通知，但无论邮件通知是否发送成功，都不影响付款成功的状态，这种情况则适用于用FollowsFrom表示。

## 跨服务如何传播

SpanContext是Opentracing中一个让人比较迷惑的概念。在Opentracing的概念模型中提到SpanContext用于跨进程边界传递分布式调用的上下文。但实际上Opentracing只定义一个SpanContext的抽象接口，该接口封装了分布式调用中一个Span的相关上下文内容，包括该Span所属的Trace id，Span id以及其它需要传递到downstream服务的信息。SpanContext自身并不能实现跨进程的上下文传递，需要由Tracer（Tracer是一个遵循Opentracing协议的实现，如Jaeger，Skywalking的Tracer）将SpanContext序列化后通过Wire Protocol传递到下一个进程中，然后在下一个进程将SpanContext反序列化，得到相关的上下文信息，以用于生成Child Span。

为了为各种具体实现提供最大的灵活性，Opentracing只是提出了跨进程传递SpanContext的要求，并未规定将SpanContext进行序列化并在网络中传递的具体实现方式。各个不同的Tracer可以根据自己的情况使用不同的Wire Protocol来传递SpanContext。

在基于HTTP协议的分布式调用中，通常会使用HTTP Header来传递SpanContext的内容。常见的Wire Protocol包含Zipkin使用的b3 HTTP header，Jaeger使用的uber-trace-id HTTP Header,LightStep使用的”x-ot-span-context” HTTP Header等。Istio/Envoy支持b3 header和x-ot-span-context header,可以和Zipkin,Jaeger及LightStep对接。其中jaeger HTTP header的示例如下：

```bash
uber-trace-id: 
```

# 参考资料

- https://mp.weixin.qq.com/s/2anscRAQB0PHAf0_npslBw

