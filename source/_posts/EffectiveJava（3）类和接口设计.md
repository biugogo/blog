---
title: EffectiveJava(3)--类和接口设计
date: 2019-6-2 16:17:54
tags:
 -EffectiveJava
categories: Basic
thumbnail: /gallery/lol/1557842705274.jpg
---

# EffectiveJava(3)--类和接口设计

## Overview
1. 类和成员的可访问性
2. 退化数据域类使用原则
3. 复合优先于继承，接口优于抽象类
4. 要么设计继承并提供文档说明，要么禁止继承
5. 拒绝常量接口，接口只用于定义类型
6. 静态成员类优于非静态成员类

-----

### 1.类和成员的可访问性
**尽可能地使每个类或者成员不被外界访问**


区分一个组件设计得好不好，唯一重要的因素在于，它对于外部的其他组件而言，是否隐藏了其内部数据和其他实现细节。设计良好的组件会隐藏所有的实现细节，把API与实现清晰地隔离开来。组件之间只通过API进行通信，一个模块不需要知道其他模块的内部工作情况---封装
1. 封装有效地解除组成系统的各组件之间的藕合关系，即解耦，使得这些组件可以独立地开发、测试、优化、使用、理解和修改。
2. 封装让组件可以并行开发，所以加快了系统开发的速度。同时减轻了维护的负担，程序员可以更快地理解这些组件，并且在调试它们的时候不影响其他的组件。
3. 封装本身无论是对内还是对外都不会带来更好的性能，但是可以让各组件就可以被进一步优化，而不会影响到其他组件的正确性。
4. 封装提高了软件的可重用性，因为组件之间并不紧密相连。
5. 降低了构建大型系统的风险，因为即使整个系统不可用，这些独立的组件仍有可能是可用的。

**封装原则**：尽可能地使每个类或者成员不被外界访问

1. 顶层类：，只有两种可能的访问级别，public和default的
2. 嵌套类：只是在某一个类的内部被用到，就应该考虑使它成为唯一使用它的那个类的私有嵌套类。
3. 实例域：公有类的实例域决不能是公有的：如果实例域是非final 的，或者是一
个指向可变对象的final 引用， 那么一旦使这个域成为公有的，就等于放弃了对存储在这个域中的值进行限制的能力。包含公有可变域的类通常并不是线程安全的。
4. 静态域：除了大写常量构成了类提供的整个抽象中的一部分，可以通过公有的静态final域来暴露这些常量。特别注意的是**长度非零的数组**总是可变的，所以让类具有公有的静态final数组域，或者返回这种域的访问方法，这是错误的。
```java
public static final Thing[] VALUES= { .. . }; //错误用法

//解决方案1: 必可变集合包裹
private static final Thing[] PRIVATE_VALUES = { ... };
public static final List<Thing> VALUES=Collections.unmodifiablelist(Arrays.asList(b))
//解决方案2：返回拷贝对象
private static final Thing[] PRIVATE_VALUES = { ... };
public static Thing[] getValue(){
    return PRIVATE_VALUES.clone();
}
```

-----

### 2.退化数据域类使用原则

```java
class A {
    private String a;
    private String b;
}
```

退化数据域是可以被直接访问的，这些类没有提供封装（Getter和Setter）的功能。
原则：如果类可以在它所在的包之外进行访问，就需要提供Getter和Setter.如果类是包级私有的，或者是私有的嵌套类，直接暴露它的数据域并没有本质的错误.


-----

### 3.复合优先于继承，接口优于抽象类

#### 3.1复合优先于继承
继承是实现代码重用的有力手段，但它并非永远是完成这项工作的最佳工具。对于专门为了继承而设计并且具有很好的文档说明的类来说，使用继承也是非常安全的，但是对普通的具体类进行跨越包边界的继承，则是非常危险的。

以下是一个使用继承出现问题的例子
```java
import java.util.Arrays;
import java.util.Collection;
import java.util.HashSet;

public class MySet<E> extends HashSet<E> {
    private int addCount;

    @Override
    public boolean add(E e) {
        addCount++;
        return super.add(e);
    }

    @Override
    public boolean addAll(Collection<? extends E> c) {
        addCount++;
        return super.addAll(c);
    }

    public int getAddCount() {
        return addCount;
    }

    public static void main(String[] args) {
        MySet<String> mySet = new MySet<>();

        mySet.addAll(Arrays.asList("1"));

        System.out.println(mySet.getAddCount());
    }
}

```
按照我们的设计，我们通过addAll添加一个 "1" 字符串元素到MySet中，addCount输出应该是1，但是很抱歉，这里会输出2。当我们仔细去addAll文档时，我们会发现，HashSet的addAll是基于add方法的，这里添加一个元素，addCount会增加两次。
```java
    //HashSet的addAll方法
    public boolean addAll(Collection<? extends E> c) {
        boolean modified = false;
        for (E e : c)
            if (add(e)) //调用了add实现
                modified = true;
        return modified;
    }
```

所以，当我们没有阅读继承文档，不清楚一个类的内部实现时，使用继承是非常危险的。当我们不想耗费时间去阅读一个类是继承文档时，我们可以选择 “复合”模式去解决
```java
public class MySet<E> {
    private int addCount;
    private Set<E> interSet;

    public MySet() {
        this.interSet = new HashSet<>();
    }

    public boolean add(E e) {
        addCount++;
        return interSet.add(e);
    }

    public boolean addAll(Collection<? extends E> c) {
        addCount++;
        return interSet.addAll(c);
    }

    public int getAddCount() {
        return addCount;
    }

    public static void main(String[] args) {
        MySet<String> mySet = new MySet<>();
        mySet.addAll(Arrays.asList("1"));
        System.out.println(mySet.getAddCount());
    }
}

```
当我们通过复合模式包装了HashSet，达到了继承它的相同的作用，这样得到的类将会非常稳固，它不依赖于现有类的实现细节。即使现有的类添加了新的方法，也不会影响新的类。只有当子类真正是超类的子类型时，才适合用继承。换句话说，对于两个类A和B,只有当两者之间确实存在“ is-a ”关系的时候，类B才应该扩展类A。每个B确实也是A吗？

#### 3.2接口优于抽象类
1. 现有的类可以很容易被更新，以实现新的接口。例如，当Comparable、Iterable和Autocloseable接口被引入Java平台时，更新了许多现有的类，以实现这些接口。
2. 一个类可以继承多个接口
3. 接口允许构造非层次结构的类型框架,使得安全地增强类的功能成为可能。


#### 骨架类，可以把接口和抽象类的优点结合起来
接口负责定义类型，或许还提供一些缺省方法，而骨架实现类则负责实现除基本类型接口方法之外，剩下的非基本类型接口方法。按照惯例，骨架实现类被称为Abstract[*interface*],例如：AbstractCollection 、AbstractSet 、AbstractList和AbstractMap等。设计得当的骨架类可以使程序员非常容易地提供他们自己的接口实现。

下面是AbstractCollection骨架类,当我们继承AbstractCollection去实现自己的collection类时，会变得相对简单。
```java
public abstract class AbstractCollection<E> implements Collection<E> {

    public abstract Iterator<E> iterator();

    public abstract int size();

    public boolean isEmpty() {
        return size() == 0;
    }

    public boolean add(E e) {
        throw new UnsupportedOperationException();
    }

    public boolean contains(Object o) {
        Iterator<E> it = iterator();
        if (o==null) {
            while (it.hasNext())
                if (it.next()==null)
                    return true;
        } else {
            while (it.hasNext())
                if (o.equals(it.next()))
                    return true;
        }
        return false;
    }
    ......
}

```

另一个例子是com.rabbitmq.client.Consumer接口

```java
public interface Consumer {

    void handleConsumeOk(String consumerTag);

    void handleCancelOk(String consumerTag);

    void handleCancel(String consumerTag) throws IOException;

    void handleShutdownSignal(String consumerTag, ShutdownSignalException sig);

    void handleRecoverOk(String consumerTag);

    void handleDelivery(String consumerTag,
                        Envelope envelope,
                        AMQP.BasicProperties properties,
                        byte[] body)
        throws IOException;
}
```
实现了骨架类DefaultConsumer，让我们可以任意覆盖使用到的方法
```java
public class DefaultConsumer implements Consumer {
    /** Channel that this consumer is associated with. */
    private final Channel _channel;
    /** Consumer tag for this consumer. */
    private volatile String _consumerTag;

    public DefaultConsumer(Channel channel) {
        _channel = channel;
    }


    @Override
    public void handleConsumeOk(String consumerTag) {
        this._consumerTag = consumerTag;
    }

    @Override
    public void handleCancelOk(String consumerTag) {
        // no work to do
    }

    @Override
    public void handleCancel(String consumerTag) throws IOException{
        // no work to do
    }

    @Override
    public void handleShutdownSignal(String consumerTag, ShutdownSignalException sig) {
        // no work to do
    }

    @Override
    public void handleRecoverOk(String consumerTag) {
        // no work to do
    }

    @Override
    public void handleDelivery(String consumerTag,
                               Envelope envelope,
                               AMQP.BasicProperties properties,
                               byte[] body)
        throws IOException{
            // no work to do
    }

    public Channel getChannel() {
        return _channel;
    }

    public String getConsumerTag() {
        return _consumerTag;
    }
}

```

----------

### 4.要么设计继承并提供文档说明，要么禁止继承

在本文第2节我们了解到，当我们没有阅读文档就去继承一个类时，会出现很多意想不到的情况。对于不是为了继承而设计并且没有文档说明的“外来”类进行子类化是多么危险。那么对于专门为了继承而设计并且具有良好文档说明的类而言，这又意味
着什么呢？
1. 首先该类必须有文档说明它可覆盖的方法的自用性。
2. 为了继承而进行的设计不仅仅涉及自用模式的文档设计,类必须以精心挑选的受保护的（ protected ）方法的形式，提供适当的钩子（ hook ），以便进入其内部工作中.
3. 对于为了继承而设计的类，唯一的测试方法就是编写子类,必须在发布类之前先编写子类对类进行测试。
4. 构造器决不能调用可被覆盖的方法

-----
### 5. 拒绝常量接口，接口只用于定义类型
很多程序员喜欢用Interface定义常量
```java
public interface Constants {
    final static int INT_1 = 1;
}
```
常量接口模式是对接口的不良使用,如果要导出常量，可以有几种合理的选择方案。
1. 如果这些常量与某个现有的类或者接口紧密相关，就应该把这些常量添加到这个类或者接口中。例如Integer中的
    ```java
    @Native public static final int   MIN_VALUE = 0x80000000;

    @Native public static final int   MAX_VALUE = 0x7fffffff;
    ```
2. 如果这些常量最好被看作枚举类型的成员，就应该用枚举类型。
3. 也可以使用不可实例化的工具类(私有化构造器)
    ```java
    public class Constants {
        public final static int INT_1 = 1;
        private Constants{}
    }
    ```

-----
### 6.静态成员类优于非静态成员类

嵌套类是指定义在另一个类的内部的类。嵌套类存在的目的应该只是为它的外围类提供服务。如果嵌套类将来可能会用于其他的某个环境中，它就应该是顶层类。

如果声明成员类不要求访问外围实例，就要始终把修饰符static放在它的声明中，使它成为静态成员类，而不是非静态成员类。如果省略static修饰符，则每个实例都将包含一个额外的指向外围对象的引用。保存这份引用要消耗时间和空间，并且会导
致外围实例在符合垃圾回收时却仍然得以保留。由此造成的内存泄漏可能是灾难性的。但是常常难以发现，因为这个引用是不可见的。


嵌套类分类：
1. 静态成员类:

    静态成员类是最简单的一种嵌套类。最好把它看作是普通的类，只是碰巧被声明在另一个类的内部而己，它可以访问外围类的所有成员.静态成员类是外围类的一个静态成员，与其他的静态成员一样，也遵守同样的可访问性规则。如果它被声明为私有的，它就只能在外围类的内部才可以被访问，等等。
    
    静态成员类的一种常见用法是作为公有的辅助类，只有与它的外部类一起使用才有意义
    
    从语法上讲，静态成员类和非静态成员类之间唯一的区别是，静态成员类的声明中包含修饰符static。如果嵌套类的实例可以在它外围类的实例之外独立存在，这个嵌套类就必须是静态成员类.


2. 非静态成员类:

    尽管非静态成员类和静态成员类的语法非常相似，但是这两种嵌套类有很大的不同。非静态成员类的每个实例都隐含地与外围类的一个外围实例相关联.在非静态成员类的实例方法内部，可以调用外围实例上的方法，或者利用修饰过的this构造获得外围实例的引用.
    
    在没有外围实例的情况下，要想创建非静态成员类的实例是不可能的。

    当非静态成员类的实例被创建的时候，它和外围实例之间的关联关系也随之被建立起来；而且，这种关联关系以后不能被修改.

    非静态成员类的一种常见用法是定义一个Adapter.它允许外部类的实例被看作是另一个不相关的类的实例。例如ArrayList中的Iterator实现
    ```java
    public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable{
        private class Itr implements Iterator<E> {
            int cursor;       // index of next element to return
            int lastRet = -1; // index of last element returned; -1 if  no such
            int expectedModCount = modCount;

            Itr() {}

            public boolean hasNext() {
                return cursor != size;
            }

            @SuppressWarnings("unchecked")
            public E next() {
                checkForComodification();
                int i = cursor;
                if (i >= size)
                    throw new NoSuchElementException();
                Object[] elementData = ArrayList.this.elementData;
                if (i >= elementData.length)
                    throw new ConcurrentModificationException();
                cursor = i + 1;
                return (E) elementData[lastRet = i];
            }
            ......
        }
        ......
    }
    ```

3. 匿名类：

    匿名类是没有名字的。它不是外围类的一个成员。它并不与其他的成员一起被声明，而是在使用的同时被声明和实例化。匿名类可以出现在代码中任何允许存在表达式的地方。当且仅当匿名类出现在非静态的环境中时，它才有外围实例。
    
    但是即使它们出现在静态的环境中，也不可能拥有任何静态成员，而是拥有常数变量），常数变量是final基本类型，或者被初始化成常量表达式的字符串域。

    匿名类的运用受到诸多的限制。除了在它们被声明的时候之外，是无法将它们实例化的。不能执行instanceof测试或者做任何需要命名类的其他事情

    无法声明一个匿名类来实现多个接口，或者扩展一个类，并同时扩展类和实现接口。

    由于匿名类出现在表达式中，它们必须保持简短，当我们能使用lambda时，优先使用lambda。
4. 局部类：
    局部类是四种嵌套类中使用最少的类。在任何“可以声明局部变量”的地方，都可以声明局部类，并且局部类也遵守同样的作用域规则。
    
    局部类与其他三种嵌套类中的每一种都有一些共同的属性。
    
    与成员类一样，局部类有名字，可以被重复使用。
    
    与匿名类一样，只有当局部类是在非静态环境中定义的时候，才有外围实例，它们也不能包含静态成员，同时它们必须非常简短，以便不会影响可读性。


如果一个嵌套类需要在单个方法之外仍然是可见的，或者它太长了，不适合放在方法内部，就应该使用成员类。

如果成员类的每个实例都需要一个指向其外围实例的引用，就要把成员类做成非静态的；否则，就做成静态的。

如果这个嵌套类属于一个方法的内部，如果你只需要在一个地方创建实例，并且已经有了一个预置的类型可以说明这个类的特征，就要把它做成匿名类；否则，就做成局部类。
