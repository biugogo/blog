---
title: Hystrix Configuration
date: 2017-11-14 21:47:51
tags:
 -Spring Cloud
categories: Spring Cloud
thumbnail: https://raw.githubusercontent.com/apollochen123/image/master/%E9%BB%98%E8%AE%A42.jpg
---

# **Hystrix配置**
 本文参考[spring官方文档](http://cloud.spring.io/spring-cloud-static/Camden.SR7/#_circuit_breaker_hystrix_clients)，[Hystrix Configuration](https://github.com/Netflix/Hystrix/wiki/Configuration) , [hystrix-javanica](https://github.com/Netflix/Hystrix/tree/master/hystrix-contrib/hystrix-javanica#configuration)完成。你可以在这里找到一些有用的信息
---
 ## Introduction
 Hystrix uses [Archaius](https://github.com/Netflix/archaius)  for the default implementation of properties for configuration.
Each property has four levels of precedence:
  1. **Global default from code**
     This is the default if none of the following 3 are set. 优先级最低的级别，如果其他的级别没有设定，默认使用此项配置信息。
     The global default is shown as “Default Value”. 所有未配置的项默认值。
  2. **Dynamic global default property**
     You can change a global default value by using properties. 你可以改变这个默认值在配置文件中。
    example in yml:

  ```yml
      hystrix:
        command:
          default:
            execution:
              isolation:
                thread:
                  timeoutInMilliseconds: 60000
  ```
example in property:
```java
       hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds: 60000
```     
  3. **Instance default from code**
     代码里设置Instance的default值，优先级高于前两项。这里有一个例子关于代码设置默认值给一个Instance：

  ```java    
     public HystrixCommandInstance(int id) {
          super(Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("ExampleGroup"))
              .andCommandPropertiesDefaults(HystrixCommandProperties.Setter()
                     .withExecutionTimeoutInMilliseconds(500)));
          this.id = id;
      }
  ```
当然，一些常用的值的设置，有简单的构造方式：
```java
      public HystrixCommandInstance(int id) {
          super(HystrixCommandGroupKey.Factory.asKey("ExampleGroup"), 500);
          this.id = id;
      }
```
  4. **Dynamic instance property**
      动态实例配置，是使用最多的方式就是使用动态设置instance配置的方式。
      example：
  ```java
      hystrix.command.HystrixCommandKey.execution.isolation.thread.timeoutInMilliseconds
  ```
      HystrixCommandKey会被你HystrixCommand的HystrixCommandKey.name()也就是commandKey替换。
---
 ## 配置使用
 这里使用注解动态配置实例的方式展示常用的配置，其他的配置级别可以参考interface中的配置demo配置。
 直接参考代码，注释部分是使用详情。

  ```java
  @HystrixCommand(
     //fallbackMethod name ,fallbackMethod name 支持重载，自动根据参数匹配，支持自动级联，支持同步异步{sync command,sync fallback}{async command, sync fallback}，{async command, async fallback}，默认同步顺序执行即可
     fallbackMethod = "ribbonGetUserFallback",
     //group key is used for grouping together commands,default = class name
     groupKey = "ConsumerTestAction",
     //default = the name of annotated method
     commandKey = "ribbonGetUser",
     //represent a HystrixThreadPool
     threadPoolKey = "hystrixThreadPool",
     commandProperties ={
         /*
          * Execution设置
          */
         //设置线程HystrixCommand执行的隔离策略
         //This property indicates which isolation strategy HystrixCommand.run() executes with, one of the following two choices: THREAD SEMAPHORE , default = THREAD
         //Generally the only time you should use semaphore isolation for HystrixCommands is when the call is so high volume (hundreds per second, per instance) that the overhead of separate threads is too high; this typically only applies to non-network calls.
         @HystrixProperty(name = "execution.isolation.strategy", value = "SEMAPHORE"),
         //设置commond执行超时时间，默认1000ms
         @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "1000"),
         //设置开关timeout，默认true
         @HystrixProperty(name = "execution.timeout.enabled", value = "true"),
         //设置发生超时是是否中断，默认true
         @HystrixProperty(name = "execution.isolation.thread.interruptOnTimeout", value = "true"),

         /*
          * 设置执行取消时是否执行中断操作，默认false, 当前版本不可用
          */
         @HystrixProperty(name = "execution.isolation.thread.interruptOnCancel", value = "false"),

         //设置最大请求数，只有在隔离策略SEMAPHORE时生效（the maximum number of requests allowed to a HystrixCommand.run() method）.默认10
         //If this maximum concurrent limit is hit then subsequent requests will be rejected.
         //理论上选择semaphore size的原则和选择thread size一致，但选用semaphore时每次执行的单元要比较小且执行速度快（ms级别），否则的话应该用thread。
         //The isolation principle is still the same so the semaphore should still be a small percentage of the overall container (i.e. Tomcat) thread pool, not all of or most of it, otherwise it provides no protection.
         @HystrixProperty(name = "execution.isolation.semaphore.maxConcurrentRequests", value = "10"),

         /*
          * Fallback设置，可以被用于THREAD和SEMAPHORE
          */

         //设置fallback方法的最大请求数（the maximum number of requests a HystrixCommand.getFallback() method is allowed to make from the calling thread.）。默认10
         @HystrixProperty(name = "fallback.isolation.semaphore.maxConcurrentRequests", value = "10"),
         //是否执行fallback方法，默认true
         @HystrixProperty(name = "fallback.enabled", value = "true"),

         /*
          *Circuit Breaker熔断设置
          */

         //是否开启熔断保护，默认true
         @HystrixProperty(name = "circuitBreaker.enabled", value = "true"),
         //set the minimum number of requests in a rolling window that will trip the circuit.
         //rolling window内最小的请求数。如果设为20，那么当一个rolling window的时间内（比如说1个rolling window是10秒）收到19个请求，即使19个请求都失败，也不会触发circuit break。默认20
         @HystrixProperty(name = "circuitBreaker.requestVolumeThreshold", value = "20"),
         //set the amount of time, after tripping the circuit, to reject requests before allowing attempts again to determine if the circuit should again be closed.
         //简单来说，设置扫描间隔，确定是否应该改变断路器状态
         @HystrixProperty(name = "circuitBreaker.sleepWindowInMilliseconds", value = "5000"),
         //设置错误比率阀值，如果错误率>=该值，circuit会被打开，并短路所有请求触发fallback。默认50
         @HystrixProperty(name = "circuitBreaker.errorThresholdPercentage", value = "50"),
         //设置强制打开断路器，拒绝所有请求，默认false
         @HystrixProperty(name = "circuitBreaker.forceOpen", value = "false"),
         //forces the circuit breaker into a closed state in which it will allow requests regardless of the error percentage.
         @HystrixProperty(name = "circuitBreaker.forceClosed", value = "false"),

         /*
          * Metrics 设置
          */

         //设置统计的时间窗口值的，毫秒值，circuit break 的打开会根据1个rolling window的统计来计算.默认10000
         @HystrixProperty(name = "metrics.rollingStats.timeInMilliseconds", value = "10000"),
         //设置rolling window 的分片数，默认10片。每个片包含success，failure，timeout，rejection的次数的统计信息。
         //必须满足rolling window % numberBuckets == 0
         @HystrixProperty(name = "metrics.rollingStats.numBuckets", value = "10"),
         //执行时是否enable指标的计算和跟踪，默认true
         @HystrixProperty(name = "metrics.rollingPercentile.enabled", value = "true"),
         //不清楚作用????????????????
         //sets the duration of the rolling window in which execution times are kept to allow for percentile calculations，默认60000
         @HystrixProperty(name = "metrics.rollingPercentile.timeInMilliseconds", value = "60000"),
         //设置rolling percentile window的分片数，默认6
         @HystrixProperty(name = "metrics.rollingPercentile.numBuckets", value = "6"),
         //如果bucket size＝100，window＝10s，若这10s里有500次执行，只有最后100次执行会被统计到bucket里去。增加该值会增加内存开销以及排序的开销。默认100
         @HystrixProperty(name = "metrics.rollingPercentile.bucketSize", value = "100"),
         //设置记录health 快照（用来统计成功和错误绿）的间隔，默认500ms
         @HystrixProperty(name = "metrics.healthSnapshot.intervalInMilliseconds", value = "500")
     },

     /*
      * control the behavior of the thread-pools that Hystrix Commands execute on.
      * Most of the time the default value of 10 threads will be fine (often it could be made smaller).
      * larger公式：
      *           每秒健康请求峰值×百分之99请求能完成的时间+一些空闲呼吸池
      *           如： 每秒峰值30请求量 ，百分之99的请求能在0.2秒内完成。30*0.2=6+空闲呼吸池4=10
      *           
      * 基本得原则时保持线程池尽可能小，他主要是为了释放压力，防止资源被阻塞。当一切都是正常的时候，线程池一般仅会有1到2个线程激活来提供服务
      */
     threadPoolProperties = {
         //并发执行的最大线程数，默认值10
         @HystrixProperty(name = "coreSize", value = "10"),
         /*
          * Added in 1.5.9.当前版本不可用
          * sets the maximum thread-pool size.
          * This is the maximum amount of concurrency that can be supported without starting to reject HystrixCommands.
          * Please note that this setting only takes effect if you also set allowMaximumSizeToDivergeFromCoreSize.
          * Prior to 1.5.9, core and maximum sizes were always equal.
          */
         @HystrixProperty(name = "maximumSize", value = "20"),
         /*
          * Added in 1.5.9.当前版本不可用
          * This property allows the configuration for maximumSize to take effect.
          * That value can then be equal to, or higher, than coreSize.
          * Setting coreSize < maximumSize creates a thread pool which can sustain maximumSize concurrency,
          * but will return threads to the system during periods of relative inactivity. (subject to keepAliveTimeInMinutes)
          */
         @HystrixProperty(name = "allowMaximumSizeToDivergeFromCoreSize ", value = "true"),
         /*
          * Added in 1.5.9.当前版本不可用
          * sets the keep-alive time, in minutes.默认1。
          * Prior to 1.5.9,all thread pools were fixed-size, as coreSize == maximumSize.
          * In 1.5.9 and after, setting allowMaximumSizeToDivergeFromCoreSize to true allows those 2 values to diverge,
          * such that the pool may acquire/release threads.
          * If coreSize < maximumSize, then this property controls how long a thread will go unused before being released.
          */

         @HystrixProperty(name = "keepAliveTimeMinutes", value = "1"),
         /*
          * 设置最大队列长度，默认值-1
          *   如果设置-1，则使用SynchronousQueue同步队列。SynchronousQueue是无界的，是一种无缓冲的等待队列，但是由于该Queue本身的特性，在某次添加元素后必须等待其他线程取走后才能继续添加；可以认为SynchronousQueue是一个缓存值为1的阻塞队列。
          *   如果设置为大于0的数，如100，使用LinkedBlockingQueue。new LinkedBlockingQueue(100)生成一个100队列长度的同步队列  LinkedBlockingQueue是无界的，是一个无界缓存的等待队列。基于链表的阻塞队列，内部维持着一个数据缓冲队列（该队列由链表构成）。
          *   当生产者往队列中放入一个数据时，队列会从生产者手中获取数据，并缓存在队列内部，而生产者立即返回；只有当队列缓冲区达到最大值缓存容量时（LinkedBlockingQueue可以通过构造函数指定该值），才会阻塞生产者队列。
          *   直到消费者从队列中消费掉一份数据，生产者线程会被唤醒，反之对于消费者这端的处理也基于同样的原理。
          */
         @HystrixProperty(name = "maxQueueSize", value = "20"),
         //一个可以动态修改队列长度的值，maxQueueSize不能被动态修改，这个参数将允许我们动态设置该值，达到queueSizeRejectionThreshold该值后，请求也会被拒绝。
         //note:如果设置 maxQueueSize == -1，该字段将不起作用
         @HystrixProperty(name = "queueSizeRejectionThreshold", value = "15"),


         /*
          * 并没有发现这两个配置放在threadPoolProperties和commandProperties的区别??????????????????????????
          */
         //分片数量，timeInMilliseconds需要整除numBuckets
         @HystrixProperty(name = "metrics.rollingStats.numBuckets", value = "10"),
         //线程池统计指标的时间，默认10000
         @HystrixProperty(name = "metrics.rollingStats.timeInMilliseconds", value = "10000")
     },
     //an ability to specify exceptions types which should be ignored without fallback logic.The exception will be wrapped in HystrixBadRequestException
     ignoreExceptions = {BaseRuntimeException.class
     },
     //all exceptions that are not ignored are raised as the cause of a HystrixRuntimeException
     raiseHystrixExceptions ={
         HystrixException.RUNTIME_EXCEPTION
     }
     )
 public User ribbonGetUser()
 {
     return this.restTemplate.getForObject("http://eureka-test-producer/getUser", User.class);
 }
```
