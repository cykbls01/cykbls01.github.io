---
layout:     post
title:      Springcloud系列总结2
subtitle:   Gateway
date:       2021-11-20
author:     Yikun Chen
header-img: img/background/header.png
catalog: true
tags:
    - springcloud
    - gateway
---


# Springcloud系列总结2

Gateway
--

## 基本概念

Gateway 不仅提供统一的路由方式，并且基于 Filter 链的方式提供了网关基本的功能，例如：安全，监控/指标，和限流。其底层采用了netty进行通信。

### 架构

![picture1](/img/springcloud/gateway_architecture.png)

#### Filter（过滤器）：

过滤器用于某一个路由的请求或者响应进行修改的组件，在 Spring Cloud Gateway 都要实现 GatewayFilter 接口，并且需要由基于 GatewayFilterFactory 具体实现类构造。SpringCloud Gateway中的filter分为Gateway FilIer和Global Filter。

#### Route（路由）：

路由是网关中最基础的部分，路由信息包括一个ID、一个目的URI、一组断言工厂、一组Filter组成。如果断言为真，则说明请求的URL和配置的路由匹配。

#### Predicate（断言）：

 Java 8 函数库的 Predicate 对象，具体类型为 Predicate ，用于匹配 HTTP 请求上数据信息，如请求头信息，请求体信息。如果对于某个请求的断言为 true，那么它所关联的路由就算匹配成功，然后将请求给这个路由处理。

![picture1](/img/springcloud/gateway_predicate.png)

#### 处理过程

1. 客户端请求首先被 GatewayHandlerMapping 获取，然后根据断言匹配找到对应的路由
2. 找到路由后，走完所关联的一组请求过滤器的处理方法，请求到目标 URI 所对应的服务程序，获得服务响应。
3. 网关收到响应后，通过关联的响应过滤器的处理方法后，同样由 GatewayHandlerMapping 返回响应给客户端。

## 路由配置方式

### 文件配置

```java
server:
  port: 8080
spring:
  application:
    name: api-gateway
  cloud:
    gateway:
      routes:
        -id: url-proxy-1
          uri: https://blog.csdn.net
          predicates:
            -Path=/csdn
```

各字段含义如下：

id：我们自定义的路由 ID，保持唯一

uri：目标服务地址

predicates：路由条件，Predicate 接受一个输入参数，返回一个布尔值结果。该接口包含多种默认方法来将 Predicate 组合成其他复杂的逻辑（比如：与，或，非）。

上面这段配置的意思是，配置了一个 id 为 url-proxy-1的URI代理规则，路由的规则为：

当访问地址http://localhost:8080/csdn/1.jsp时，会路由到上游地址https://blog.csdn.net/1.jsp。

### 代码配置

也可以不在启动类设置，结合配置文件进行配置，见断言的例子。

```java
package com.springcloud.gateway;
 
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.gateway.route.RouteLocator;
import org.springframework.cloud.gateway.route.builder.RouteLocatorBuilder;
import org.springframework.context.annotation.Bean;
 
@SpringBootApplication
public class GatewayApplication {
 
    public static void main(String[] args) {
        SpringApplication.run(GatewayApplication.class, args);
    }
 
    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()
                .route("path_route", r -> r.path("/csdn")
                        .uri("https://blog.csdn.net"))
                .build();
    }
 
}
```

### Predicate配置类型

#### 时间匹配

对应after，before和between，处于时间点之外的请求会被阻拦。

```java
spring:
  cloud:
    gateway:
      #设置路由：路由id、路由到微服务的uri、断言
      routes:
      - id: order_route  #路由ID，全局唯一
        uri: http://localhost:8020  #目标微服务的请求地址和端口
        predicates:
         # 测试：http://localhost:8888/order/findOrderByUserId/1
        # 匹配在指定的日期时间之后发生的请求  入参是ZonedDateTime类型
        - After=2021-01-31T22:22:07.783+08:00[Asia/Shanghai]
```

#### Cookie匹配

不满足对应cookie的内容会被阻拦。

```java
spring:
  cloud:
    gateway:
      #设置路由：路由id、路由到微服务的uri、断言
      routes:
      - id: order_route  #路由ID，全局唯一
        uri: http://localhost:8020  #目标微服务的请求地址和端口
        predicates:
         # Cookie匹配
        - Cookie=username, fox
```

#### Header匹配

请求头header需要携带数字。

```java
spring:
  cloud:
    gateway:
      #设置路由：路由id、路由到微服务的uri、断言
      routes:
      - id: order_route  #路由ID，全局唯一
        uri: http://localhost:8020  #目标微服务的请求地址和端口
        predicates:
         # Header匹配  请求中带有请求头名为 x-request-id，其值与 \d+ 正则表达式匹配
        - Header=X-Request-Id, \d+
```

#### Path匹配

请求路径需要与设置匹配。

```java
spring:
  cloud:
    gateway:
      #设置路由：路由id、路由到微服务的uri、断言
      routes:
      - id: order_route  #路由ID，全局唯一
        uri: http://localhost:8020  #目标微服务的请求地址和端口
        predicates:
         # 测试：http://localhost:8888/order/findOrderByUserId/1
        - Path=/order/**   #Path路径匹配
```

#### 自定义匹配

```java
@Component
@Slf4j
public class CheckAuthRoutePredicateFactory extends AbstractRoutePredicateFactory<CheckAuthRoutePredicateFactory.Config> {

    public CheckAuthRoutePredicateFactory() {
        super(Config.class);
    }

    @Override
    public Predicate<ServerWebExchange> apply(Config config) {
        return new GatewayPredicate() {

            @Override
            public boolean test(ServerWebExchange serverWebExchange) {
                log.info("调用CheckAuthRoutePredicateFactory" + config.getName());
                if(config.getName().equals("fox")){
                    return true;
                }
                return false;
            }
        };
    }


    @Override
    public List<String> shortcutFieldOrder() {
        return Collections.singletonList("name");
    }

    public static class Config {

        private String name;

        public String getName() {
            return name;
        }

        public void setName(String name) {
            this.name = name;
        }
    }
}
```


```java
spring:
  cloud:
    gateway:
      #设置路由：路由id、路由到微服务的uri、断言
      routes:
      - id: order_route  #路由ID，全局唯一
        uri: http://localhost:8020  #目标微服务的请求地址和端口
        predicates:
         # 测试：http://localhost:8888/order/findOrderByUserId/1
        - Path=/order/**   #Path路径匹配
        #自定义CheckAuth断言工厂
        - CheckAuth=fox   
```

### Filter配置类型

#### 添加请求头

```java
spring:
  cloud:
    gateway:
      #设置路由：路由id、路由到微服务的uri、断言
      routes:
      - id: order_route  #路由ID，全局唯一
        uri: http://localhost:8020  #目标微服务的请求地址和端口
        #配置过滤器工厂
        filters:
        - AddRequestHeader=X-Request-color, red  #添加请求头 
```

#### 添加请求参数

```java
spring:
  cloud:
    gateway:
      #设置路由：路由id、路由到微服务的uri、断言
      routes:
      - id: order_route  #路由ID，全局唯一
        uri: http://localhost:8020  #目标微服务的请求地址和端口
        #配置过滤器工厂
        filters:
        - AddRequestParameter=color, blue  # 添加请求参数
```

#### 添加前缀

```java
spring:
  cloud:
    gateway:
      #设置路由：路由id、路由到微服务的uri、断言
      routes:
      - id: order_route  #路由ID，全局唯一
        uri: http://localhost:8020  #目标微服务的请求地址和端口
        #配置过滤器工厂
        filters:
        - PrefixPath=/mall-order  # 添加前缀 对应微服务需要配置context-path
```

#### 添加重定向

```java
spring:
  cloud:
    gateway:
      #设置路由：路由id、路由到微服务的uri、断言
      routes:
      - id: order_route  #路由ID，全局唯一
        uri: http://localhost:8020  #目标微服务的请求地址和端口
        #配置过滤器工厂
        filters:
        - RedirectTo=302, https://www.baidu.com/  #重定向到百度
```

#### 自定义过滤

```java
@Component
@Slf4j
public class CheckAuthGatewayFilterFactory extends AbstractNameValueGatewayFilterFactory {

    @Override
    public GatewayFilter apply(NameValueConfig config) {
        return (exchange, chain) -> {
            log.info("调用CheckAuthGatewayFilterFactory==="
                    + config.getName() + ":" + config.getValue());
            return chain.filter(exchange);
        };
    }
}
```


```java
spring:
  cloud:
    gateway:
      #设置路由：路由id、路由到微服务的uri、断言
      routes:
      - id: order_route  #路由ID，全局唯一
        uri: http://localhost:8020  #目标微服务的请求地址和端口
        #配置过滤器工厂
        filters:
        - CheckAuth=fox,男
```

#### 全局过滤
```java
@Component
@Order(-1)
@Slf4j
public class CheckAuthFilter implements GlobalFilter {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        //校验请求头中的token
        List<String> token = exchange.getRequest().getHeaders().get("token");
        log.info("token:"+ token);
        if (token.isEmpty()){
            return null;
        }
        return chain.filter(exchange);
    }
}

@Component
public class CheckIPFilter implements GlobalFilter, Ordered {

    @Override
    public int getOrder() {
        return 0;
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        HttpHeaders headers = exchange.getRequest().getHeaders();
        if (getIp(headers).equals("127.0.0.1")) {
            return null;
        }
        return chain.filter(exchange);
    }

    private String getIp(HttpHeaders headers) {
        return headers.getHost().getHostName();
    }
}
```

### 跨域配置
```java
@Configuration
public class CorsConfig {
    @Bean
    public CorsWebFilter corsFilter() {
        CorsConfiguration config = new CorsConfiguration();
        config.addAllowedMethod("*");
        config.addAllowedOrigin("*");
        config.addAllowedHeader("*");

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource(new PathPatternParser());
        source.registerCorsConfiguration("/**", config);

        return new CorsWebFilter(source);
    }
}
```


参考文献
--

