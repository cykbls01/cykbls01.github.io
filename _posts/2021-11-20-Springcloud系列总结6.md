---
layout:     post
title:      Springcloud系列总结6
subtitle:   Sentinel
date:       2021-11-20
author:     Yikun Chen
header-img: img/background/header.png
catalog: true
tags:
    - springcloud
    - sentinel
---


# Springcloud系列总结6

**Sentinel**
--

## 基本概念

随着微服务的流行，服务和服务之间的稳定性变得越来越重要。Sentinel 是面向分布式服务架构的流量控制组件，主要以流量为切入点，从限流、流量整形、熔断降级、系统负载保护、热点防护等多个维度来帮助开发者保障微服务的稳定性。

Sentinel具有以下特征:

- **丰富的应用场景**： Sentinel 承接了阿里巴巴近 10 年的双十一大促流量的核心场景，例如秒杀（即突发流量控制在系统容量可以承受的范围）、消息削峰填谷、实时熔断下游不可用应用等。
- **完备的实时监控**： Sentinel 同时提供实时的监控功能。您可以在控制台中看到接入应用的单台机器秒级数据，甚至 500 台以下规模的集群的汇总运行情况。
- **广泛的开源生态**： Sentinel 提供开箱即用的与其它开源框架/库的整合模块，例如与 Spring Cloud、Dubbo、gRPC 的整合。您只需要引入相应的依赖并进行简单的配置即可快速地接入 Sentinel。
- **完善的 SPI 扩展点**： Sentinel 提供简单易用、完善的 SPI 扩展点。您可以通过实现扩展点，快速的定制逻辑。例如定制规则管理、适配数据源等。

### 设计架构

![picture1](/img/springcloud/sentinel_structure.png)

在 Sentinel 里面，所有的资源都对应一个资源名称（resourceName），每次资源调用都会创建一个 Entry 对象。Entry 可以通过对主流框架的适配自动创建，也可以通过注解的方式或调用 SphU API 显式创建。Entry 创建的时候，同时也会创建一系列功能插槽（slot chain），这些插槽有不同的职责，例如:

- NodeSelectorSlot 负责收集资源的路径，并将这些资源的调用路径，以树状结构存储起来，用于根据调用路径来限流降级；
- ClusterBuilderSlot 则用于存储资源的统计信息以及调用者信息，例如该资源的 RT, QPS, thread count 等等，这些信息将用作为多维度限流，降级的依据；
- StatisticSlot 则用于记录、统计不同纬度的 runtime 指标监控信息；
- FlowSlot 则用于根据预设的限流规则以及前面 slot 统计的状态，来进行流量控制；
- AuthoritySlot 则根据配置的黑白名单和调用来源信息，来做黑白名单控制；
- DegradeSlot 则通过统计信息以及预设的规则，来做熔断降级；
- SystemSlot 则通过系统的状态，例如 load1 等，来控制总的入口流量；

## 接入方式

### 单独使用

编写测试接口，单独接入业务侵入性过强，改动大。

```java
@RestController
@Slf4j
public class HelloController {

    private static final String RESOURCE_NAME = "hello";

    @RequestMapping(value = "/hello")
    public String hello() {

        Entry entry = null;
        try {
            // 资源名可使用任意有业务语义的字符串，比如方法名、接口名或其它可唯一标识的字符串。
            entry = SphU.entry(RESOURCE_NAME);
            // 被保护的业务逻辑
            String str = "hello world";
            log.info("====="+str);
            return str;
        } catch (BlockException e1) {
            // 资源访问阻止，被限流或被降级
            //进行相应的处理操作
            log.info("block!");
        } catch (Exception ex) {
            // 若需要配置降级规则，需要通过这种方式记录业务异常
            Tracer.traceEntry(ex, entry);
        } finally {
            if (entry != null) {
                entry.exit();
            }
        }
        return null;
    }

    /**
     * 定义流控规则
     */
    @PostConstruct
    private static void initFlowRules(){
        List<FlowRule> rules = new ArrayList<>();
        FlowRule rule = new FlowRule();
        //设置受保护的资源
        rule.setResource(RESOURCE_NAME);
        // 设置流控规则 QPS
        rule.setGrade(RuleConstant.FLOW_GRADE_QPS);
        // 设置受保护的资源阈值
        // Set limit QPS to 20.
        rule.setCount(1);
        rules.add(rule);
        // 加载配置好的规则
        FlowRuleManager.loadRules(rules);
    }
}
```


### 注解接入

配置切面支持+@SentinelResource注解。

```java
@Configuration
public class SentinelAspectConfiguration {

    @Bean
    public SentinelResourceAspect sentinelResourceAspect() {
        return new SentinelResourceAspect();
    }
}
```

```java
@RequestMapping(value = "/findOrderByUserId/{id}")
@SentinelResource(value = "findOrderByUserId",
                  fallback = "fallback",fallbackClass = ExceptionUtil.class,
                  blockHandler = "handleException",blockHandlerClass = ExceptionUtil.class
                 )
public R  findOrderByUserId(@PathVariable("id") Integer id) {
    //ribbon实现
    String url = "http://mall-order/order/findOrderByUserId/"+id;
    R result = restTemplate.getForObject(url,R.class);

    if(id==4){
        throw new IllegalArgumentException("非法参数异常");
    }

    return result;
}
```

配置fallback方法。

```java
public class ExceptionUtil {

    public static R fallback(Integer id,Throwable e){
        return R.error(-2,"===被异常降级啦===");
    }

    public static R handleException(Integer id, BlockException e){
        return R.error(-2,"===被限流啦===");
    }
}
```

### springcloud+控制台接入

配置文件。

```java
server:
  port: 8800

spring:
  application:
    name: mall-user-sentinel-demo
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848

    sentinel:
      transport:
        # 添加sentinel的控制台地址
        dashboard: 127.0.0.1:8080
        # 指定应用与Sentinel控制台交互的端口，应用本地会起一个该端口占用的HttpServer
        # port: 8719
    
#暴露actuator端点   
management:
  endpoints:
    web:
      exposure:
        include: '*'
```

在sentinel控制台中设置流控规则。

- **资源名**:  接口的API   
- **针对来源**:  默认是default，当多个微服务都调用这个资源时，可以配置微服务名来对指定的微服务设置阈值
- **阈值类型**: 分为QPS和线程数 假设阈值为10
- **QPS类型**: 只得是每秒访问接口的次数>10就进行限流
- **线程数**: 为接受请求该资源分配的线程数>10就进行限流  

![picture1](/img/springcloud/sentinel_console.png)

### 通信原理

Sentinel控制台与微服务端之间，实现了一套服务发现机制，集成了Sentinel的微服务都会将元数据传递给Sentinel控制台。

![picture1](/img/springcloud/sentinel_dashboard.png)


参考文献
--

图灵学院java架构课程
