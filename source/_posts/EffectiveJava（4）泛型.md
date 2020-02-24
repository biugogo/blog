---
title: EffectiveJava(4)--泛型
date: 2019-6-2 16:17:54
tags:
 -EffectiveJava
categories: Basic
thumbnail: /gallery/lol/1557842817807.jpg
---

# EffectiveJava(4)--泛型

## Overview
1. 尽量不使用原生态类型
2. List<?>优先于数组
3. 泛型通配符
4. 类型安全的异构容器
5. asSubclass

-----

### 1.尽量不使用原生态类型

#### 泛型安全性
相比原生态类型, 泛型的优势是在安全性和描述性方面。
```java
private final List stamps;

private final List<Stamp> stamps;

for (Iterator i = stamps.iterator();i.hasNext();){
    Stamp s = (Stamp) i.next(); //抛出异常ClassCastException
}
```
使用原生态类型, 我们可以把非Stamp的对象放入stamps, 并且编译时不会报错, 只有在执行遍历时, 才发现stamps里存在一个不是Stamp的对象, 抛出ClassCastException。

出错之后应该尽快发现, 最好是编译时就发现。在本例中, 直到运行时才发现错误, 已经出错很久了, 而且它在代码中所处的位置, 距离包含错误的这部分代码已经很远了。－旦发现ClassCastException, 就必须搜索代码, 查找将非Stamp放进stamps集合的方法调用。

#### 泛型通配符

在不确定或者不在乎集合中的元素类型的情况下, 你也许会使用原生态类型。
```java
private void method(List list){} ×
```
这个方法可行, 但它使用了原生态类型, 这是很危险的。安全的替代做法是使用无限制的通配符类型。
```java
private void method(List<?> list){} √
```
要使用泛型, 但不确定或者不关心实际的类型参数, 就可以用一个问号代替。我们可以将任何元素放进使用原生态类型的集合中, 因此很容易破坏该集合的类型约束条件
, 但不能将任何元素（除了null 之外）放到collection<?>中。

#### 使用原生态类型列外情况

1. 必须在类文字中使用原生态类型。规范不允许使用参数化类型：List.class,tring[].class和int.class都合法,但是List<String>.class是不合法的
2. instanceof操作的时候
    ```java
        if (o instanceof List) {
            List<?> list1 = (List<?>) o;
        }
    ```

总而言之, 使用原生态类型会在运行时导致异常, 因此不要使用.原生态类型只是为了与引人泛型之前的遗留代码进行兼容和互用而提供的。


------


### 2.List<?>优先于数组

#### 协变和可变
数组与泛型相比, 有两个重要的不同点。首先, 数组是协变的。如果Sub为Super的子类型, 那么Sub[]就是Super[]的子类型。泛型则是可变的。List<Sub>不是List<Super>的子类。你可能认为, 这意味着泛型是有缺陷的, 但实际上可以说数组才是有缺陷的。

```java
    Object[] array = new Long[10];
    array[0] = "string"; //编译时不会报错, 运行时ArrayStoreException

    List<Integer> list = new ArrayList<>();
    list.add("string"); //编译时报错
```
这其中无论哪一种方法, 都不能将String 放进Long 容器中, 但是利用数组, 你会在运行时才发现所犯的错误；而利用列表, 则可以在编译时就发现错误。我们当然希望在编译时就发现错误。

#### 数组具体化和泛型类型擦除

数组是具体化的, 因此数组会在运行时知道和强化它们的元素类型, 如果企图将String保存到Long数组中, 就会得到一个ArrayStoreException异常.泛型则是通过类型擦除实现的, 只在编译时强化它们的类型信息, 并在运行时擦除它们的元素类型信息。擦除使泛型可以与没有使用泛型的代码随意进行互用, 以确保在Java 5 中平滑过渡到泛型。

![](/gallery/effectiveJava/泛型数组不合法.png)

一般来说, 数据和泛型是不混用的。
```java
    public static class Chooser<T> {
        private final T[] array; //泛型数组混用

        public Chooser(List<T> array) {
            this.array = (T[]) array.toArray(); //需要强转并且产生警告
        }
    }
    public static class Chooser<T> {
        private final List<T> array;//全泛型

        public Chooser(List<T> array) {
            this.array = new ArrayList<>(array);//无报错
        }
    }

```
如果你发现自己将它们混合起来使用, 并且得到了编译时错误或者警告, 你的第一反应就应该是用列表代替数组。


总而言之, 数组和泛型有着截然不同的类型规则。数组是协变且可以具体化的；泛型
是不可变的且可以被擦除的.总而言之, 数组和泛型有着截然不同的类型规则。数组是协变且可以具体化的；泛型是不可变的且可以被擦除的。

------

### 3.泛型通配符

**PECS: producer extends ,consumer super**

如果参数化类型表示一个生产者T , 就使用<? extends T>；如果它表示一个消费者T,就使用<? super T>。

#### extends
```java
    public static void main(String[] args) {
        List<Integer> integers = Arrays.asList(1, 2, 3);
        supplier(integers);

        List<Long> longs = Arrays.asList(1L, 2L, 3L);
        supplier(longs)
    }

    private static void supplier(List<? extends Number> list) {
        Number number = list.get(0);
        System.out.println(number);
    }   
```
list生产number对象, list是一个生产者。list存放的都是Number子类, 所以从中得到一个对象, 对象一定是Number的子类,可以用Number为引用。

#### super

```java
    public static void main(String[] args) {
        List<Number> list = new ArrayList<>();
        consumer(list);
        System.out.println(list);

        List<Serializable> slist = new ArrayList<>();
        consumer(slist);
        System.out.println(slist);
    }
    private static void consumer(List<? super Number> list) {
        Integer int_1 = 4;
        Long long_5 = 5L;

        list.add(int_1);
        list.add(long_5);

        list
    }
```
list消费所有Number子类对象,list是一个消费者。list可以是List<Number>或者List<Serializable>, 所以向其中add Integer或者Long没什么错误的地方。

#### 不要用通配符类型作为返回类型

不要用通配符类型作为返回类型,除了为用户提供额外的灵活性之外, 它还会强制用户在客户端代码中使用通配符类型。

```java
    private static List<? extends Number> getList() {
        return new ArrayList<>();
    }

    public static void main(String[] args) {
        //只能用通配符或者擦除掉
        List<? extends Number> list1 = getList();
    }
```

如果使用得当, 通配符类型对于类的用户来说几乎是无形的。它们使方法能够接受它
们应该接受的参数, 并拒绝那些应该拒绝的参数。如果类的用户必须考虑通配符类型, 类的API或许就会出错。

#### 泛型事件驱动模式

试想一下, 当我们定义了不同event, 然后有不同的event handler, 每种event都对应一种handler。我们如何去编写这样一个listener？

1. 定义一个事件接口
    ```java
    public interface Event {
    }
    ```
2. 定义一个添加事件
    ```java
    public class AddEvent implements Event {}
    ```
3. 定义泛型事件处理接口
    ```java
    public interface EventHandler<T extends Event> {
        void doHandle(T t);
    }
    ```
4. 定义添加事件处理程序
    ```java
    public class AddEventHandler implements EventHandler<AddEvent> {
        @Override
        public void doHandle(AddEvent event) {
            System.out.println("AddEventHandler handle event");
        }
    }
    ```
5. Listener
    ```java
    public class Listener {
        private Map<Class<?>, EventHandler<? extends Event>> handlerMap = new HashMap<>();

        public Listener() {
            //初始化事件处理Map
            handlerMap.put(AddEvent.class, new AddEventHandler());
        }

        //监听添加事件
        private void listenAdd(AddEvent event) {
            EventHandler<? extends Event> handler = handlerMap.get(event.getClass());
            helper(handler, event);
        }

        //一个私有的辅助方法来捕捉通配符类型
        @SuppressWarnings("unchecked")
        private static <E extends Event> void helper(EventHandler<E> handler, Event event) {
            handler.doHandle((E) event);
        }
    }
    ```

上面的代码中, helper()方法是非常有用的, 它是一个**辅助方法来捕捉通配符类型**。因为我们EventHandler是泛型<? extends Event>的, 不经过转换是无法doHandle任何对象。我们通过helper捕捉到 ? 通配符类型为 E 。然后把Event强转为捕捉到的 E 类型。这样就可以不通过类型擦除的方式使用EventHandler。


-----

### 4.类型安全的异构容器


```java
import java.util.HashMap;
import java.util.Map;

public class Favorite {

    Map<Class<?>, Object> context = new HashMap<>();

    public <T> void putFavorite(Class<T> clazz, T instance) {
        context.put(clazz, clazz.cast(instance));
    }

    public <T> T getInstance(Class<T> clazz) {
        return clazz.cast(context.get(clazz));
    }

}
```
Favorite实例是类型安全的:当你向它请求String的时候,它从来不会返回一个Integer给你。同时它也是异构的:不像普通的映射,它的所有键都是不同类型的。因此, 我们将Favorite称作类型安全的异构容器.

每个Favorites 实例都得到一个称作favorites 的私有Map<Class<?>, Object>的支持。你可能认为由于无限制通配符类型的关系, 将不能把任何东西放进这个Map 中, 但事实正好相反。耍注意的是通配符类型是嵌套的： 它不是属于通配符类型的Map 的类型, 而是它的键的类型。由此可见, 每个键都可以有一个不同的参数化类型：一个可以是Class<String>, 接下来是Class<Integer>等。异构就是从这里来的。

getInstance方法使用了Class的cast方法, 将对象引用动态地转换(dynamically cast)成了Class对象所表示自由类型。cast方法是Java的转换操作符的动态模拟。它只检验它的参数是否为Class对象所表示的类型的实例。如果是, 就返回参数;否则就抛出ClassCastException异常。

### 5.asSubclass

```java
    public <T extends Event> T getInstance(Class<T> clazz) {
        return clazz.cast(context.get(clazz));
    }
    public <T extends Event> void putFavorite(Class<T> clazz, T instance) {
        context.put(clazz, clazz.cast(instance));
    }

    public static void main(String[] args) throws ClassNotFoundException {
        Favorite f = new Favorite();
        f.putFavorite(AddEvent.class, new AddEvent());


        Class<?> name = Class.forName("com.apollo.demo.generic.AddEvent");

        Event instance = f.getInstance(name.asSubclass(Event.class));
    }
```

假设你有一个类型为Class<?>的对象, 并且想将它传给一个需要有限制的类型令牌的方法, 例如getInstance()。你可以将对象转换成<T extends Event>,但是这种转换是非受检的, 因此会产生一条编译时警告。幸运的是,Class提供了一个安全（且动态）地执行这种转换的实例方法。该方法称作asSubclass,,  它将调用它的Class对象转换成用其参数表示的类的一个子类。如果转换成功,该方法返回它的参数；如果失败, 则抛出ClassCastException异常。

-----

### 6.获取泛型类型

泛型是编译时具有的，当运行期间，会被擦除至上限，当没有通配符上限时，会被擦除至Object。

```java
        List<Integer> list = new ArrayList<>();
        ParameterizedType parameterizedType = (ParameterizedType) list.getClass().getGenericSuperclass();
        System.out.println(parameterizedType);
```
当使用反射获取list的类型时，只能获取到java.util.ArrayList<E> 获取不到具体类型。但是当我们想要获取类型时，怎么处理呢？

答案是使用父类固化的方式

```java
        List<Integer> list = new ArrayList<Integer>(){};
        ParameterizedType parameterizedType = (ParameterizedType) list.getClass().getGenericSuperclass();
        System.out.println(parameterizedType);
```

同样的代码，只是多了一对{}，为什么这时候，可以获取到list的type是
java.util.ArrayList<java.lang.Integer> 。这是因为list此时是 ArrayList<Integer>的匿名子类。这段代码相当于

```java
    private static class A extends ArrayList<Integer> {

    }
    public static void main(String[] args){
        ParameterizedType parameterizedType2 = (ParameterizedType) A.class.getGenericSuperclass();
        System.out.println(parameterizedType2);
    }
```

通过A的继承，锁定了他的类型不再是泛型ArrayList<?>，而是一定的ArrayList<Integer>。这样就可以拿到准确类型。这也是 jackson 反序列化json为Object的TypeReference类实现原理。通过建立一个TypeReference的匿名子类，获取泛型类型。

```java
import com.fasterxml.jackson.core.type.TypeReference;

        TypeReference<List<List<Integer>>> type = new TypeReference<List<List<Integer>>>() {};
```


------
