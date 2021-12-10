---
layout:     post
title:      Springcloud系列总结9
subtitle:   OAuth2
date:       2021-11-20
author:     Yikun Chen
header-img: img/background/header.png
catalog: true
tags:
    - springcloud
    - oauth2
---


# Springcloud系列总结9

**OAuth2**
--

## 基本概念

使用OAuth2 认证的好处就是你只需要一个账号密码，就能在各个网站进行访问，而免去了在每个网站都进行注册的繁琐过程。

resource owner：资源所有者（指用户）

resource server：资源服务器存放受保护资源，要访问这些资源，需要获得访问令牌（下面例子中的 Twitter 资源服务器）

client：客户端代表请求资源服务器资源的第三方程序（下面例子中的 Quora）客户端同时也可能是一个资源服务器

authrization server：授权服务器用于发放访问令牌给客户端（下面例子中的 Twitter 授权服务器）

### 工作流程

![picture1](/img/springcloud/oauth2_process.png)

### 四种授权模式的选择

![picture1](/img/springcloud/oauth2_mode.png)

## 接入方式

### 资源服务器

配置服务器

```java
@Configuration
@EnableResourceServer
public class OAuth2ResourceServer extends ResourceServerConfigurerAdapter {
 @Override
 public void configure(HttpSecurity http) throws Exception {
 http.authorizeRequests()
 // 对 "/api/**" 开启认证
 .anyRequest()
 .authenticated()
 .and()
 .requestMatchers()
 .antMatchers("/api/**");
 }
}
```

编写接口

```java
@RestController
@RequestMapping("/api/example")
public class ExampleController {
 @RequestMapping("/hello")
 public String hello() {
 return "world";
 }
}
```

### 授权服务器

### 授权码模式

配置服务器


```java
@Configuration
@EnableAuthorizationServer
public class OAuth2AuthorizationServer extends AuthorizationServerConfigurerAdapter {
 @Override
 public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
 clients.inMemory() // <1>
 // <2> begin ...
 .withClient("clientapp").secret("112233") // Client 账号、密码。
 .redirectUris("http://localhost:9001/callback") // 配置回调地址，选填。
 .authorizedGrantTypes("authorization_code") // 授权码模式
 .scopes("read_userinfo", "read_contacts") // 可授权的 Scope
 // <2> end ...
// .and().withClient() // 可以继续配置新的 Client // <3>
 ;
 }
}
```

配置账户

```java
# Spring Security Setting
security.user.name=test
security.user.password=1024
```

获取授权码，访问http://localhost:8080/oauth/authorize?client\_id=clientapp&redirect\_uri=http://localhost:9001/callback&response\_type=code&scope=read\_userinfo，登陆之后获取回调地址的code。

获取访问令牌

```java
curl -X POST --user clientapp:112233 http://localhost:8080/oauth/token -H "content-type: application/x-www-form-urlencoded" -d "code=UydkmV&grant_type=authorization_code&redirect_uri=http%3A%2F%2Flocalhost%3A9001%2Fcallback&scope=read_userinfo"
```

访问url

```java
curl -X GET http://localhost:8080/api/example/hello -H "authorization: Bearer e60e41f2-2ad0-4c79-97d5-49af38e5c2e8"

```

### 密码模式

配置服务器


```java
@Configuration
@EnableAuthorizationServer
public class OAuth2AuthorizationServer extends AuthorizationServerConfigurerAdapter {
 // 用户认证
 @Autowired
 private AuthenticationManager authenticationManager;
 @Override
 public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
 endpoints.authenticationManager(authenticationManager);
 }
 @Override
 public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
 clients.inMemory()
 .withClient("clientapp").secret("112233") // Client 账号、密码。
 .authorizedGrantTypes("password") // 密码模式
 .scopes("read_userinfo", "read_contacts") // 可授权的 Scope
// .and().withClient() // 可以继续配置新的 Client
 ;
 }
}
```
配置账户

```java
# Spring Security Setting
security.user.name=test
security.user.password=1024
```
获取访问令牌
```java
curl -X POST --user clientapp:112233 http://localhost:8080/oauth/token -H "accept: application/json" -H "content-type: application/x-www-form-urlencoded" -d "grant_type=password&username=yunai&password=1024&scope=read_userinfo"
```
访问url
```java
curl -X GET http://localhost:8080/api/example/hello -H "authorization: Bearer e60e41f2-2ad0-4c79-97d5-49af38e5c2e8"

```

### 简化模式

配置服务器

```java
// 授权服务器配置
@Configuration
@EnableAuthorizationServer
public class OAuth2AuthorizationServer extends AuthorizationServerConfigurerAdapter {
 @Override
 public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
 clients.inMemory()
 .withClient("clientapp").secret("112233") // Client 账号、密码。
 .redirectUris("http://localhost:9001/callback") // 配置回调地址，选填。
 .authorizedGrantTypes("implicit") // 授权码模式
 .scopes("read_userinfo", "read_contacts") // 可授权的 Scope
// .and().withClient() // 可以继续配置新的 Client
 ;
 }
}
```
配置账户

```java
# Spring Security Setting
security.user.name=test
security.user.password=1024
```

直接获取访问令牌，访问http://localhost:8080/oauth/authorize?client\_id=clientapp&redirect\_uri=http://localhost:9001/callback&response\_type=implicit&scope=read\_userinfo，登陆之后获取回调地址的访问令牌。

访问url

```java
curl -X GET http://localhost:8080/api/example/hello -H "authorization: Bearer e60e41f2-2ad0-4c79-97d5-49af38e5c2e8"

```
#### 客户端模式

配置服务器
```java
@Configuration
@EnableAuthorizationServer
public class OAuth2AuthorizationServer extends AuthorizationServerConfigurerAdapter {
 // 用户认证
 @Autowired
 private AuthenticationManager authenticationManager;
 @Override
 public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
 endpoints.authenticationManager(authenticationManager);
 }
 @Override
 public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
 clients.inMemory()
 .withClient("clientapp").secret("112233") // Client 账号、密码。
 .authorizedGrantTypes("password") // 密码模式
 .scopes("read_userinfo", "read_contacts") // 可授权的 Scope
// .and().withClient() // 可以继续配置新的 Client
 ;
 }
}
```

无需配置账户

获取访问令牌
```java
curl -X POST "http://localhost:8080/oauth/token" --user clientapp:112233 -d "grant_type=client_credentials&scope=read_contacts"

```
访问url

```java
curl -X GET http://localhost:8080/api/example/hello -H "authorization: Bearer e60e41f2-2ad0-4c79-97d5-49af38e5c2e8"

```

#### 访问令牌

刷新令牌

修改服务配置

```java
@Configuration
@EnableAuthorizationServer
public class OAuth2AuthorizationServer extends AuthorizationServerConfigurerAdapter {
 // 用户认证
 @Autowired
 private AuthenticationManager authenticationManager;
 @Override
 public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
 endpoints.authenticationManager(authenticationManager);
 }
 @Override
 public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
 clients.inMemory()
 .withClient("clientapp").secret("112233") // Client 账号、密码。
 .authorizedGrantTypes("password", "refresh_token") // 密码模式 // <1>
 .scopes("read_userinfo", "read_contacts") // 可授权的 Scope
// .and().withClient() // 可以继续配置新的 Client
 ;
 }
}
```

删除令牌

新增删除接口

```java
@Autowired
private ConsumerTokenServices tokenServices;
@RequestMapping(method = RequestMethod.POST, value = "api/access_token/revoke")
public String revokeToken(@RequestParam("token") String token) {
 tokenServices.revokeToken(token);
 return token;
}
```



参考文献
--

[Spring Security OAuth2 入门](https://juejin.cn/post/7023980561186160677#heading-5)

[Spring Security 与 OAuth2（介绍）](https://www.jianshu.com/p/68f22f9a00ee)

