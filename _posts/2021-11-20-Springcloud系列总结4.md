---
layout:     post
title:      Springcloud系列总结4
subtitle:   Ribbon
date:       2021-11-20
author:     Yikun Chen
header-img: img/background/header.png
catalog: true
tags:
    - springcloud
    - ribbon
---


# Springcloud系列总结4

Ribbon
--

## 基本概念

Spring Cloud Ribbon是基于Netflix Ribbon 实现的一套客户端的负载均衡工具，Ribbon客户端组件提供一系列的完善的配置，如超时，重试等。通过Load Balancer获取到服务提供的所有机器实例，Ribbon会自动基于某种规则(轮询，随机)去调用这些服务。Ribbon也可以实现我们自己的负载均衡算法。

### 客户端的负载均衡

![picture1](/img/springcloud/ribbon_architecture.png)

### Ribbon模块

| ribbon-loadbalancer | 负载均衡模块，可独立使用，也可以和别的模块一起使用。         |
| ------------------- | ------------------------------------------------------------ |
| Ribbon              | 内置的负载均衡算法都实现在其中。                             |
| ribbon-eureka       | 基于 Eureka 封装的模块，能够快速、方便地集成 Eureka。        |
| ribbon-transport    | 基于 Netty 实现多协议的支持，比如 HTTP、Tcp、Udp 等。        |
| ribbon-httpclient   | 基于 Apache HttpClient 封装的 REST 客户端，集成了负载均衡模块，可以直接在项目中使用来调用接口。 |
| ribbon-example      | Ribbon 使用代码示例，通过这些示例能够让你的学习事半功倍。    |
| ribbon-core         | 一些比较核心且具有通用性的代码，客户端 API 的一些配置和其他 API 的定义。 |

### 原理

![picture1](/img/springcloud/ribbon_structure.png)

## 接入方式

添加@LoadBalanced注解。
```java
@Configuration
public class RestConfig {
    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    } 
```

修改controller。
```java
@Autowired
private RestTemplate restTemplate;

@RequestMapping(value = "/findOrderByUserId/{id}")
public R  findOrderByUserId(@PathVariable("id") Integer id) {
    // RestTemplate调用
    //String url = "http://localhost:8020/order/findOrderByUserId/"+id;
    //模拟ribbon实现
    //String url = getUri("mall-order")+"/order/findOrderByUserId/"+id;
    // 添加@LoadBalanced
    String url = "http://mall-order/order/findOrderByUserId/"+id;
    R result = restTemplate.getForObject(url,R.class);

    return result;
}
```
### 负载均衡策略

1. **RandomRule**： 随机选择一个Server。
2. **RetryRule**： 对选定的负载均衡策略机上重试机制，在一个配置时间段内当选择Server不成功，则一直尝试使用subRule的方式选择一个可用的server。
3. **RoundRobinRule**： 轮询选择， 轮询index，选择index对应位置的Server。
4. **AvailabilityFilteringRule**： 过滤掉一直连接失败的被标记为circuit tripped的后端Server，并过滤掉那些高并发的后端Server或者使用一个AvailabilityPredicate来包含过滤server的逻辑，其实就是检查status里记录的各个Server的运行状态。
5. **BestAvailableRule**： 选择一个最小的并发请求的Server，逐个考察Server，如果Server被tripped了，则跳过。
6. **WeightedResponseTimeRule**： 根据响应时间加权，响应时间越长，权重越小，被选中的可能性越低。
7. **ZoneAvoidanceRule**： 默认的负载均衡策略，即复合判断Server所在区域的性能和Server的可用性选择Server，在没有区域的环境下，类似于轮询(RandomRule)
8. **NacosRule:**  同集群优先调用

#### 修改全局策略

```java
@Configuration
public class RibbonConfig {

    /**
     * 全局配置
     * 指定负载均衡策略
     * @return
     */
    @Bean
    public IRule() {
        // 指定使用Nacos提供的负载均衡策略（优先调用同一集群的实例，基于随机权重）
        return new NacosRule();
    }
```

#### 修改局部配置

```java
# 被调用的微服务名
mall-order:
  ribbon:
    # 指定使用Nacos提供的负载均衡策略（优先调用同一集群的实例，基于随机&权重）
    NFLoadBalancerRuleClassName: com.alibaba.cloud.nacos.ribbon.NacosRule
```

#### 自定义策略

```java
@Slf4j
public class NacosRandomWithWeightRule extends AbstractLoadBalancerRule {

    @Autowired
    private NacosDiscoveryProperties nacosDiscoveryProperties;

    @Override
    public Server choose(Object key) {
        DynamicServerListLoadBalancer loadBalancer = (DynamicServerListLoadBalancer) getLoadBalancer();
        String serviceName = loadBalancer.getName();
        NamingService namingService = nacosDiscoveryProperties.namingServiceInstance();
        try {
            //nacos基于权重的算法
            Instance instance = namingService.selectOneHealthyInstance(serviceName);
            return new NacosServer(instance);
        } catch (NacosException e) {
            log.error("获取服务实例异常：{}", e.getMessage());
            e.printStackTrace();
        }
        return null;
    }
    @Override
    public void initWithNiwsConfig(IClientConfig clientConfig) {

    }
```

全局配置
```java
@Bean
public IRule ribbonRule() {
    return new NacosRandomWithWeightRule();
}
```

局部配置
```java
# 被调用的微服务名
mall-order:
  ribbon:
    # 自定义的负载均衡策略（基于随机&权重）
    NFLoadBalancerRuleClassName: com.tuling.mall.ribbondemo.rule.NacosRandomWithWeightRule

```

参考文献
--

图灵学院java架构课程
