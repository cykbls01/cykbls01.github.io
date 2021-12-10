---
layout:     post
title:      Springcloud系列总结5
subtitle:   Feign
date:       2021-11-20
author:     Yikun Chen
header-img: img/background/header.png
catalog: true
tags:
    - springcloud
    - feign
---


# Springcloud系列总结5

Feign
--

## 基本概念

Feign是Netflix开发的声明式、模板化的HTTP客户端，其灵感来自Retrofit、JAXRS-2.0以及WebSocket。Feign可帮助我们更加便捷、优雅地调用HTTP API。Feign支持多种注解，例如Feign自带的注解或者JAX-RS注解等。Spring Cloud openfeign对Feign进行了增强，使其支持Spring MVC注解，另外还整合了Ribbon和Eureka，从而使得Feign的使用更加方便。

Feign可以做到使用 HTTP 请求远程服务时就像调用本地方法一样的体验，开发者完全感知不到这是远程方法，更感知不到这是个 HTTP 请求。它像 Dubbo 一样，consumer 直接调用接口方法调用 provider，而不需要通过常规的 Http Client 构造请求再解析返回数据。它解决了让开发者调用远程接口就跟调用本地方法一样，无需关注与远程的交互细节，更无需关注分布式环境开发。

### 设计架构

![picture1](/img/springcloud/feign_architecture.png)

## 接入方式

### 单独使用

编写接口。

```java
public interface RemoteService {

    @Headers({"Content-Type: application/json","Accept: application/json"})
    @RequestLine("GET /order/findOrderByUserId/{userId}")
    public R findOrderByUserId(@Param("userId") Integer userId);

```

调用。
```java
public class FeignDemo {

    public static void main(String[] args) {

        //基于json
        // encoder指定对象编码方式，decoder指定对象解码方式
        RemoteService service = Feign.builder()
                .encoder(new JacksonEncoder())
                .decoder(new JacksonDecoder())
                .options(new Request.Options(1000, 3500))
                .retryer(new Retryer.Default(5000, 5000, 3))
                .target(RemoteService.class, "http://localhost:8020/");

        System.out.println(service.findOrderByUserId(1));
    }
```
### spring cloud接入

编写调用接口+@FeignClient注解。

```java
@FeignClient(value = "mall-order",path = "/order")
public interface OrderFeignService {

    @RequestMapping("/findOrderByUserId/{userId}")
    public R findOrderByUserId(@PathVariable("userId") Integer userId);
}
```

调用端在启动类上添加@EnableFeignClients注解。

```java
@SpringBootApplication
@EnableFeignClients
public class MallUserFeignDemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(MallUserFeignDemoApplication.class, args);
    }
}
```

发起调用，像调用本地方式一样调用远程服务。

```java
@RestController
@RequestMapping("/user")
public class UserController {

    @Autowired
    OrderFeignService orderFeignService;

    @RequestMapping(value = "/findOrderByUserId/{id}")
    public R  findOrderByUserId(@PathVariable("id") Integer id) {
        //feign调用
        R result = orderFeignService.findOrderByUserId(id);
        return result;
    }
}
```

### 自定义配置

#### 配置日志

配置日志，可以自由选择全局或者局部。

```java
// 注意： 此处配置@Configuration注解就会全局生效，如果想指定对应微服务生效，就不能配置
public class FeignConfig {
    /**
     * 日志级别
     *
     * @return
     */
    @Bean
    public Logger.Level feignLoggerLevel() {
        return Logger.Level.FULL;
    }
}
```

在注解里面可以进行局部配置。
```java
@FeignClient(value = "mall-order",path = "/order", configuration = FeignConfig.class)
```

yml配置日志输出级别。
```java
logging:
  level:
    com.tuling.mall.feigndemo.feign: debug

```

#### 配置拦截器

```java
public class FeignAuthRequestInterceptor implements RequestInterceptor {
    @Override
    public void apply(RequestTemplate template) {
        // 业务逻辑
        String access_token = UUID.randomUUID().toString();
        template.header("Authorization",access_token);
    }
}

@Configuration  // 全局配置
public class FeignConfig {
    @Bean
    public Logger.Level feignLoggerLevel() {
        return Logger.Level.FULL;
    }
    /**
     * 自定义拦截器
     * @return
     */
    @Bean
    public FeignAuthRequestInterceptor feignAuthRequestInterceptor(){
        return new FeignAuthRequestInterceptor();
    }
}
```

yml配置。
```java
feign:
  client:
    config:
      mall-order:  #对应微服务
        requestInterceptors[0]:  #配置拦截器
          com.tuling.mall.feigndemo.interceptor.FeignAuthRequestInterceptor
```

#### 配置超时
```java
@Configuration
public class FeignConfig {
    @Bean
    public Request.Options options() {
        return new Request.Options(5000, 10000);
    }
}
```
yml配置。

```java
feign:
  client:
    config:
      mall-order:  #对应微服务
        # 连接超时时间，默认2s
        connectTimeout: 5000
        # 请求处理超时时间，默认5s
        readTimeout: 10000
```


参考文献
--

图灵学院java架构课程
