---
layout:     post
title:      Springcloud系列总结8
subtitle:   Zipkin
date:       2021-11-20
author:     Yikun Chen
header-img: img/background/header.png
catalog: true
tags:
    - springcloud
    - zipkin
---


# Springcloud系列总结8

**Zipkin**
--

## 基本概念

Zipkin 是一个基于 Java 开发的、开源的、分布式实时数据跟踪系统（Distributed Tracking System）。它采集有助于解决服务架构中延迟问题的实时数据。Zipkin 主要功能是聚集来自各个异构系统的实时监控数据。

如果日志文件中有跟踪 ID，则可以直接跳至该跟踪 ID。 否则，您可以基于属性进行查询，例如服务，操作名称，标签和持续时间。 将为您总结一些有趣的数据，例如在服务中花费的时间百分比以及操作是否失败。

Zipkin UI 还提供了一个依赖关系图，该关系图显示了每个应用程序中跟踪了多少个请求。这对于识别聚合行为（包括错误路径或对不赞成使用的服务的调用）很有帮助。

### 设计架构

![picture1](/img/springcloud/zipkin_architecture.png)

ZipKin 可以分为两部分， 一部分是 Zipkin server，用来作为数据的采集存储、数据分析与展示；另一部分是 Zipkin client 是 Zipkin 基于不同的语言及框架封装的一些列客户端工具，这些工具完成了追踪数据的生成与上报功能。

### Zipkin Server

Zipkin Server 主要包括四个模块：

- **Collector** - 负责采集客户端传输的数据。
- **Storage** - 负责存储采集的数据。当前支持 Memory，MySQL，Cassandra，ElasticSearch 等，默认存储在内存中。
- **API（Query）** - 负责查询 Storage 中存储的数据。提供简单的 JSON API 获取数据，主要提供给 web UI 使用。
- **UI** - 提供简单的 web 界面。

Instrumented Client 和 Instrumented Server，是指分布式架构中使用了 Trace 工具的两个应用，Client 会调用 Server 提供的服务，两者都会向 Zipkin 上报 Trace 相关信息。在 Client 和 Server 通过 Transport 上报 Trace 信息后，由 Zipkin 的 Collector 模块接收，并由 Storage 模块将数据存储在对应的存储介质中，然后 Zipkin 提供 API 供 UI 界面查询 Trace 跟踪信息。Non-Instrumented Server，指的是未使用 Trace 工具的 Server，显然它不会上报 Trace 信息。

### Zipkin Client

- **Tracer** - `Tracer` 存在于你的应用中，它负责采集关于已发生操作的实时元数据。它们通常会检测库，因此对于用户是透明的。例如，已检测的 Web 服务器记录它何时接收到请求，以及何时发送响应。收集的跟踪数据称为跨度（Span）。
- **Instrumentation** - Instrumentation 保证了生产环境的安全性和很少的开销。因此，它们仅在内部传播 ID，以告知接收方正在进行追踪。完成的 Span 将通过外部通信告知 Zipkin，类似于应用程序异步报告指标的方式。例如，当跟踪某个操作并且需要发出 http 请求时，会添加一些 header 来传播 ID。header 不用于发送详细信息，例如操作名称。
- **Reporter** - 能够将数据发送到 Zipkin 的检测应用程序中的组件，被称为 Reporter。Reporter 有多种传输方式，可以将跟踪数据发送到 Zipkin 采集器，后者将跟踪数据持久化保存到存储中。稍后，API 会查询存储以向 UI 提供渲染数据。

## 接入方式

### docker安装
```java
docker run -d -p 9411:9411 openzipkin/zipkin
```

### springcloud 接入

zipkin配置类

```java
@Configuration
public class ZipkinConfig {
    //span（一次请求信息或者一次链路调用）信息收集器  
    @Bean  
    public SpanCollector spanCollector() {  
        Config config = HttpSpanCollector.Config.builder()  
                .compressionEnabled(false)// 默认false，span在transport之前是否会被gzipped  
                .connectTimeout(5000)  
                .flushInterval(1)  
                .readTimeout(6000)  
                .build();  
        return HttpSpanCollector.create("http://localhost:9411", config, new EmptySpanCollectorMetricsHandler());  
    }  
      
    //作为各调用链路，只需要负责将指定格式的数据发送给zipkin  
    @Bean  
    public Brave brave(SpanCollector spanCollector){  
        Builder builder = new Builder("service1");//指定serviceName  
        builder.spanCollector(spanCollector);  
        builder.traceSampler(Sampler.create(1));//采集率  
        return builder.build();  
    }  
  
  
    //设置server的（服务端收到请求和服务端完成处理，并将结果发送给客户端）过滤器  
    @Bean  
    public BraveServletFilter braveServletFilter(Brave brave) {  
        BraveServletFilter filter = new BraveServletFilter(brave.serverRequestInterceptor(),  
                brave.serverResponseInterceptor(), new DefaultSpanNameProvider());  
        return filter;  
    }  
      
    //设置client的（发起请求和获取到服务端返回信息）拦截器  
    @Bean  
    public CloseableHttpClient okHttpClient(Brave brave){  
       CloseableHttpClient httpclient = HttpClients.custom()
                .addInterceptorFirst(new BraveHttpRequestInterceptor(brave.clientRequestInterceptor(), new DefaultSpanNameProvider()))
                .addInterceptorFirst(new BraveHttpResponseInterceptor(brave.clientResponseInterceptor()))
                .build();
        return httpclient;  
    }
    
}
```

接口编写

```java
@RestController
public class ZipkinBraveController {
    @Autowired
    private CloseableHttpClient okHttpClient;
    
    @GetMapping("/service1")
    public String myboot() throws Exception {
        Thread.sleep(100);//100ms
        HttpGet get = new HttpGet("http://localhost:81/test");
        CloseableHttpResponse execute = okHttpClient.execute(get);
        /*
         * 1、执行execute()的前后，会执行相应的拦截器（cs,cr）
         * 2、请求在被调用方执行的前后，也会执行相应的拦截器（sr,ss）
         */
        return EntityUtils.toString(execute.getEntity(), "utf-8");
    }
}
```

参考文献
--

[Zipkin 应用指南](https://dunwu.github.io/java-tutorial/javatool/monitor/zipkin.html#docker)

[zipkin教程简单介绍及环境搭建](https://mykite.github.io/2017/04/21/zipkin%E6%95%99%E7%A8%8B%E7%AE%80%E5%8D%95%E4%BB%8B%E7%BB%8D%E5%8F%8A%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA%EF%BC%88%E4%B8%80%EF%BC%89/)

