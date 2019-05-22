---
title: EffectiveJava(1)--对象
date: 2019-5-22 23:12:54
tags:
 -EffectiveJava
categories: Basic
thumbnail: /gallery/lol/1557842698798.jpg
---

# EffectiveJava(1)--对象


## Overview
1. 静态工厂方法
2. Builder模式
3. 单例模式
4. 终结方法


--------

## 静态工厂方法

静态工厂方法的好处：

1. **优势1**:静态工厂方法有名称。举个GUAVA的例子Lists类的例子。生成一个带容量的List。
```java
  public static <E> ArrayList<E> newArrayListWithCapacity(int initialArraySize) {
    checkNonnegative(initialArraySize, "initialArraySize"); // for GWT.
    return new ArrayList<>(initialArraySize);
  }
```
虽然这个例子在java7以后不是必要的，因为使用new ArrayList<>(int)更好。这里主要是想说，这个静态工厂方法的名称可以很清晰的表达我们创建了一个带容量的ArrayList。

2. **优势2**:静态工厂方法不必在每次调用它们的时候都创建一个新对象。举个Boolean的例子
```java
    public static Boolean valueOf(String s) {
        return parseBoolean(s) ? TRUE : FALSE;
    }
```
对象每次返回持有的两个Boolean对象之一。为重复的调用返回相同对象，这种方法类似于享元(Flyweight)模式。这样有助于类总能严格控制在某个时刻哪些实例应该存在。这种类被称作实例受控的类（ instance-controlled ）。单例模式也是借助这种特性实现的。

3. **优势3**:静态工厂方法它可以返回原返回类型的任何子类型的对象。

4. **优势4**:静态工厂方法返回的对象的类可以随着每次调用而发生变化，这取决于静态工厂方法的参数值。

优势3和4可以合并到一起讲，静态方法返回接口类型，实现类可以对外界屏蔽，根据参数等不同可以返回不同的实现。EnumSet是一个很好的例子(感兴趣可以查看一下EnumSet的两种实现，真的很骚)。
```java
    public static <E extends Enum<E>> EnumSet<E> noneOf(Class<E> elementType) {
        Enum<?>[] universe = getUniverse(elementType);
        if (universe == null)
            throw new ClassCastException(elementType + " not an enum");

        if (universe.length <= 64)
            return new RegularEnumSet<>(elementType, universe);
        else
            return new JumboEnumSet<>(elementType, universe);
    }

```
如果枚举类的项小于等于64，使用RegularEnumSet实现(elements只需要一个long实现就OK).如果枚举项大于64，使用JumboEnumSet实现(elements使用long数组实现)
外界不需要知道EnumSet是使用RegularEnumSet还是JumboEnumSet实现。只需要使用即可。

5. **优势5**:静态工厂的方法返回的对象所属的类，在编写包含该静态工厂方
法的类时可以不存在。
这里有个很注明的例子。JDBC Driver的静态工厂方法通过反射创建实现对象。

6. **优势6**:通过静态工厂方法可以实现参数类型相同但是却不同的对象。考虑下面代码
```java
public class A{
    private int a;
    private int b;

    public A(int i){
        this.a = i;
    }
    public A(int i){
        this.b = i;
    }
}
```
如果A这个class只需要a或者b就可以初始化，但是构造函数重载却无法实现。静态工厂方法可以帮我解决这个问题。

```java
    public static A initWithA(int i){
        A res = new A();
        res.setA(i);
        return res;
    }
    public static A initWithB(int i){
        A res = new A();
        res.setB(i);
        return res;
    }
```

-----

## Builder模式
　
### 重载构造函数

静态工厂和构造器有个共同的局限性：它们都不能很好地扩展到大量的可选参数。对于这种场景，我们一向习惯采用重载构造函数（ telescoping cons tructor ）模式
比如Excuption类

```java
    public Exception() {
        super();
    }
    public Exception(String message) {
        super(message);
    }
    public Exception(String message, Throwable cause) {
        super(message, cause);
    }
```

重载构造函数模式下，提供的第一个构造器只有必要的参数，第二个构造器有一个可选参数，第三个构造器有两个可选参数，依此类推，最后一个构造器包含所有可选的参数。

### JavaBeans模式

但是遇到，4个必须参数，4个可选参数。重载构造函数的方式，但是当有许多参数的时候,客户端代码会很难缩写，并且仍然较难以阅读。这种类，通过什么方式构建比较好呢？有些同学想到JavaBeans模式。遗憾的是，JavaBeans模式自身有着很严重的缺点。因为构造过程被分到了几个调用中，在构造过程中JavaBean可能处于不一致的状态，同时JavaBeans模式使得把类做成不可变的可能性不复存在。


### Builder模式

幸运的是，还有第三种替代方法，它既能保证像重叠构造器模式那样的安全性，也能
保证像JavaBeans模式那么好的可读性  ---Builder模式

```java
public class BuilderDemo {
    /**
     * 必须参数
     */
    private final String requiredParam1;
    private final String requiredParam2;
    /**
     * 可选参数
     */
    private final String optionalParam1;
    private final String optionalParam2;


    private BuilderDemo(Builder builder) {
        this.requiredParam1 = builder.requiredParam1;
        this.requiredParam2 = builder.requiredParam2;
        this.optionalParam1 = builder.optionalParam1;
        this.optionalParam2 = builder.optionalParam2;
    }

    public static class Builder {
        private String requiredParam1;
        private String requiredParam2;
        private String optionalParam1;
        private String optionalParam2;

        public Builder(String requiredParam1, String requiredParam2) {
            this.requiredParam1 = requiredParam1;
            this.requiredParam2 = requiredParam2;
        }

        public Builder setOptionalParam1(String optionalParam1) {
            this.optionalParam1 = optionalParam1;
            return this;
        }

        public Builder setOptionalParam2(String optionalParam2) {
            this.optionalParam2 = optionalParam2;
            return this;
        }

        public BuilderDemo build() {
            return new BuilderDemo(this);
        }
    }

    public static void main(String[] args) {
        BuilderDemo.Builder builder = new Builder("1", "2");

        BuilderDemo demo = builder.setOptionalParam1("optional1")
                .setOptionalParam2("optional2")
                .build();

    }
}
```

但是Builder模式会有很大的代码冗余。不过不需要担心，工作中可以使用Lombook这个代码补齐神器。

```java
@Builder
@AllArgsConstructor(access = AccessLevel.PRIVATE)
public class BuilderDemo {
    /**
     * 必须参数
     */
    private final String requiredParam1;
    private final String requiredParam2;
    /**
     * 可选参数
     */
    private final String optionalParam1;
    private final String optionalParam2;
}
```
一个两个注解就可以搞定。没有用过的同学可以了解一下。

-----------

## 单例模式Singleton

### 最优美的单例模式

枚举单例模式最简洁，无偿地提供了序列化机制，绝对防止多次实例化，即使是在面对复杂的序列化或者反射攻击的时候。虽然这种方法还没有广泛采用，但是单元素的枚举类型经常成为实现Singleton的最佳方法。

```java
public enum EnumSingleton {
    INSTANCE;

    public static EnumSingleton getInstance() {
        return INSTANCE;
    }
}

```

### Ojbk的单例模式

比较OK的单例模，需要注意反射攻击和序列化攻击（没人闲的蛋疼）

```java
public class OjbkSingleton implements Serializable {
    private static final OjbkSingleton INSTANCE = new OjbkSingleton();

    private OjbkSingleton() {
    }

    public static OjbkSingleton getInstance() {
        return INSTANCE;
    }

    //防止反序列化假冒单例
    private Object readResolve() {
        return INSTANCE;
    }
}

```

### 懒加载的单例模式

通过静态内部内类的方式实现懒加载和单利模式，需要懒加载的时候可以使用。

```java
public class LazySingleton {

    private static class SingletonHolder {
        private final static LazySingleton INSTANCE = new LazySingleton();
    }

    public static LazySingleton getInstance() {
        return SingletonHolder.INSTANCE;
    }
}

```

### 双重检查吃力不讨单例模式

双重检查在1.5之前，是不安全的。主要是volatile关键字在1.5之前不能保证不会指令重排。所以也是不安全的。1.5之后，volatile保证了变量可见性的同时，也保证了指令不会重排。

这里的volatile不能去掉。如果去掉，可能导致问题：INSTANCE = new DoubleCheckSingleton();并不是一个原子操作。
1. 给INSTANCE分配内存
2. 调用INSTANCE的构造函数来初始化成员变量，形成实例
3. 将INSTANCE对象指向分配的内存空间（执行完这步INSTANCE就非null了）
上面3个指令，在指令重排后可能出现1,3,2的顺序执行。所以其他线程拿到非null但是没有调用构造函数的INSTANCE就会出问题。


```java
public class DoubleCheckSingleton {
    private static volatile DoubleCheckSingleton INSTANCE;

    public static DoubleCheckSingleton getInstance() {
        if (INSTANCE == null) {
            synchronized (DoubleCheckSingleton.class) {
                if (INSTANCE == null) {
                    INSTANCE = new DoubleCheckSingleton();
                }
            }
        }
        return INSTANCE;
    }
}

```

------

## 终结方法
终结方法（finalize）通常是不可预测的，也是很危险的，一般情况下是不必要的。使用终结方法会导致行为不稳定、性能降低，以及可移植性问题。

### 缺点
1. 终结方法和清除方法的缺点在于不能保证会被及时执行［ JLS . 12.6 ］。所以注重时间（ time-c「itical l 的任务不应该由终结方法或者清除方法来完成。
2. 终结方法线程的优先级比该应用程序的其他线程的优先级要低得多，因此不能确保及时运行所有的finalize方法。
3. Java 语言规范不仅不保证终结方法或者清除方法会被及时地执行，而且根本就不保证它们会被执行。当一个程序终止的时候，某些已经无法访问的对象上的终结方法却根本没有被执行，这是完全有可能的。
4. 使用终结方法的另一个问题是：如果忽略在终结过程中被抛出来的未被捕获的异常，该对象的终结过程也会终止［ JLS, 12 . 6 ］。正常情况下，未被捕获的异常将会使线程终止，并打印出战轨迹（ Stack Trace ），但是，如果异常发生在终结方法之中，则不会如此，甚至连警告都不会打印出来。
5. 使用终结方法和清除方法有一个非常严重的性能损失
6. 终结方法有一个严重的安全问题： 它们为终结方法攻击（ finalizer attack ） 打开了类的大门。

### 作用
1. 当资源的所有者忘记调用它的close 方法时，终结方法或者清除方法可以充当"安全网"。
2. 终止非关键的本地资源

**既然不会用，那就尽量不要用finalize这个方法。**

-------





