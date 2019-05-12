---
title: TaskExecutor Scheduling and Async
date: 2018-1-17 22:50:10
tags:
 -Spring
categories: Spring
thumbnail: https://raw.githubusercontent.com/apollochen123/image/master/%E9%BB%98%E8%AE%A42.jpg
---


# Spring TaskExecutor Scheduling and Async
Spring 提供了异步执行的TaskExeutor 和 TaskScheduler 的接口。对于有时候在web服务器想要定时任务时和异步请求接口，可以很简单的去完成。

-----
## What is Executor?
这里引用spring官方document 中的一段话：
>Executors are the JDK name for the concept of thread pools. The "executor" naming is due to the fact that there is no guarantee that the underlying implementation is actually a pool; an executor may be single-threaded or even synchronous. Spring’s abstraction hides implementation details between Java SE and Java EE environments.

Executors是JDK官方对于线程池的称呼，executor被命名是为了保证可能底层实现可能并不是一个池。一个executor可能是单线程的或者同步的。spring的抽象隐藏了JavaEE和SE的细节。

-----

## TaskExecutor
>The TaskExecutor was originally created to give other Spring components an abstraction for thread pooling where needed.

TaskExecutor 最初被创建主要是给spring其他组件提供一个线程池。

#### TaskExecutor Types
1. **SimpleAsyncTaskExecutor**
  >This implementation does not reuse any threads, rather it starts up a new thread for each invocation. However, it does support a concurrency limit which will block any invocations that are over the limit until a slot has been freed up. If you are looking for true pooling, see the discussions of SimpleThreadPoolTaskExecutor and ThreadPoolTaskExecutor below.

  简单异步执行器，这个没有重用任何的线程（不是池似，申请资源，使用完成归还），而是开一个新的线程给每个调用。支持并发线程限制。
   **Example**：
   ```java
   private static void testSimpleAsyncTaskExecutor()
 {
     SimpleAsyncTaskExecutor sexc = new SimpleAsyncTaskExecutor();
     //最多执行10个任务
     sexc.setConcurrencyLimit(10);
     ThreadFactory threadFactory = Executors.defaultThreadFactory();
     sexc.setThreadFactory(threadFactory);
     for (int i = 0; i < 30; i++)
     {
         System.out.println("提交了task" + i);
         sexc.execute(new MessagePrinterTask("I am printing" + i));
     }

 }
   ```
   **result**：
   ```
提交了task0
提交了task1
提交了task2
提交了task3
14时44分38秒--Thread : Thread[pool-2-thread-1,5,main]   I am printing0
提交了task4
14时44分38秒--Thread : Thread[pool-2-thread-2,5,main]   I am printing1
提交了task5
提交了task6
14时44分38秒--Thread : Thread[pool-2-thread-3,5,main]   I am printing2
提交了task7
提交了task8
14时44分38秒--Thread : Thread[pool-2-thread-4,5,main]   I am printing3
提交了task9
14时44分38秒--Thread : Thread[pool-2-thread-5,5,main]   I am printing4
提交了task10
14时44分38秒--Thread : Thread[pool-2-thread-6,5,main]   I am printing5
14时44分38秒--Thread : Thread[pool-2-thread-9,5,main]   I am printing8
     ...
提交了task29
14时44分58秒--Thread : Thread[pool-2-thread-29,5,main]   I am printing28
14时44分58秒--Thread : Thread[pool-2-thread-30,5,main]   I am printing29
   ```
根据结果可以看到，任务10个一组被执行，当执行的任务为10时，线程阻塞。

 **内部实现细则**：
  ```java
public void execute(Runnable task, long startTimeout) {
  Assert.notNull(task, "Runnable must not be null");
  Runnable taskToUse = (this.taskDecorator != null ? this.taskDecorator.decorate(task) : task);
  if (isThrottleActive() && startTimeout > TIMEOUT_IMMEDIATE) {
    this.concurrencyThrottle.beforeAccess();
    doExecute(new ConcurrencyThrottlingRunnable(taskToUse));
  }
  else {
    doExecute(taskToUse);
  }
}
  ```
  这段代码主要调用了beforeAccess()方法和包装了我们的RunnableTask。其实实现Task限制为10的功能主要就是通过beforeAccess()和afterAccess()实现的。
    ```java
  protected void beforeAccess() {
  if (this.concurrencyLimit == NO_CONCURRENCY) {
    throw new IllegalStateException(
        "Currently no invocations allowed - concurrency limit set to NO_CONCURRENCY");
  }
  if (this.concurrencyLimit > 0) {
    boolean debug = logger.isDebugEnabled();
    synchronized (this.monitor) {
      boolean interrupted = false;
      while (this.concurrencyCount >= this.concurrencyLimit) {
        if (interrupted) {
          throw new IllegalStateException("Thread was interrupted while waiting for invocation access, " +
              "but concurrency limit still does not allow for entering");
        }
        if (debug) {
          logger.debug("Concurrency count " + this.concurrencyCount +
              " has reached limit " + this.concurrencyLimit + " - blocking");
        }
        try {
          this.monitor.wait();
        }
        catch (InterruptedException ex) {
          // Re-interrupt current thread, to allow other threads to react.
          Thread.currentThread().interrupt();
          interrupted = true;
        }
      }
      if (debug) {
        logger.debug("Entering throttle at concurrency count " + this.concurrencyCount);
      }
      this.concurrencyCount++;
    }
  }
}
  ```
  这段源码使用对象锁实现了使用concurrencyLimit限制任务同时执行的数量。
  ```Java
  private class ConcurrencyThrottlingRunnable implements Runnable {

  private final Runnable target;

  public ConcurrencyThrottlingRunnable(Runnable target) {
    this.target = target;
  }

  @Override
  public void run() {
    try {
      this.target.run();
    }
    finally {
      concurrencyThrottle.afterAccess();
    }
  }
}
  ```
  这个Runnable的包装类，增强了afterAccess的自动调用
  ```Java
  protected void afterAccess() {
  if (this.concurrencyLimit >= 0) {
    synchronized (this.monitor) {
      this.concurrencyCount--;
      if (logger.isDebugEnabled()) {
        logger.debug("Returning from throttle at concurrency count " + this.concurrencyCount);
      }
      this.monitor.notify();
    }
  }
}
  ```
  其实看到这，实现原理就已经知道很明了了。使用了一个wait and notify模型，实现了控制并发线程数。
2. **SyncTaskExecutor**
 >This implementation doesn’t execute invocations asynchronously. Instead, each invocation takes place in the calling thread. It is primarily used in situations where multi-threading isn’t necessary such as simple test cases.

  同步执行器，不会新开线程，就在当前线程执行，适用于简单测试场景。
  **example**
  ```java
  private static void testSyncTaskExecutor()
{
    SyncTaskExecutor sexc = new SyncTaskExecutor();
    for (int i = 0; i < 30; i++)
    {
        System.out.println("提交了task" + i);
        sexc.execute(new MessagePrinterTask("I am printing" + i));
    }
}
  ```
  **result**
  ```
  提交了task0
15时28分26秒--Thread : Thread[main,5,main]   I am printing0
提交了task1
15时28分36秒--Thread : Thread[main,5,main]   I am printing1
提交了task2
15时28分46秒--Thread : Thread[main,5,main]   I am printing2
  ```
  **实现方式**
  ```java
  public void execute(Runnable task) {
		Assert.notNull(task, "Runnable must not be null");
		task.run();
	}
  ```
  简单粗暴，直接run。
3. **ConcurrentTaskExecutor**
  >This implementation is an adapter for a java.util.concurrent.Executor object. There is an alternative, ThreadPoolTaskExecutor, that exposes the Executor configuration parameters as bean properties. It is rare to need to use the ConcurrentTaskExecutor, but if the ThreadPoolTaskExecutor isn’t flexible enough for your needs, the ConcurrentTaskExecutor is an alternative

  这个执行器是对java.util.concurrent.Executor的调整，不常用。可用bean配置。
  **example**
  ```java
  private static void testConcurrentTaskExecutor()
{
    ConcurrentTaskExecutor cexc = new ConcurrentTaskExecutor();
    BlockingQueue<Runnable> queue = new LinkedBlockingQueue<Runnable>(19);
    ThreadPoolExecutor exc = new ThreadPoolExecutor(3, 30, 30000, TimeUnit.SECONDS, queue);
    cexc.setConcurrentExecutor(exc);
    for (int i = 0; i < 10; i++)
    {
        System.out.println("提交了task" + i);
        cexc.execute(new MessagePrinterTask("I am printing" + i));
    }
}
  ```
  这里Executor使用ThreadPoolExecutor。
  **result**
  ```
  提交了task0
提交了task1
提交了task2
提交了task3
15时37分21秒--Thread : Thread[pool-2-thread-1,5,main]   I am printing0
提交了task4
提交了task5
15时37分21秒--Thread : Thread[pool-2-thread-3,5,main]   I am printing2
15时37分21秒--Thread : Thread[pool-2-thread-2,5,main]   I am printing1
提交了task6
提交了task7
提交了task8
提交了task9
15时37分31秒--Thread : Thread[pool-2-thread-1,5,main]   I am printing3
15时37分31秒--Thread : Thread[pool-2-thread-3,5,main]   I am printing4
15时37分31秒--Thread : Thread[pool-2-thread-2,5,main]   I am printing5
15时37分41秒--Thread : Thread[pool-2-thread-1,5,main]   I am printing6
15时37分41秒--Thread : Thread[pool-2-thread-2,5,main]   I am printing8
15时37分41秒--Thread : Thread[pool-2-thread-3,5,main]   I am printing7
15时37分51秒--Thread : Thread[pool-2-thread-1,5,main]   I am printing9

  ```
4. **SimpleThreadPoolTaskExecutor**
  >This implementation is actually a subclass of Quartz’s SimpleThreadPool which listens to Spring’s lifecycle callbacks. This is typically used when you have a thread pool that may need to be shared by both Quartz and non-Quartz components.
5. **ThreadPoolTaskExecutor**
  >This implementation is the most commonly used one. It exposes bean properties for configuring a java.util.concurrent.ThreadPoolExecutor and wraps it in a TaskExecutor. If you need to adapt to a different kind of java.util.concurrent.Executor, it is recommended that you use a ConcurrentTaskExecutor instead.

  最常用的的执行器，如果需要换java.util.concurrent.java.util.concurrent.Executor，可以使用ConcurrentTaskExecutor。
  **example**
  ```java
  private static void testThreadPoolTaskExecutor()
{
    RejectedExecutionHandler rejectionHandler = new RejectHandler();
    //Get the ThreadFactory implementation to use
    ThreadFactory threadFactory = Executors.defaultThreadFactory();
    ThreadPoolTaskExecutor exc = new ThreadPoolTaskExecutor();
    exc.setThreadFactory(threadFactory);
    exc.setRejectedExecutionHandler(rejectionHandler);

    exc.setQueueCapacity(4);
    exc.setCorePoolSize(5);
    exc.setMaxPoolSize(7);
    exc.initialize();

    for (int i = 0; i < 11; i++)
    {
        System.out.println("提交了task" + i);
        exc.execute(new MessagePrinterTask("I am printing" + i));
    }
    exc.getThreadPoolExecutor().shutdown();
}
  ```
  这里，重点介绍一下CorePoolSize和MaxPoolSize。
  JavaDoc:
  >When a new task is submitted [...], and fewer than corePoolSize threads are running, a new thread is created to handle the request, even if other worker threads are idle. If there are more than corePoolSize but less than  maximumPoolSize threads running, a new thread will be created only if the queue is full. By setting  corePoolSize and maximumPoolSize the same, you create a fixed-size thread pool. By setting  maximumPoolSize to an essentially unbounded value such as  Integer.MAX_VALUE, you allow the pool to accommodate an arbitrary number of concurrent tasks.

  总的来说，就是CorePoolSize是pool初始值的大小，MaxPoolSize是最大pool的大小，这两个值与队列长度也有关。当有11个task时，首先前五个开始执行，然后开始添加队列4个task，发现队列占满了，还有2个任务没法执行，此时扩充pool大小，扩充到7恰好能满足任务。只有超过了队列长度，pool的size才开始增加。
  ```
提交了task0
提交了task1
提交了task2
提交了task3
提交了task4
提交了task5
15时43分48秒--Thread : Thread[pool-1-thread-2,5,main]   I am printing1
提交了task6
15时43分48秒--Thread : Thread[pool-1-thread-3,5,main]   I am printing2
15时43分48秒--Thread : Thread[pool-1-thread-1,5,main]   I am printing0
提交了task7
提交了task8
提交了task9
15时43分48秒--Thread : Thread[pool-1-thread-4,5,main]   I am printing3
15时43分48秒--Thread : Thread[pool-1-thread-5,5,main]   I am printing4
提交了task10
15时43分48秒--Thread : Thread[pool-1-thread-6,5,main]   I am printing9
15时43分48秒--Thread : Thread[pool-1-thread-7,5,main]   I am printing10
15时43分58秒--Thread : Thread[pool-1-thread-2,5,main]   I am printing6
15时43分58秒--Thread : Thread[pool-1-thread-3,5,main]   I am printing7
15时43分58秒--Thread : Thread[pool-1-thread-1,5,main]   I am printing5
15时43分58秒--Thread : Thread[pool-1-thread-7,5,main]   I am printing8
  ```
  QueueCapacity=4；CorePoolSize=5;MaxPoolSize=7;11个task，7+4的方式执行。
假如MaxPoolSize+QueueCapacity< task的数量，会报错：
 比如我们修改队列长度为QueueCapacity=3，
 ```
提交了task0
提交了task1
提交了task2
提交了task3
提交了task4
16时04分53秒--Thread : Thread[pool-1-thread-1,5,main]   I am printing0
16时04分53秒--Thread : Thread[pool-1-thread-2,5,main]   I am printing1
提交了task5
16时04分53秒--Thread : Thread[pool-1-thread-3,5,main]   I am printing2
提交了task6
提交了task7
提交了task8
16时04分53秒--Thread : Thread[pool-1-thread-5,5,main]   I am printing4
提交了task9
16时04分53秒--Thread : Thread[pool-1-thread-4,5,main]   I am printing3
提交了task10
16时04分53秒--Thread : Thread[pool-1-thread-6,5,main]   I am printing8
16时04分53秒--Thread : Thread[pool-1-thread-7,5,main]   I am printing9
Exception in thread "main"
org.springframework.core.task.TaskRejectedException: Executor [java.util.concurrent.ThreadPoolExecutor@5c1a8622[Running, pool size = 7, active threads = 7, queued tasks = 3, completed tasks = 0]] did not accept task: com.sap.csc.ems.configuration.util.TaskExecutorTest$MessagePrinterTask@5ad851c9
	at org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor.execute(ThreadPoolTaskExecutor.java:296)
	at com.sap.csc.ems.configuration.util.TaskExecutorTest.testThreadPoolTaskExecutor(TaskExecutorTest.java:126)
	at com.sap.csc.ems.configuration.util.TaskExecutorTest.main(TaskExecutorTest.java:49)
Caused by: java.util.concurrent.RejectedExecutionException: 我满了,I am printing10失败
	at com.sap.csc.ems.configuration.util.TaskExecutorTest$RejectHandler.rejectedExecution(TaskExecutorTest.java:163)
	at java.util.concurrent.ThreadPoolExecutor.reject(ThreadPoolExecutor.java:823)
	at java.util.concurrent.ThreadPoolExecutor.execute(ThreadPoolExecutor.java:1369)
	at org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor.execute(ThreadPoolTaskExecutor.java:293)
	... 2 more
16时05分03秒--Thread : Thread[pool-1-thread-1,5,main]   I am printing5
16时05分03秒--Thread : Thread[pool-1-thread-2,5,main]   I am printing6
16时05分03秒--Thread : Thread[pool-1-thread-3,5,main]   I am printing7

 ```
 第11个任务被拒绝。
 **实现细节**
 这个ThreadPoolTaskExecutor其实就是java.util.concurrent.ThreadPoolExecutor的包装。

------

## TaskScheduler
 此部分来至[spring 官方文档](https://docs.spring.io/spring/docs/5.0.3.BUILD-SNAPSHOT/spring-framework-reference/integration.html#scheduling)

 >In addition to the TaskExecutor abstraction, Spring 3.0 introduces a TaskScheduler with a variety of methods for scheduling tasks to run at some point in the future.

 ```java
 public interface TaskScheduler {

    ScheduledFuture schedule(Runnable task, Trigger trigger);

    ScheduledFuture schedule(Runnable task, Date startTime);

    ScheduledFuture scheduleAtFixedRate(Runnable task, Date startTime, long period);

    ScheduledFuture scheduleAtFixedRate(Runnable task, long period);

    ScheduledFuture scheduleWithFixedDelay(Runnable task, Date startTime, long delay);

    ScheduledFuture scheduleWithFixedDelay(Runnable task, long delay);

}
 ```
 >The simplest method is the one named 'schedule' that takes a Runnable and Date only. That will cause the task to run once after the specified time. All of the other methods are capable of scheduling tasks to run repeatedly. The fixed-rate and fixed-delay methods are for simple, periodic execution, but the method that accepts a Trigger is much more flexible.

### QuickStart
 1. 添加注解在springboot启动类
 ```java
@SpringBootApplication
@EnableScheduling
public class MiddlewareMain
{
    public static void main(String[] args)
    {
        SpringApplication.run(MiddlewareMain.class, args);
    }
}
 ```
 2. 在你需要定时的方法上添加@Scheduled注解
 ```java
 @Component
public class UploadData
{   
      @Scheduled(initialDelay=10000,fixedRate=4000)
      public void sendData()
      {
        System.out.println("Thread:"+Thread.currentThread()+"开始");
        System.out.println("Thread:"+Thread.currentThread()+"---"+dateFormat.format(System.currentTimeMillis())+"--- 我正在执行sendData!");
        try
        {
            Thread.sleep(6000);
        }
        catch (InterruptedException e)
        {
             e.printStackTrace();  
        }
        System.out.println("Thread:"+Thread.currentThread()+"结束");
      }
}
 ```

### Use Annotation Config Scheduling
@Scheduled注解配置：
设置每间隔5秒执行一次，参数为long。任务结束后开始计时，也就是上一个任务0秒开始，执行了10秒，下一个任务第15秒开始执行。
```Java
@Scheduled(fixedDelay=5000)
public void doSomething() {
    // something that should execute periodically
}
```
设置每间隔5秒执行一次，参数为long。任务开始后开始计时，也就是上一个任务0秒开始，执行了2秒，下一个任务第5秒开始执行。
```Java
@Scheduled(fixedRate=5000)
public void doSomething() {
    // something that should execute periodically
}
```
设置每间隔5秒执行一次，参数为long，第一次执行延迟10秒开始。任务开始后开始计时，也就是上一个任务0秒开始，执行了2秒，下一个任务第5秒开始执行。
```Java
@Scheduled(initialDelay=10000,fixedRate=5000)
public void doSomething() {
    // something that should execute periodically
}
```
使用cron表达式
Seconds Minutes Hours DayofMonth Month DayofWeek Year
或 Seconds Minutes Hours DayofMonth Month DayofWeek
```java
@Scheduled(cron="*/5 * * * * MON-FRI")
public void doSomething()
     // something that should execute periodically
}
```
详细参考：[百度经验](https://jingyan.baidu.com/article/7f41ecec0d0724593d095c19.html)
 0 0 10,14,16 * * ? 每天上午10点，下午2点，4点
 0 0/30 9-17 * * ? 朝九晚五工作时间内每半小时
 0 15 10 ? * * 每天上午10:15触发
 0 15 10 * * ? 每天上午10:15触发
 0 15 10 * * ? * 每天上午10:15触发
 0 15 10 * * ? 2005 2005年的每天上午10:15触发
 0 * 14 * * ? 在每天下午2点到下午2:59期间的每1分钟触发
 0 0/5 14 * * ? 在每天下午2点到下午2:55期间的每5分钟触发
 0 0/5 14,18 * * ? 在每天下午2点到2:55期间和下午6点到6:55期间的每5分钟触发
 0 0-5 14 * * ? 在每天下午2点到下午2:05期间的每1分钟触发
 0 10,44 14 ? 3 WED 每年三月的星期三的下午2:10和2:44触发
 0 15 10 ? * MON-FRI 周一至周五的上午10:15触发
 0 15 10 15 * ? 每月15日上午10:15触发
 0 15 10 L * ? 每月最后一日的上午10:15触发
 0 15 10 ? * 6L 每月的最后一个星期五上午10:15触发
 0 15 10 ? * 6L 2002-2005 2002年至2005年的每月的最后一个星期五上午10:15触发
 0 15 10 ? * 6#3 每月的第三个星期五上午10:15触发

### SchedulingConfigurer
使用SchedulingConfigurer可以配置你的定时任务执行的方式，可以使用你想的TaskExecutor.下面是一个例子：
```Java
@Configuration
public class ScheduleConfig implements SchedulingConfigurer
{
    @Override
    public void configureTasks(ScheduledTaskRegistrar taskRegistrar)
    {
        ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
        scheduler.setPoolSize(5);
        scheduler.initialize();
        //申请5个线程池给Schedule任务
        taskRegistrar.setScheduler(scheduler);
    }
}
```
当然你也可以放弃使用注解@Scheduled配置你想要执行的任务，你可以在SchedulingConfigurer中直接配置你想要执行的任务：
```Java
@Configuration
public class ScheduleConfig implements SchedulingConfigurer
{
    @Override
    public void configureTasks(ScheduledTaskRegistrar taskRegistrar)
    {
        ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
        scheduler.setPoolSize(5);
        scheduler.initialize();
        //申请5个线程池给Schedule任务
        taskRegistrar.setScheduler(scheduler);
        taskRegistrar.addFixedRateTask(new IntervalTask(
            new Runnable() {
                @Override
                public void run() {
                    myUploadData().sendData();
                }
            },
            10000, 0));
    }
}
```
------
## Async
@Async注解可以使方法变为异步执行，使用它可以简单的将同步请求变为一个异步请求。
### QuickStart
1. 在SpringApplication启动类上加上注解
```java
@SpringBootApplication
@EnableAsync
public class MiddlewareMain
{
    public static void main(String[] args)
    {
        SpringApplication.run(MiddlewareMain.class, args);
    }
}
```
2. 在你想要异步执行的方法上加上@Async注解
```java
@Async
public void testAsync(String a){
  System.out.println("Thread:"+Thread.currentThread()+"开始");
  System.out.println("Thread:"+Thread.currentThread()+"---"+dateFormat.format(System.currentTimeMillis())+"--- 我正在执行sendData!"+a);
  try
  {
      Thread.sleep(10000);
  }
  catch (InterruptedException e)
  {
       e.printStackTrace();  
  }
  if("AAAA".equals(a)){
      throw new RuntimeException("666666666666666666666666666");
  }
  System.out.println("Thread:"+Thread.currentThread()+"结束");
}
```

### AsyncConfigurer interfaces
1. 配置Eexcutor
@Async注解，spring默认使用SimpleAsyncTaskExecutor执行你的task
```
Thread:Thread[SimpleAsyncTaskExecutor-1,5,main]开始
Thread:Thread[SimpleAsyncTaskExecutor-1,5,main]---2018-01-17 16时03分43秒 --- 我正在执行sendData2!AAAA1
Thread:Thread[SimpleAsyncTaskExecutor-1,5,main]结束
```
AsyncConfigurer接口可以改变你异步任务的Executor,我们可以使用
```java
@Override
@Configuration
public class ExcutorConfig implements AsyncConfigurer
{
    @Override
    public Executor getAsyncExecutor()
    {
        ThreadPoolTaskScheduler executor = new ThreadPoolTaskScheduler();
        executor.setBeanName("myExecutor");
        executor.setPoolSize(5);
        executor.initialize();
        return executor;
    }
    ...
}
```
可以在控制台看到执行器改变：
```
Thread:Thread[myExecutor-1,5,main]开始
Thread:Thread[myExecutor-1,5,main]---2018-01-17 16时09分19秒 --- 我正在执行sendData!AAAA1
Thread:Thread[myExecutor-1,5,main]结束
```
2. 配置ExceptionHandler
```java
@Configuration
public class ExcutorConfig implements AsyncConfigurer
{
    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler()
    {
        return new AsyncUncaughtExceptionHandler()
        {
            @Override
            public void handleUncaughtException(Throwable ex, Method method, Object... params)
            {
                System.out.println("methodName:"+method.getName()+"   Modifier"+method.getModifiers()+"  param:"+(String)params[0]);
                System.out.println("执行出错了！！！"+ex);
            }
        };
    }
}
```
console：
```
Thread:Thread[myExecutor-1,5,main]开始
Thread:Thread[myExecutor-1,5,main]---2018-01-17 16时13分09秒 --- 我正在执行sendData!AAAA
methodName:testAsync   Modifier1  param:AAAA
执行出错了！！！java.lang.RuntimeException: 666666666666666666666666666
```
3. @Async支持namespace配置，支持不同任务使用不同的Executor
配置一个id为secondExecutor的Executor
```java
@Bean("secondExecutor")
public Executor MyExecutor()
{
    ThreadPoolTaskScheduler beanExecutor = new ThreadPoolTaskScheduler();
    beanExecutor.setBeanName("secondExecutor");
    beanExecutor.setPoolSize(5);
    beanExecutor.initialize();
    return beanExecutor;
}
```
使用secondExecutor
```Java
@Async("secondExecutor")
public void testAsync(String a){
  System.out.println("Thread:"+Thread.currentThread()+"开始");
  System.out.println("Thread:"+Thread.currentThread()+"---"+dateFormat.format(System.currentTimeMillis())+"--- 我正在执行sendData!"+a);
  try
  {
      Thread.sleep(10000);
  }
  catch (InterruptedException e)
  {
       e.printStackTrace();  
  }
  if("AAAA".equals(a)){
      throw new RuntimeException("666666666666666666666666666");
  }
  System.out.println("Thread:"+Thread.currentThread()+"结束");
}
```
console:
```
Thread:Thread[secondExecutor-1,5,main]开始
Thread:Thread[secondExecutor-1,5,main]---2018-01-17 16时18分01秒 --- 我正在执行sendData!AAAA1
Thread:Thread[secondExecutor-1,5,main]结束
```
4. 使用Future
```java
@Async
public Future<String> testAsync2(String a){
  System.out.println("Thread2:"+Thread.currentThread()+"开始");
  System.out.println("Thread2:"+Thread.currentThread()+"---"+dateFormat.format(System.currentTimeMillis())+"--- 我正在执行sendData2!"+a);
  try
  {
      Thread.sleep(10000);
  }
  catch (InterruptedException e)
  {
       e.printStackTrace();  
  }
  if("AAAA".equals(a)){
      throw new RuntimeException("22222222222222");
  }
  System.out.println("Thread2:"+Thread.currentThread()+"结束");
  //return CompletableFuture.completedFuture(a);
  return new AsyncResult<String>(a);
}
```
主线程等待执行完成：
```java
@RequestMapping("/testAsync2")
public String hello2(@PathParam("a") String a){
    Future<String> f = uploadData.testAsync2(a);
    try
    {
        System.out.println(f.get());
    }
    catch (InterruptedException | ExecutionException e)
    {
         e.printStackTrace();  
    }
    return "OK2";
}
```
