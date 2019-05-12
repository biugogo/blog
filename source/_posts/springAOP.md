---
title: Spring AOP
date: 2017-12-15 21:47:51
tags:
 -Spring
categories: Spring
thumbnail: https://raw.githubusercontent.com/apollochen123/image/master/%E9%BB%98%E8%AE%A42.jpg
---

# **spring AOP**
This page just is written for a reference when forget. I won't tell you what is AOP.

### Attention To
&emsp;&emsp;spring AOP is based on JVM dynamic proxy, so only class implements interface can be proxied. And if there is no interface of class ,spring AOP will use CGLIB(Code Generation Library) to write a proxy subclass.
&emsp;&emsp;so we can get a conclusion that spring AOP can't proxy private method.(subclass can get private method and interface also can't have private method)
<font color=#gray  size=5 face="黑体">spring AOP无法拦截private的方法</font>

-----
### Keyword
1. **Aspect**: An aspect is a class that implements enterprise application concerns that cut across multiple classes, such as transaction management. Aspects can be a normal class configured through Spring XML configuration or we can use Spring AspectJ integration to define a class as Aspect using @Aspect annotation.
2.  **Join Point**: A join point is the specific point in the application such as method execution, exception handling, changing object variable values etc. In Spring AOP a join points is always the execution of a method.
3.  **Advice**: Advices are actions taken for a particular join point. In terms of programming, they are methods that gets executed when a certain join point with matching pointcut is reached in the application. You can think of Advices as Struts2 interceptors or Servlet Filters.
4.  **Pointcut**: Pointcut are expressions that is matched with join points to determine whether advice needs to be executed or not. Pointcut uses different kinds of expressions that are matched with the join points and Spring framework uses the AspectJ pointcut expression language.
Target Object: They are the object on which advices are applied. Spring AOP is implemented using runtime proxies so this object is always a proxied object. What is means is that a subclass is created at runtime where the target method is overridden and advices are included based on their configuration.
5.  **AOP proxy**: Spring AOP implementation uses JDK dynamic proxy to create the Proxy classes with target classes and advice invocations, these are called AOP proxy classes. We can also use CGLIB proxy by adding it as the dependency in the Spring AOP project.
6.  **Weaving**: It is the process of linking aspects with other objects to create the advised proxy objects. This can be done at compile time, load time or at runtime. Spring AOP performs weaving at the runtime.

### AOP Advice Types：
- **Before Advice**: These advices runs before the execution of join point methods. We can use @Before annotation to mark an advice type as Before advice.
- **After (finally) Advice**: An advice that gets executed after the join point method finishes executing, whether normally or by throwing an exception. We can create after advice using @After annotation.
- **After Returning Advice**: Sometimes we want advice methods to execute only if the join point method executes normally. We can use @AfterReturning annotation to mark a method as after returning advice.
- **After Throwing Advice**: This advice gets executed only when join point method throws exception, we can use it to rollback the transaction declaratively. We use @AfterThrowing annotation for this type of advice.
- **Around Advice**: This is the most important and powerful advice. This advice surrounds the join point method and we can also choose whether to execute the join point method or not. We can write advice code that gets executed before and after the execution of the join point method. It is the responsibility of around advice to invoke the join point method and return values if the method is returning something. We use @Around annotation to create around advice methods.

做个总结（个人意见）：Before主要是进入方法前增强；After主要是用在方法结束后增强；After returning主要是对方法正常返回值进行处理，比如改变返回值的值；After Throwing主要是对方法异常进行处理。Around是可以对前四个方式进行组合，是最强的。

----
### Pointcut
#### Declaring Pointcut

There is three ways to declare Pointcut, two methods is based on annotation and the last is based on XML.

```java
//method 1：
@Pointcut("execution(public String com.chen.eureka.producer.service.ProducerServiceImpl.*(..))")
public void pointCut()
{
};

@Before("pointCut()")
@Order(2)
public void beforeAspect()
{
    System.out.println("this is before Aspect 1 !");
}

//method 2:
@Before("execution(* com.chen.eureka.producer.service.ProducerService.*(..))")
@Order(3)
public void beforeAspect2()
{
    System.out.println("this is before Aspect 2 !");
}

//method 3：

/**
 * <aop:config>
 * <aop:pointcut id="anyMethod" expression="@target(org.springframework.stereotype.Repository)" order="1"/>
 * </aop:config>
 */
```

#### Nine types of pointcut
1. **execution** - for matching method execution join points, this is the primary pointcut designator you will use when working with Spring AOP
```java
@Pointcut("execution(String com.chen.eureka.producer.service.ProducerService.getTest(String))")
public void executionPointCut(){}
```
定位于com.chen.eureka.producer.service包的ProducerService接口的getTest（）的方法，方法返回值为String.class，参数也只能是一个String.class。the frist String means return string ，the second String means the args is String.class.
```java
@Pointcut("execution(* com.chen.eureka.producer.service.ProducerService.*2(..))")
public void executionPointCut2(){}
```
定位于com.chen.eureka.producer.service包ProducerService接口，匹配所有结尾为2的方法，如getTest2（）.第一个*代表所有返回值，第二个*代表ProducerService的所有方法，（..）代表不管什么参数。
2. **within** - limits matching to join points within certain types (simply the execution of a method declared within a matching type when using Spring AOP)
```java
@Pointcut("within(com.chen.eureka.producer.service.ProducerService)")
public void withinPointCut(){}
```
定位于com.chen.eureka.producer.service包ProducerService接口下，所有方法。
```java
@Pointcut("within(com.chen.eureka.producer..*)")
public void withinPointCut2(){}
```
定位于com.chen.eureka.producer包下和其子包下所有类的所有方法。
3. **this** - limits matching to join points (the execution of methods when using Spring AOP) where the bean reference (Spring AOP proxy) is an instance of the given type
```java
@Pointcut("this(com.chen.eureka.producer.service.ProducerServiceImpl)")
public void thisPointCut(){}
```
this的情景用于没有实现任何接口的类。
4. **target** - limits matching to join points (the execution of methods when using Spring AOP) where the target object (application object being proxied) is an instance of the given type
```java
@Pointcut("target(com.chen.eureka.producer.service.ProducerService)")
public void targetPointCut(){}
```
定位于实现ProducerService接口的所有类。
5. **args** - limits matching to join points (the execution of methods when using Spring AOP) where the arguments are instances of the given types
```java
@Before("execution(* ong.customer.bo.CustomerBo.addCustomer(String)), && args(inputString)")
public void logBefore2(JoinPoint joinPoint, String inputString) {
    System.out.println(inputString);
 }
```
6. **@target** - limits matching to join points (the execution of methods when using Spring AOP) where the class of the executing object has an annotation of the given type.
```java
@Pointcut("@target(org.springframework.stereotype.Service)")
public void atargetPointCut(){}
```
定位于所有标注了@Service注解的类的方法.
7. **@args** - limits matching to join points (the execution of methods when using Spring AOP) where the runtime type of the actual arguments passed have annotations of the given type(s)
```java
@Pointcut("@args(com.chen.eureka.producer.aspect.ParamAnnotation, ..)")
public void aargsPointCut(){}
```
定位于，参数是带有@ParamAnnotation注解的类.的方法。方法参数为User类，而User类带有@ParamAnnotation注解，则切入此方法。
8. **@within** - limits matching to join points within types that have the given annotation (the execution of methods declared in types with the given annotation when using Spring AOP)
```java
@Pointcut("@within(org.springframework.stereotype.Service)")
public void awithinPointCut(){}
```
定位于持有@Service注解的class的方法，接口不起作用。
9. **@annotation** - limits matching to join points where the subject of the join point (method being executed in Spring AOP) has the given annotation
```java
@Pointcut("@annotation(com.chen.eureka.producer.aspect.MyAnnotation)")
public void aannotationPointCut(){}
```
持有@MyAnnotation的方法。
------
### Some Example
##### before+@args+execution
controller：
```java
@GetMapping("/aoptest2/{in}")
public String aopTest2(@PathVariable String in)
{
    User u = new User();
    u.setAddress("address");
    return producerService.beforeTest(u, in);
}
```
service:
```java
@Override
public String beforeTest(User u, String in)
{
    System.out.println("This is beforeTest..."+u.getAddress());
    return "beforeTest result";
}
```
Aspect
```java
@Before("@args(paramAnnotation, ..) && execution(* com.chen.eureka.producer.service.ProducerService.before*(..))")
public void beforeAndAargsAspect(JoinPoint thisJoinPoint, ParamAnnotation paramAnnotation)
{
    System.out.println("beforeAndAargsAspect："+paramAnnotation.toString());
}
```
参数类User@args(paramAnnotation, ..)
```java
@ParamAnnotation
public class User {
	private String id;
	private String name;
	private String address;
    ...
}
```
result：
```
beforeAndAargsAspect：@com.chen.eureka.producer.aspect.ParamAnnotation()
This is beforeTest...
```
##### after+execution
controller:
```java
@GetMapping("/aoptest/{in}")
public String aopTest(@PathVariable String in)
{
    return producerService.afterTest(in);
}
```
service
```java
@Override
public String afterTest(String in)
{
     System.out.println("This is afterTest...");
     return "afterTest result";
}
```
Aspect:
```java
@After("execution(* com.chen.eureka.producer.service.ProducerService.afterTest(..))")
public void AfterAspect(){
    System.out.println("AfterAndAnotationAspect： clear resource");
}
```
result:
```
This is afterTest...
AfterAndAnotationAspect： clear resource
```
##### AfterReturning+@Annotation
controller:
```java
@GetMapping("/aoptest3/{in}")
public String aopTest3(@PathVariable String in)
{
    return producerService.afterRunningTest(in).getId();
}
```
service
```java
@MyAnnotation
public User afterRunningTest(String in)
{
    System.out.println("This is afterRunningTest...");
    User user = new User();
    user.setId("id");
    System.out.println("This is orginal id:"+user.getId());
    return user;
}
```
Aspect:
```java
@AfterReturning(returning="rvt",pointcut="@annotation(com.chen.eureka.producer.aspect.MyAnnotation)")
public void AfterReturningAndAnotationAspect(Object rvt){
    User u  = (User) rvt;
    System.out.println("AfterReturningAndAnotationAspect： 正在处理返回值·····:"+u.getId());
    u.setId("new Id");
    System.out.println("返回值被改变为:"+u.getId());
}
```
result:
```
This is afterTest...
AfterAndAnotationAspect： clear resource
This is afterRunningTest...
This is orginal id:id
AfterReturningAndAnotationAspect： 正在处理返回值·····:id
返回值被改变为:new Id
```
这个例子有点注意，public void AfterReturningAndAnotationAspect(Object rvt)的rvt是切入函数的返回值，是一个引用。所以，rvt=rvt+1；不会改变返回值，他只是使rvt引用指向了其他对象。所以返回值处理不能处理String，Integer等对象
##### AfterThrowingAdvice+@target
controller:
```java
@GetMapping("/aoptest4/{in}")
public String aopTest4(@PathVariable String in) throws Exception
{
    return producerService.afterThrowingTest(in);
}
```
service
```java
@Override
public String afterThrowingTest(String in) throws Exception
{
    System.out.println("This is afterThrowingTest...");
    System.out.println("I have throwed a Exception");
    throw new Exception("I have throwed a Exception");
}
```
Aspect:
```java
@AfterThrowing(throwing="ex",pointcut="@target(org.springframework.stereotype.Service) && within(com.chen.eureka.producer..*)")
public void AfterThrowingAndAtarget(Throwable ex){
    System.out.println("AfterThrowingAndAtarget： 捕捉到异常·····"+ex);
    System.out.println("AfterThrowingAndAtarget 模拟异常处理");
}
```
result:
```
This is afterThrowingTest...
I have throwed a Exception
AfterThrowingAndAtarget： 捕捉到异常·····java.lang.Exception: I have throwed a Exception
AfterThrowingAndAtarget 模拟异常处理
```
##### Around+within+@annotation
controller:
```java
@GetMapping("/aoptest5/{in}")
public String aopTest5(@PathVariable String in)
{
    return producerService.aroundTest(in);
}
```
service
```java
@Override
@MyAnnotation
public String aroundTest(String in)
{
     System.out.println("This is aroundTest");
     return "aroundTest result";
}
```
Aspect:
```java
@Around("@annotation(myAnnotation)")
public Object AroundAndAannotation(ProceedingJoinPoint point, MyAnnotation myAnnotation){
    Object object = null;
    System.out.println("正在使用Around进行Before增强···");
    try
    {
        System.out.println("shortString"+point.toShortString());
        System.out.println("LongString:"+point.toLongString());
        System.out.println("第一个参数是："+point.getArgs()[0]);
        object = point.proceed();
        System.out.println("正在使用Around进行AfterReturning增强···");
        object = "增强型Object"+object;
    }
    catch (Throwable e)
    {
        System.out.println("正在使用Around进行AfterThrowing增强···");
         e.printStackTrace();  
    }finally{
        System.out.println("正在使用Around进行After增强···");
        System.out.println("资源正在释放······");
    }
    return object;
}
```
result:
```
正在使用Around进行Before增强···
shortStringexecution(ProducerServiceImpl.aroundTest(..))
LongString:execution(public java.lang.String com.chen.eureka.producer.service.ProducerServiceImpl.aroundTest(java.lang.String))
第一个参数是：test2
This is aroundTest
正在使用Around进行AfterReturning增强···
正在使用Around进行After增强···
资源正在释放······
增强型ObjectaroundTest result
```
##### 同一class内进行SpringAOP
在工作遇到一个问题，类本身调用本身的方法，需要对同一类方法进行切入。乍一看这是不能办到的
```java
public ApiMessage<String> postData(Document document)
{
    mainProcess(document, transaction);
    return null;
}
@ProcessExceptionAnnotation(status = InboundMonitorLogStatusCodeConstant.PROCESS_ERROR_STATUS_CODE)
public void mainProcess(Document document, InboundTransaction transaction)
{
}
```
上类中postData（）调用mainProcess（）方法，AOP无法织入，因为默认mainProcess(document, transaction)；相当于this.mainProcess(document, transaction);不会去调用代理类的增强方法。
解决方法：
自身@Autowired自身
```java
public class TransactionDataServiceImpl implements TransactionDataService
{
  @Autowired
  private TransactionDataServiceImpl proxySelf;
}
```
调用方法时，强制使用代理类方法
```java
public ApiMessage<String> postData(Document document)
{
    proxySelf.mainProcess(document, transaction);
    return null;
}
@ProcessExceptionAnnotation(status = InboundMonitorLogStatusCodeConstant.PROCESS_ERROR_STATUS_CODE)
public void mainProcess(Document document, InboundTransaction transaction)
{
}
```
