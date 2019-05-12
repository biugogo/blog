---
title: SpringCloud Gateway系列（三）-整合
date: 2018-9-27 15:00:02
tags:
 -Spring Cloud
categories: Spring Cloud
thumbnail: https://raw.githubusercontent.com/apollochen123/image/master/%E9%BB%98%E8%AE%A42.jpg
---


# SpringCloud Gateway系列（三）-整合

----------
## 1. 自己实现GlobalFilter
实现全局过滤器很简单。比如我们想要拒绝所有请求或者想要做权限校验时，可以使用GlobalFilter
 - 方式一：简单的鉴权接口
```java
@Bean
@Order(0)
public GlobalFilter udbFilter() {
    return (exchange, chain) -> {
        System.out.println("udbFilter权限校验");
        return chain.filter(exchange);
    };
}
```
  -方式二:当swt设置为true，拒绝所有请求
  ```java
  @Bean
  public GlobalFilter myGlobalFilter() {
      return new MyGlobalFilter();
  }

  public static class MyGlobalFilter implements GlobalFilter, Ordered {
    private volatile AtomicBoolean swt = new AtomicBoolean(false);

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        if (swt.get()) {
            System.out.println("拒绝所有请求，直接返回");
            setResponseStatus(exchange, HttpStatus.SERVICE_UNAVAILABLE);
            return exchange.getResponse().setComplete();
        }
        return chain.filter(exchange);
    }

    @Override
    public int getOrder() {
        return 0;
    }

    public void setSwt(Boolean swt) {
        this.swt.compareAndSet(!swt, swt);
    }
}
  ```


----------

## 2. 整合Ribbon达到负载均衡
SpringCloud Gateway整合Ribbon很简单，主要因为是本身已经提供了Ribbon整合的配置类
GatewayLoadBalancerClientAutoConfiguration
```java
@Configuration
@ConditionalOnClass({LoadBalancerClient.class, RibbonAutoConfiguration.class, DispatcherHandler.class})
@AutoConfigureAfter(RibbonAutoConfiguration.class)
public class GatewayLoadBalancerClientAutoConfiguration {

	// GlobalFilter beans

	@Bean
	@ConditionalOnBean(LoadBalancerClient.class)
	public LoadBalancerClientFilter loadBalancerClientFilter(LoadBalancerClient client) {
		return new LoadBalancerClientFilter(client);
	}
}
```
当classPath下有RibbonAutoConfiguration，LoadBalancerClient，DispatcherHandler就会执行这个配置。生成一个GlobalFilter。

LoadBalancerClientFilter中，使用
```java
ServiceInstance instance = loadBalancer.choose(url.getHost());
```
获取loadBalancer集合中的一个实例，进行请求。

可以看一下choose这个方法
```java
@Override
public ServiceInstance choose(String serviceId) {
  Server server = getServer(serviceId);
  if (server == null) {
    return null;
  }
  return new RibbonServer(serviceId, server, isSecure(server, serviceId),
      serverIntrospector(serviceId).getMetadata(server));
}
```
把Server对象包装为RibbonServer。继续深入 getServer(serviceId);
```java
protected Server getServer(String serviceId) {
  return getServer(getLoadBalancer(serviceId));
}

....

protected Server getServer(ILoadBalancer loadBalancer) {
  if (loadBalancer == null) {
    return null;
  }
  return loadBalancer.chooseServer("default"); // TODO: better handling of key
}

....

protected ILoadBalancer getLoadBalancer(String serviceId) {
  return this.clientFactory.getLoadBalancer(serviceId);
}
```
这里是从clientFactory得到一个ILoadBalancer。spring默认实现是ZoneAwareLoadBalancer。
ZoneAwareLoadBalancer继承自DynamicServerListLoadBalancer，拥有动态更新ServerList功能。又实现了分组功能。
DynamicServerListLoadBalancer中实现了updateListOfServers()方法，用于刷新ListOfServer。触发这个动作的是一个叫做PollingServerListUpdater的类。有兴趣的可以看一下。

所以可以通过重新ServerList<Server>这个Bean来达到动态更新Servce集的功能。（不借助于服务发现）

```java
@Bean
@SuppressWarnings("unchecked")
public ServerList<Server> ribbonServerList() {
    return new ServerList<Server>() {
        @Override
        public List<Server> getInitialListOfServers() {
            return get();
        }

        @Override
        public List<Server> getUpdatedListOfServers() {
            return get();
        }

        private List<Server> get() {
            //这里可以是DB访问配置，可以是从其他地方拉取
            List<Server> arrayList = new ArrayList();
            Server server = new Server("http://localhost:9001");
            Server server2 = new Server("http://localhost:9002");
            arrayList.add(server);
            arrayList.add(server2);
            return arrayList;
        }
    };
}
```
当然你可以把初始化配置写在yml中,注意lb前缀
```yml
cloud:
  gateway:
    routes:
    - id: bizdata_search
      uri: lb:http://testEnv

testEnv:
  ribbon:
    listOfServers: http://wolfkill-game.yy.com,http://wolfkill-game.yy.com
```

总结：
  - 添加Ribbon依赖：spring-cloud-starter-netflix-ribbon
  - 配置serverList
  - 整合成功。

----------

## 3. 整合Hystrix达到fallback和熔断
spring cloud gateway对于Hystrix的整合也是提供了很好的支持
  - 添加Hystrix依赖spring-cloud-starter-netflix-hystrix
  - yml中添加
  ```yml
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

  local:
    ribbon:
      listOfServers: http://localhost:9001
  hystrix:
    command:
      local:
        execution:
          isolation:
            thread:
              timeoutInMilliseconds: 1000
        circuitBreaker:
          enabled: true
          errorThresholdPercentage: 10
          sleepWindowInMilliseconds: 5000
  ```
  这样就为local_prefix_route添加了Hystix，并且集成了Ribbon。
  - 加入降级接口
  ```java
  @GetMapping("/error")
  public Mono<String> error() {
      return Mono.just("error");
  }
  ```

  想要了解原理可以参看HystrixGatewayFilterFactory的源码。



------

## 4. 自己实现RateLimiter
SpringCloud Gateway本身自带了一个基于Redis的RateLimiter.我们这里不使用这个。我这里写一个比较简单的RateLimiter.(基于guava的com.google.common.util.concurrent.RateLimiter写法).仅仅测试用，生产可能得完善一下
```java
public class YYHttpRateLimiter implements RateLimiter<YYHttpRateLimiter.Config> {
    private static final Logger logger = LoggerFactory.getLogger(YYHttpRateLimiter.class);

    private volatile ConcurrentHashMap<String, com.google.common.util.concurrent.RateLimiter> rateLimiterMap = new ConcurrentHashMap<>();

    @Override
    public Mono<Response> isAllowed(String routeId, String id) {
        String uri = routeId + id;
        com.google.common.util.concurrent.RateLimiter rateLimiter = rateLimiterMap.get(uri);
        if (rateLimiter == null) {
            if (rateLimiterMap.size() >= Config.maxUri) {
                //throw new IllegalStateException("uri size exceed " + maxUri + " when create uri " + uri);
                logger.warn("[cmd=acquire,uri size exceed {} when create uri {}]", Config.maxUri, uri);
                return Mono.just(new Response(true, Collections.emptyMap()));
            }

            rateLimiter = com.google.common.util.concurrent.RateLimiter.create(Config.permitPerSecond);
            rateLimiterMap.putIfAbsent(uri, rateLimiter);
            rateLimiter = rateLimiterMap.get(uri);
        }
        return Mono.just(new Response(rateLimiter.tryAcquire(), Collections.emptyMap()));
    }

    @Override
    public Map<String, Config> getConfig() {
        return null;
    }

    @Override
    public Class<Config> getConfigClass() {
        return null;
    }

    @Override
    public Config newConfig() {
        return null;
    }

    public static class Config {
        private static int permitPerSecond = 1;
        private static int maxUri = 2 << 10;
    }
}
```
再重写一个KeyResolver
```java
public class YYKeyResolver implements KeyResolver {

    @Override
    public Mono<String> resolve(ServerWebExchange exchange) {
        return Mono.just(exchange.getRequest().getURI().getRawPath());
    }
}
```
配置bean
```java
@Configuration
public class RateLimiterConfig {
    @Bean
    public YYHttpRateLimiter yyHttpRateLimiter() {
        return new YYHttpRateLimiter();
    }

    @Bean
    public YYKeyResolver yyKeyResolver() {
        return new YYKeyResolver();
    }
}
```

在yml中配置使用
```yml
spring:
  cloud:
    gateway:
      routes:
      - id: local_prefix_route
        uri: lb:http://local
        predicates:
        - Path=/local/**
        filters:
        - StripPrefix=1
        - name: RequestRateLimiter
          args:
            rate-limiter: "#{@yyHttpRateLimiter}"
            key-resolver: "#{@yyKeyResolver}"
```


-----


## 5. 实现动态route CRUD操作
动态配置Route主要是通过event机制完成的。
我在这里实现一个启动后60秒后定时更新route的demo。
```java
@Autowired
private RouteDefinitionWriter routeDefinitionWriter;

.....

public RouteConfigService() {
    new Thread(() -> {
        try {
            Thread.currentThread().sleep(6 * 1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        save();
    }).start();
}

public void save() {
    System.out.println("================ update ==================");
    RouteDefinition routeDefinition = new RouteDefinition();
    routeDefinition.setId("writer_test");
    routeDefinition.setOrder(1);

    try {
        routeDefinition.setUri(new URI("http://wolfkill-game.yy.com"));
    } catch (URISyntaxException e) {
        e.printStackTrace();
    }

    List<FilterDefinition> filters = new ArrayList<>();
    filters.add(new FilterDefinition("StripPrefix=1"));

    List<PredicateDefinition> predicateDefinitions = new ArrayList<>();
    predicateDefinitions.add(new PredicateDefinition("Path=/gamemeta3/**"));

    routeDefinition.setFilters(filters);
    routeDefinition.setPredicates(predicateDefinitions);

    routeDefinitionWriter.save(Mono.just(routeDefinition)).then(Mono.fromRunnable(() -> {
        this.publisher.publishEvent(new RefreshRoutesEvent(this));
    })).subscribe();

    System.out.println("================ updated ==================");
}
```
routeDefinitionWriter是一个接口，默认实现是InMemoryRouteDefinitionRepository。如果你需要实现自己的writer，可以模仿重写（放redis，放配置中心都可以）。


#### 原理分析：
CachingRouteLocator源码
```java
/**
 * Clears the routes cache
 * @return routes flux
 */
public Flux<Route> refresh() {
  this.cache.clear();
  return this.routes;
}

@EventListener(RefreshRoutesEvent.class)
/* for testing */ void handleRefresh() {
  refresh();
}
```
当我们接收到RefreshRoutesEvent时，清空缓存，这时候，我们在subscribe一下，就会重新执行RouteDefinition到Route的流程，重新形成缓存。
