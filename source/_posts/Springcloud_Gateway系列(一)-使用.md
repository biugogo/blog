---
title: SpringCloud_Gateway系列(一)-使用
date: 2018-9-25 21:47:51
tags:
 -Spring Cloud
categories: Spring Cloud
thumbnail: https://raw.githubusercontent.com/apollochen123/image/master/%E9%BB%98%E8%AE%A42.jpg
---

# SpringCloud_Gateway系列(一)-使用
---------
# Overview
最近闲着没事，看了一下Spring提供网关。基于Netty + Spring webflux 反应式编程web框架的Spring Cloud Gateway。抛弃了传统servlet模式，投入了netty怀抱的spring-webflux，很适用网关高并发场景。
>Spring Framework 5 includes a new spring-webflux module. The module contains support for reactive HTTP and WebSocket clients as well as for reactive server web applications including REST, HTML browser, and WebSocket style interactions.

这里没有自己实测zuul，zuul2.0，SpringCloudGateway的性能，但是鉴于网上对于SpringCloudGateway很多有失偏颇的测试：（国内大部分都道听途说看了这篇帖子）
* http://www.infoq.com/cn/articles/comparing-api-gateway-performances

这里得说一句，没那么差（可能是最好的网关），这个帖子的性能差是归结于reactor-netty使用http 1.0的问题。具体可以参考官方spencergibb给出的解释与测试：
* https://github.com/spring-cloud/spring-cloud-gateway/issues/124

--------

# Reference
* https://docs.spring.io/spring/docs/5.0.0.BUILD-SNAPSHOT/spring-framework-reference/html/web-reactive.html
* http://cloud.spring.io/spring-cloud-static/Finchley.SR1/single/spring-cloud.html#_spring_cloud_gateway

----
# 专业名词解释
1. Route:路由，网关的基础模块，他包括一个ID，URL，一个Predicate集合，一个filter集合。
2. Predicate：就是Java8的Predicate。决定这个请求走哪个Route。入参是org.springframework.web.server.ServerWebExchange。
3. filter：继承GatewayFilter，对请求一系列操作。

一句话解释，网关有很多路---Route，每个route都有Predicate,当你碰到一个完全符合的Predicate集合，你就会进入这条路，进入这条路后，filter就会对你进行不可描述的操作，比如我给你http头加个字段，做身份校验，负载均衡等。

---

# How it works
![howitworks](https://raw.githubusercontent.com/apollochen123/image/master/springcloudgateway/howitworks.png)

---------

# 简单Yml配置
可以参考官方文档，比我这写的详细 [官方文档](http://cloud.spring.io/spring-cloud-static/Finchley.SR1/single/spring-cloud.html#gateway-request-predicates-factories)

1. ### 内置常用Predicate Factories
   * After/Before/Between Route Predicate Factory
   设置match After/Before/Between时间的请求
   ```yml
   spring:
     cloud:
       gateway:
         routes:
         - id: before_route
           uri: http://example.org
           predicates:
           - Before=2017-01-20T17:42:47.789-07:00[America/Denver]
#        - After=2017-01-20T17:42:47.789-07:00[America/Denver]
#        - Between=2017-01-20T17:42:47.789-07:00[America/Denver], 2017-01-21T17:42:47.789-07:00[America/Denver]
   ```
   * 设置match header，cookie，Host，正则表达式匹配Predicate
   ```yml
   spring:
     cloud:
       gateway:
         routes:
         - id: cookie_route
           uri: http://example.org
           predicates:
             # Cookies匹配正则表达式ch.p
           - Cookie=chocolate, ch.p
             # Header中的X-Request-Id匹配正则表达式\d+
           - Header=X-Request-Id, \d+
             # Host匹配**.somehost.org
           - Host=**.somehost.org
   ```

   * 最常用的Path Route Predicate Factory。匹配url的Predicate
   ```yml
   spring:
     cloud:
       gateway:
         routes:
         - id: host_route
           uri: http://example.org
           predicates:
             #匹配/local/a,/local/a/b等
           - Path=/local/**
   ```
   * 其他
   ```yml
   spring:
     cloud:
       gateway:
         routes:
           - id: host_route
             uri: http://example.org
             predicates:
               #匹配请求方式为Get的请求
             - Method=GET
               #匹配请求路径带有参数baz
             - Query=baz
               #匹配请求路径带参数foo=ba.
             - Query=foo, ba.
               #匹配远程请求ip地址符合192.168.1.1/24
             - RemoteAddr=192.168.1.1/24
   ```

2. ### 内置常用GatewayFilter Factories
```yml
spring:
  cloud:
    gateway:
      routes:
      - id: test_routes
        uri: http://example.org
        filters:
          #给请求Header添加headerX-Request-Foo=Bar
        - AddRequestHeader=X-Request-Foo, Bar
          #给请求添加param foo=bar
        - AddRequestParameter=foo, bar
          #给Respouse header添加X-Response-Foo=Bar
        - AddResponseHeader=X-Response-Foo, Bar
          #很常用的前缀剥离
          #比如：StripPrefix=1就是剥离一层前缀，StripPrefix=2就是剥离两层前缀。
          #StripPrefix=1情况下http://localhost:8828/gamemeta3/gamemeta/admin/bizdata/search将会被转发：http://example.org/gamemeta/admin/bizdata/search
        - StripPrefix=0
          #符合的请求将会被转发到http://example.org/kxdRoom/question/search
        - SetPath=/kxdRoom/question/search
```
3. ### 配置例子

```yml
spring:
  logging:
    level: debug
  application:
    name: gateway
  cloud:
    gateway:
      routes:
      - id: local_prefix_route
        uri: lb:http://local
        predicates:
        - Path=/local/**
        filters:
        - StripPrefix=1
#        - Hystrix=local
        - name: Hystrix
          args:
            name: local
            fallbackUri: forward:/error

      #定向跳转
      - id: question_search_route
        uri: https://wolfkill-game-test.yy.com
        predicates:
        - Path=/room/search
        #- Path=/kxdRoom/**
        filters:
        - SetPath=/kxdRoom/question/search

server:
  port: 8828

ribbon:
  eureka:
   enabled: false

local:
  ribbon:
    listOfServers: http://localhost:9001,http://localhost:9002
hystrix:
  command:
    local:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 1000
      circuitBreaker:
        enabled: true
        errorThresholdPercentage: 50
        sleepWindowInMilliseconds: 5000

```
local_prefix_route:这个route是一个包含了hystrix加ribbon配置的.当我们请求
http://localhost:8828/local/pro/get会被转发到http://localhost:9001/pro/get或者http://localhost:9001/pro/get。如果一秒不反回，会转发到error。

question_search_route：这个route会让http://localhost:8828/kxdRoom/** 转发到https://wolfkill-game-test.yy.com/kxdRoom/question/search。



# 代码配置
见官方文案：
http://cloud.spring.io/spring-cloud-static/Finchley.SR1/single/spring-cloud.html#_fluent_java_routes_api

# 动态配置Route
见SpringCloud_Gateway系列(三)整合
