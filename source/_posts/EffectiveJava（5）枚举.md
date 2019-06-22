---
title: EffectiveJava(5)--枚举
date: 2019-6-22 12:53:54
tags:
 -EffectiveJava
categories: Basic
thumbnail: /gallery/lol/1557842817807.jpg
---

# EffectiveJava(5)--枚举

## Overview

1. 枚举代替int常量
2. EnumSet和EnumMap
3. 枚举常用写法和策略模式

----

### 1. 枚举代替int常量

下面是几个常量，代表各种角色的SALRY
```java
    private static final int SALRY_MANAGER = 100_000;
    private static final int SALRY_PROGRAMMER = 1;
    private static final int SALRY_PM = 88_888;

    private static final int WORK_HOUR_MANAGER = 8;
    private static final int WORK_HOUR_PROGRAMMER = 12;
    private static final int WORK_HOUR_PM = 8;
```
这种方法称作int 枚举棋式,int 枚举模式不具有类型安全性， 也几乎没有描述性可言。例如你将任何int值与SALRY比，或者可以SALRY传到WORK_HOUR等变量。
同时很难将int枚举常量转换成可打印的字符串。就算将这种常量打印出来，或者从调试器中将它显示出来，你所见到的也只是一个数字，这几乎没有什么用处。

这种时候，换用枚举是个很好的选择

```java
public enum Position {
    Manager(8, 100_000),
    Programmer(12, 1),
    PM(8, 88_888);

    int workHour;
    int salary;

    Position(int workHour, int salary) {
        this.workHour = workHour;
        this.salary = salary;
    }
}
```
枚举类型保证了编译时的类型安全。例如声明参数的类型为Position，它就能保证传到该参数上的任何非空的对象引用一定属于三个有效的Position值之一，而其他任何试图传递类型错误的值都会导致编译时错误，就像试图将某种枚举类型的表达式赋给另一种枚举类型的变量，或者试图利用==操作符比较不同枚举类型的值都会导致编译时错误。包含同名常量的多个枚举类型可以在一个系统中和平共处，因为每个类型都有自己的命名空间。你可以增加或者重新排列枚举类型中的常量，而无须重新编译它的客户端代码。除了完善int枚举模式的不足之外，枚举类型还允许添加任意的方法和域，并实现任意的接口。


**每当需要一组固定常量．并且在编译时就知道其成员的时候，就应该使用枚举。**

总而言之，与int常量相比，枚举类型的优势是不言而喻的。枚举的可读性更好，也更加安全，功能更加强大。许多枚举都不需要显式的构造器或者成员，但许多其他枚举则受益于属性与每个常量的关联以及其行为受该属性影响的方法。只有极少数的枚举受益于将多种行为与单个方法关联。在这种相对少见的情况下，特定于常量的方法要优先于启用自有值的枚举。如果多个（但非所有）枚举常量同时共享相同的行为，则要考虑策略枚举。


```java
    public enum Operation {
        PLUS, MINUS, TIMES, DIVIDE;

        public double apply(double x, double y) {
            switch (this) {
                case PLUS:
                    return x + y;
                case MINUS:
                    return x - y;
                case TIMES:
                    return x * y;
                case DIVIDE:
                    return x / y;
                default:
                    throw new RuntimeException("can not support");
            }
        }
    }
```
考虑这段代码，根据不同的操作符，选择不同的运算规则。这段代码能用，但是不太好看。如果没有throw语句，它就不能进行编译，虽然从技术角度来看代码的结束部分是可以执行到的，但是实际上是不可能执行到这行代码的。

更糟糕的是，这段代码很脆弱。如果你添加了新的枚举常量，却忘记给switch添加相应的条件，枚举仍然可以编译，但是当你试图运用新的运算时，就会运行失败。


幸运的是，有一种更好的方法可以将不同的行为与每个枚举常量关联起来：在枚举类
型中声明一个抽象的apply方法.这种方法被称作特定于常量的方法实现

```java
    public enum Operation {
        PLUS {
            @Override
            public double apply(double x, double y) {
                return x + y;
            }
        },
        MINUS {
            @Override
            public double apply(double x, double y) {
                return x - y;
            }
        },
        TIMES {
            @Override
            public double apply(double x, double y) {
                return x * y;
            }
        },

        DIVIDE {
            @Override
            public double apply(double x, double y) {
                return x / y;
            }
        };

        public abstract double apply(double x, double y);
    }
}
```
如果给Operation 的第二种版本添加新的常量，你就不可能会忘记提供apply 方
法，因为该方法紧跟在每个常量声明之后。即使你真的忘记了，编译器也会提醒你，因为枚举类型中的抽象方法必须被它的所有常量中的具体方法所覆盖。

---

### 2. EnumSet和EnumMap

所有的枚举都有一个ordinal 方法，它返回每个枚举常量在类型中的数字位置。一般情况下，对普通程序员来说是不会使用ordinal()方法的，根据序数编写的程序虽然可以工作，但是维护起来就像一场噩梦，如果常量进行重新排序，你依赖序数的程序可能保持正常工作了。


EnumSet和EnumMap的实现原理就和ordinal有关

#### EnumSet

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

上文是EnumSet的构造方式一种，可以看到EnumSet的实现方式有两种，RegularEnumSet和JumboEnumSet。枚举类型的枚举数量小于64时，采用RegularEnumSet（实现方式是long），否则使用JumboEnumSet（实现方式long[]）。

这里分析下简单的RegularEnumSet。

```java

    /**
    * add一个枚举，实现是通过把1左移序数位，再跟原数|操作
    * 比如添加第一个序数2的枚举，左移后   100，与0 | 操作后位 elements=100 
    * 比如添加第二个序数4的枚举，左移后 10000，与100 |操作后位 elements=10100
    * elements=10100 代表序数2和序数4添加到了Set中
    */
    public boolean add(E e) {
        typeCheck(e);

        long oldElements = elements;
        elements |= (1L << ((Enum<?>)e).ordinal());
        return elements != oldElements;
    }

    /**
    * contains则先判断是否是这个枚举，不是直接返回false
    * 判断序数2在不在，先将1左移2位变成 100,
    * 再100 与 10100 &操作变为 00100 !=0 ,说明存在，如果是序数位3判断，1000与10100 & 操作00000 = 0 ，说明不存在。
    */
    public boolean contains(Object e) {
        if (e == null)
            return false;
        Class<?> eClass = e.getClass();
        if (eClass != elementType && eClass.getSuperclass() != elementType)
            return false;

        return (elements & (1L << ((Enum<?>)e).ordinal())) != 0;
    }
```

当序数大于64后，long不能装下整个枚举，而是long[]方式。


#### EnumMap

```java
    public EnumMap(Class<K> var1) {
        this.keyType = var1;
        this.keyUniverse = getKeyUniverse(var1);
        this.vals = new Object[this.keyUniverse.length];
    }
```

EnumMap的实现更为简单，直接通过一个Object[]实现，ordinal等于1则 Object[1] = value.

```java
    /**
    * 添加就是 取ordinal作为下标，然后 Object[ordinal] = value。maskNull和unmaskNull则是对null的处理，EnumMap支持null位Key
    * 
    */
    public V put(K key, V value) {
        typeCheck(key);

        int index = key.ordinal();
        Object oldValue = vals[index];
        vals[index] = maskNull(value);
        if (oldValue == null)
            size++;
        return unmaskNull(oldValue);
    }
```


总而言之， 正是因为枚举类型要用在集合中，所以没有理由用位域来表示它。EnumSet类集位域的简洁和性能优势。最好不要用序数来索引数组，而要使用EnumMap。应用程序的程序员在一般情况下都不使用Enurn.ordinal方法。

----

### 3. 枚举常用写法和枚举策略模式

1. 根据id反序列化枚举,valueOf(int id) 

```java
public enum Position {
    Manager(1, 8, 100_000),
    Programmer(2, 12, 1),
    PM(3, 8, 88_888);

    int id;
    int workHour;
    int salary;

    Position(int id, int workHour, int salary) {
        this.id = id;
        this.workHour = workHour;
        this.salary = salary;
    }

    private static Map<Integer, Position> idParse = Collections.emptyMap();

    static {
        ImmutableMap.Builder<Integer, Position> builder = ImmutableMap.builderWithExpectedSize(values().length);
        for (Position position : values()) {
            builder.put(position.id, position);
        }
        idParse = builder.build();
    }

    public static Optional<Position> valueOf(int id) {
        return Optional.ofNullable(idParse.get(id));
    }
}
```
这里利用了Enum的特性，Enum的构造函数是先于static代码块执行的。所以static中进行的操作，是有效的。返回值是Optional包裹的，因为id可能不存在于这个Enum，强制客户端处理null的情况。


2. 枚举策略委托
```java
public enum Position {
    Manager(100_000, Allowance.Others) {
        @Override
        int money() {
            return (salary + allowance.value) * 5;
        }
    },
    Programmer(1, Allowance.Programmer) {
        @Override
        int money() {
            return salary + allowance.value;
        }
    },
    PM(88_888, Allowance.Others) {
        @Override
        int money() {
            return (salary + allowance.value) * 2;
        }
    };

    int salary;
    Allowance allowance;

    Position(int salary, Allowance allowance) {
        this.salary = salary;
        this.allowance = allowance;
    }

    private enum Allowance {
        Programmer(0),
        Others(10_000);
        int value;

        Allowance(int value) {
            this.value = value;
        }
    }

    abstract int money();
}

```

把Allowance津贴策略委托给了Position，再和salary挂钩，算出money。


-----

