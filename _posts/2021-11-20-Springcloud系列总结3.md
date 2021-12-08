---
layout:     post
title:      Springcloud系列总结3
subtitle:   LoadBalancer
date:       2021-11-20
author:     Yikun Chen
header-img: img/background/header.png
catalog: true
tags:
    - springcloud
    - loadbalancer
---


# Springcloud系列总结3

LoadBalancer
--

## 基本概念

Spring Cloud LoadBalancer是Spring Cloud官方自己提供的客户端负载均衡器, 用来替代Ribbon。包括两种客户端。

**RestTemplate**

RestTemplate是Spring提供的用于访问Rest服务的客户端，RestTemplate提供了多种便捷访问远程Http服务的方法，能够大大提高客户端的编写效率。默认情况下，RestTemplate默认依赖jdk的HTTP连接工具。

**WebClient**

WebClient是从Spring WebFlux 5.0版本开始提供的一个非阻塞的基于响应式编程的进行Http请求的客户端工具。它的响应式编程的基于Reactor的。WebClient中提供了标准Http请求方式对应的get、post、put、delete等方法，可以用来发起相应的请求。

## 接入方式

### RestTemplate

首先要在配置文件里面移除nacos的ribbon。
```java
spring:
  application:
    name: mall-user-loadbalancer-demo
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
    # 不使用ribbon
    loadbalancer:
      ribbon:
        enabled: false
```

使用@LoadBalanced注解配置RestTemplate。
```java
@Configuration
public class RestConfig {
    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```
调用服务。
```java
@RestController
@RequestMapping("/user")
public class UserController {

    @Autowired
    private RestTemplate restTemplate;

    @RequestMapping(value = "/findOrderByUserId/{id}")
    public R findOrderByUserId(@PathVariable("id") Integer id) {
        String url = "http://mall-order/order/findOrderByUserId/"+id;
        R result = restTemplate.getForObject(url,R.class);
        return result;
    }
}
```


### WebClient

配置WebClient作为负载均衡器的client。
```java
@Configuration
public class WebClientConfig {

    @LoadBalanced
    @Bean
    WebClient.Builder webClientBuilder() {
        return WebClient.builder();
    }
    
    @Bean
    WebClient webClient() {
      return webClientBuilder().build();
    }
}
```
调用服务。
```java
@Autowired
private WebClient webClient;

@RequestMapping(value = "/findOrderByUserId2/{id}")
public Mono<R> findOrderByUserIdWithWebClient(@PathVariable("id") Integer id) {

    String url = "http://mall-order/order/findOrderByUserId/"+id;
    //基于WebClient
    Mono<R> result = webClient.get().uri(url)
            .retrieve().bodyToMono(R.class);
    return result;
}
```



参考文献
--

