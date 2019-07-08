---
title: EffectiveJava(6)--异常
date: 2019-6-22 12:53:54
tags:
 -EffectiveJava
categories: Basic
thumbnail: /gallery/lol/1557842779295.jpg
---

# EffectiveJava(6)--异常

--

## Overview

1. 异常使用大纲
2. 优先使用标准的异常
3. 抛出与抽象对应的异常
4. 使用切面捕捉异常
5. 业务异常与业务异常提速

---

## 1. 异常使用大纲

Java提供了三种可抛出结构（throwable）：受检异常(exception),运行时异常run-time exception)和错误(error)。如果期望调用者能够适当地恢复， 对于这种情况就应该使用受检异常。通过抛出受检的异常强迫调用者在一个catch子句中处理该异常，或者将它传播出去。因此，方法中声明要抛出的每个受检异常，都是对API用户的一种潜在指示：与异常相关联的条件是调用这个方法的一种可能的结果。

运行时异常和错误。在行为上两者是等同的：它们都是不需要也不应该被捕获的可抛出结构。如果程序抛出未受检的异常或者错误，往往就属于不可恢复的情形，继续执行下去有害无益。如果程序没有捕捉到这样的可抛出结构，将会导致当前线程中断，并出现适当的错误消息。

用运行时异常来表明编程错误。大多数的运行时异常都表示前提违例。所谓前提违例是指API 的客户没有遵守API 规范建立的约定。例如，数组访问的约定指明了数组的下标值必须在零和数组长度减l 之间。ArraylndexOutOfBoundsException表明违反了这个前提。

错误往往被JVN保留下来使用，以表明资源不足、约束失败，或者其他使程序无法继续执行的条件。

你实现的所有未受检的抛出结构都应该是RuntimeException的子类。不仅不应该定义Error子类，甚至也不应该抛出AssertionError异常

总而言之，对于可恢复的情况，要抛出受检异常；对于程序错误，要抛出运行时异常。不确定是否可恢复，则抛出未受检异常。不要定义任何既不是受检异常也不是运行时异常的抛出类型。要在受检异常上提供方法，以便协助恢复。


-----

### 2. 优先使用标准的异常

代码重用是值得提倡的，这是一条通用的规则，异常也不例外。Java提供了一组基本的未受检异常，它们满足了绝大多数API的异常抛出需求。重用标准的异常有多个好处。其中最主要的好处：
1. 它使API更易于学习和使用，因为它与程序员已经熟悉的习惯用法一致。
2. 对于用到这些API的程序而言，它们的可读性会更好，因为它们不会出现很多程序员不熟悉的异常。
3. 异常类越少，意味着内存占用就越小，装载这些类的时间开销也越少(最不重要的)

下面是一些标准异常(加粗的是常用的)：
1. ArithmeticException - 算术运算异常，如除数为0
2. ArrayIndexOutOfBoundsException - 数组索引越界异常。当对数组的索引值为负数或大于等于数组大小时抛出。
3. ArrayStoreException - 向数组中存放与声明类型不兼容对象异常 
4. **ClassCastException** - 类型强制转换异常。
5. ClassNotFoundException - 找不到类异常。当应用试图通过反射构造类，而在遍历CLASSPAH之后找不到对应名称的class文件时，抛出该异常。
6. CloneNotSupportedException - 不支持克隆异常。当没有实现Cloneable接口或者不支持克隆方法时,调用其clone()方法则抛出该异常。
7. IllegalAccessException - 违法的访问异常。当应用试图通过反射方式创建某个类的实例、访问该类属性、调用该类方法，而当时又无法访问类的、属性的、方法的或构造方法的定义时抛出该异常。
8. **IllegalArgumentException** - 传递非法参数异常
9. IllegalMonitorStateException - 违法的监控状态异常。当某个线程试图等待一个自己并不拥有的对象（O）的监控器或者通知其他线程等待该对象（O）的监控器时，抛出该异常。
10. IllegalStateException - 违法的状态异常。当在Java环境和应用尚未处于某个方法的合法调用状态，而调用了该方法时，抛出该异常
11. IllegalThreadStateException - 违法的线程状态异常。当县城尚未处于某个方法的合法调用状态，而调用了该方法时，抛出异常。
12. **IndexOutOfBoundsException** - 索引越界异常。当访问某个序列的索引值小于0或大于等于序列大小时，抛出该异常。
13. InstantiationException - 当试图通过newInstance()方法创建某个类的实例，而该类是一个抽象类或接口时，抛出该异常。
14. **InterruptedException** - 当某个线程处于长时间的等待、休眠或其他暂停状态，而此时其他的线程通过Thread的interrupt方法终止该线程时抛出该异常。
15. NegativeArraySizeException - 数组大小为负值异常。当使用负数大小值创建数组时抛出该异常。

16. NoSuchFieldException - 属性不存在异常。当访问某个类的不存在的属性时抛出该异常。

17. NoSuchMethodException - 方法不存在异常。当访问某个类的不存在的方法时抛出该异常。

18. **NullPointerException** - 空指针异常。当应用试图在要求使用对象的地方使用了null时，抛出该异常。譬如：调用null对象的实例方法、访问null对象的属性、计算null对象的长度、使用throw语句抛出null等等。

19. NumberFormatException - 数字格式异常。当试图将一个String转换为指定的数字类型，而该字符串确不满足数字类型要求的格式时，抛出该异常。

20. **SecurityException** - 安全异常
21. StringIndexOutOfBoundsException - 字符串索引越界异常。当使用索引值访问某个字符串中的字符，而该索引值小于0或大于等于序列大小时，抛出该异常。
22. TypeNotPresentException - 类型不存在异常。当应用试图以某个类型名称的字符串表达方式访问该类型，但是根据给定的名称又找不到该类型是抛出该异常。该异常与ClassNotFoundException的区别在于该异常是unchecked（不被检查）异常，而ClassNotFoundException是checked（被检查）异常。
23. **UnsupportedOperationException** - 不支持的方法异常。指明请求的方法不被支持情况的异常。 


-----

### 3. 抛出与抽象对应的异常

如果方法抛出的异常与它所执行的任务没有明显的联系，这种情形将会使人不知所措。当方法传递由低层抽象抛出的异常时，往往会发生这种情况。除了使人感到困惑之外，这也“污染”.具有实现细节的更高层的API。如果高层的实现在后续的发行版本中发生了变化，它所抛出的异常也可能会跟着发生变化，从而潜在地破坏现有的客户端程序。

**更高层的实现应该捕获低层的异常，同时抛出可以按照高层抽象进行解释的异常。这种做法称为异常转译**：

```java
        try{
            //throw 底层错误
        }catch (LowerLevelException e){
            throw  new HighLevelException(e);
        }
```

异常链对于调试低级异常导致高层异常的问题有非常大的帮助,大多数标准的异常都有支持链的构造器:
```java
    public RuntimeException(Throwable cause) {
        super(cause);
    }
```

异常链对高层和低层异常都提供了最佳的功能：它允许抛出适当的高层异常，同时又能捕获低层的原因进行失败分析。


-----

### 4. 使用切面捕捉异常

1. 手写切面

使用一个callable来捕捉异常

```java
public class ManualExceptionAdvice {

    public Object handle(Callable<Object> task) {
        try {
            return task.call();
        } catch (Exception e) {
            //handle Exception
        }
    }

    private Object method(String args1) {
        System.out.println(args1);
        return null;
    }

    private Object test() {
        return handle(() -> method("1"));
    }
}

```


2. Spring AOP捕捉异常

使用切面捕捉异常很简单，继承ThrowsAdvice即可，甚至你可以通过重载处理不同的Exception.

```java
public class SpringExceptionAdvice implements ThrowsAdvice {
    public void afterThrowing(Method m, Object[] args, Object target, Exception ex) {
        // Do something with all arguments
    }
}

//可以重载
public class MutilTypeExceptionAdvice implements ThrowsAdvice {
    public void afterThrowing(AException ex) throws Throwable {
        // Do something with AException
    }
    public void afterThrowing(BException ex) throws Throwable {
        // Do something with AException
    }
    public void afterThrowing(CException ex) throws Throwable {
        // Do something with AException
    }
    public void afterThrowing(DException ex) throws Throwable {
        // Do something with AException
    }
}

//或者使用Aroud来处理方法，捕捉它的Exception
public class AroudAdvice implements MethodInterceptor {

    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        try {
            return invocation.proceed();
        } catch (Throwable e) {
            //handle exception
        }
    }
}


```



### 5. 业务异常与业务异常提速

业务中，如果代码层次比较多，如果出现中断，需要一层一层的校验。下面是一个例子

```java
    private void test(String paramA, Long paramB, Integer paramC) {
        if (StringUtils.isEmpty(paramA)) {
            throw new RuntimeException();
        }
        if (checkB(paramB)) {
            throw new RuntimeException();
        }
        if (paramC == null) {
            throw new RuntimeException();
        }

        //真正的业务代码
    }
    
    private boolean checkB(Long paramB){
        if(condition){
            return Boolean.TRUE;
        }
        return Boolean.FALSE;
    }
    
```

过多的if校验导致真正的业务方法很长，判断也比较冗余。可以尝试使用业务异常来中断。
下面是自己代码创建一个评论的例子

```java
    public long createReply(long momentId, long comment, Long fromUid, Long toUid, String content) {
        try{
            //速率限制
            rateLimitChecker(fromUid);
            //惩罚限制
            punishChecker(fromUid);
            //敏感词限制
            sensitiveWordChecker(content, fromUid);
            //评论是否存在
            commentValidChecker(comment, momentId, fromUid);

            //真正的创建逻辑
        }catch(BizException ex){
            //handle 异常
        }
    }

    /**
     * 频次校验，通过异常来终止
     */
    private void rateLimitChecker(Long uid) {
        if (rateLimiter.acquireForComment(uid)) {
            return;
        }
        throw new BizException("xxx");
    }
    //其余校验类似
```

当然，使用BizException业务异常来控制业务逻辑有一定的性能损失。但是我们可以通过优化业务异常来减少这种损失。

我们知道，异常性能损耗很大部分是因为打印和获取堆栈所需要的时间，所以我们可以通过让业务异常不打印堆栈的方式来提升性能。使用全参数构造器构造业务Exception。

```java
    protected Exception(String message, Throwable cause,
                        boolean enableSuppression,
                        boolean writableStackTrace) {
        super(message, cause, enableSuppression, writableStackTrace);
    }
    protected RuntimeException(String message, Throwable cause,
                               boolean enableSuppression,
                               boolean writableStackTrace) {
        super(message, cause, enableSuppression, writableStackTrace);
    }
```

message：异常的描述信息，也就是在打印栈追踪信息时异常类名后面紧跟着的描述字符串
cause：导致此异常发生的父异常，即追踪信息里的caused by
enableSuppress：关于异常挂起的参数，这里我们永远设为false即可
writableStackTrace：表示是否生成栈追踪信息，只要将此参数设为false, 则在构造异常对象时就不会调用fillInStackTrace。

所以可以这样定义BizException
```java
public class BizException extends RuntimeException {
    public BizException(String message) {
        super(message, null, false, false);
    }

    public BizException(String message, Throwable cause) {
        super(message, cause, false, false);
    }

    //ErrorCode可以是一个异常枚举，让异常消息复用和集中
    public BizException(ErrorCode code) {
        super(code.message(), null, false, false);
    }
}

```

下面测试代码：

```java
        long start = System.nanoTime();
        for (int i = 0; i < 10000; i++) {
            try {
                System.out.println(test.method1(""));
            } catch (Exception e) {
                System.out.println(e.getMessage());
            }
        }
        System.out.println("cost:" + (System.nanoTime() - start));
```
BizException:
不打印：12356900
打印message：95827300
打印堆栈：127270100


RuntimeException
不打印：27327100
打印message：134160000
打印堆栈：1534584600

可以看出，什么都不打印时，性能提升一倍~两倍。当然，打印堆栈耗时是占大多数时间的。