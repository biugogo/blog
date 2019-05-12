---
title: Guava常用写法
date: 2018-5-31 16:14:10
tags:
 -Guava
categories: Guava
thumbnail: https://raw.githubusercontent.com/apollochen123/image/master/%E9%BB%98%E8%AE%A42.jpg
---


# Guava常用写法
----------
## Overview
最近由于项目很多还是使用jdk1.7 没有java8的stream等遍历集合的API。但是有看到老师用了guava中遍历集合的方法，知道了guava这个强大的工具包。发现java8很多东西都是抽象或者风格模仿自guava。这里总结一些常用的写法。在写代码时可以提高效率。不求一次能总结，慢慢的总结吧。

-----
#### List处理
1. 需求：数据库模型Entity转交互Data时，两个结构相似。在java8中我们stream后直接collection就好。但是guava有个类似写法：
```java
List<Data> dataList = Lists.transform(entityList, new Function<Entity, Data>() {
            @Nullable
            @Override
            public Data apply(@Nullable Entity input) {
                 .....
                 Entity ---> Data;
            }
        }).toList();
```
2. 生成一个List， 带有数据
```java
List<Data> list = Lists.newArrayList(data1,data2...);
//Lists.newLinkedList();
```
同时可以初始化100的list
```java
List<Data> list = Lists.newArrayListWithCapacity(100);
```
3. 把一个list拆分子list
```java
//把allList拆分为每个3个成员的子list。最后一个可能小于3个成员
List<List<Long>> list = Lists.partition(allList, 3);
```
4. 返回一个队列倒序队列
```java
List<Data> list = Lists.reverse(originalList);
```
5. 返回一个并发写队列。队列在迭代时不能写入。
```java
List<Data> list = Lists.newCopyOnWriteArrayList();
```
6. 不可变List。
 * 对不可靠的客户代码库来说，它使用安全，可以在未受信任的类库中安全的使用这些对象
 * 线程安全的：immutable对象在多线程下安全，没有竞态条件
 * 不需要支持可变性, 可以尽量节省空间和时间的开销. 所有的不可变集合实现都比可变集合更加有效的利用内存 (analysis)
 * 可以被使用为一个常量，并且期望在未来也是保持不变的  

 创建不可变list:
 ```java
 List<Person> immutableList = ImmutableList.of(new Person("chenhao2", 23),new Person("chenhao2", 24));
List<Person> immutableList1 = ImmutableList.<Person>builder().add(new Person("chenhao2", 23),new Person("chenhao2", 24)).build();
List<Person> immutableList2 = ImmutableList.copyOf(persons);
 ```
------------
#### FluentIterable
流试迭代
```java
package com.yy.apollo.demo;

import java.util.List;

import com.google.common.base.Function;
import com.google.common.base.Predicate;
import com.google.common.collect.FluentIterable;
import com.google.common.collect.Lists;

import lombok.Data;

public class FluentIterableTest {

    public static void main(String[] args) {
       //创建一个队列
        List<Person> persons = Lists.newArrayList(
                new Person("chenhao2", 23),
                new Person("chenhao3", 30),
                new Person("chenhao4", 40),
                new Person("chenhao5", 50),
                new Person("chenhao6", 60),
                new Person("chenhao7", 70),
                new Person("chenhao100", 100));

        FluentIterable.from(
        //从Person-->User 队列
        Lists.transform(persons, new Function<Person, User>() {
            @Override
            public User apply(Person input) {
                return new User(input);
            }
        }))
         //过滤User age>70的User
        .filter(new Predicate<User>() {
            @Override
            public boolean apply(User input) {
                if (input.getAge() > 70) {
                    return false;
                }
                return true;
            }
        })
        //截取前4个
        .limit(4)
        //跳过第一个
        .skip(1)
        //循环遍历剩下的
        .forEach(action -> {
            System.out.println(action);
        });
    }

}

@Data
class Person {
    public Person(String name, int age) {
        super();
        this.name = name;
        this.age = age;
    }

    private String name;
    private int age;
}

@Data
class User {
    public User(String name, int age) {
        super();
        this.name = name;
        this.age = age;
    }

    public User(Person person) {
        this.name = person.getName();
        this.age = person.getAge();
    }

    private String name;
    private int age;
}
```
FluentIterable 提供了很多方法。如：
1. from：从Iterable 生成FluentIterable
2. concat： 拼接
3. append： 添加
4. filter： 过滤器
5. anyMatch： 返回boolean是否有任何匹配到
6. allMatch： 返回boolean是否全部匹配到
7. transform: 先转化再匹配
8. toList: 收集为队列
9. toSet: 收集成set
10. toSortedList 收集为排序队列
11. toSortedSet 收集为排序set
12. toMap 收集为Map
等。

------
#### Map处理
1. 各种Map创建
  ```java
  // 创建hashmap
  Map<String, Object> hashmap = Maps.newHashMap();
  // 创建map固定长度
  Map<String, Object> hashmap2 = Maps.newHashMapWithExpectedSize(10);
  // 创建IdentityHashMap。IdentityHashMap判定相同是key1==key2严格的map
  Map<String, Object> hashmap3 = Maps.newIdentityHashMap();
  // 保持顺序的hashMap，额外维持一个顺序链表。
  Map<String, Object> hashmap4 = Maps.newLinkedHashMap();
  ```
2. Map value值对象定向转换,
   ```java
   // Map中设计的元素value改变。比如从Map<String,Person>改变为Map<String,User>
   Map<String, Person> fromMap = ImmutableMap.of("user1", new Person("chen", 19), "user2",new Person("chen1", 22));
   Map<String, User> toMap = Maps.transformValues(fromMap, new Function<Person, User>() {
    @Override
    public User apply(Person input) {
        return new User(input);
    }
   });
   ```
3. 根据key和value生成一个新Map
   ```java
   Map<String, User> toMap2 = Maps.transformEntries(fromMap, new EntryTransformer<String, Person, User>() {
    @Override
    public User transformEntry(String key, Person value) {
        return new User(value);
    }
   });
   ```
4. 根据entry过滤
   ```java
   Map<String, Person> toMap3 = Maps.filterEntries(fromMap, new Predicate<Entry<String, Person>>() {
    @Override
    public boolean apply(Entry<String, Person> input) {
        return false;
    }
  });
   ```
5. list,set转Map。Key为List元素处理而来。
   ```java
   List<Person> inList = Lists.newArrayList(new Person("111", 22),new Person("222", 222));
   Map<String, Person> map4 = Maps.uniqueIndex(inList, new Function<Person,String>(){
    @Override
    public String apply(Person input) {
        return input.getName();
    }
   });
   ```
6. set转map，set值做Key
  ```java
  Set<Person> set = Sets.newHashSet(new Person("111", 22),new Person("222", 222));
  Map<Person, String> map5 = Maps.asMap(set, new Function<Person,String>(){
    @Override
    public String apply(Person input) {
        return null;
    }
  });
  ```
-----------

### Joiner，Splitter
  ```java
  * list转换为字符串
 */  
@Test  
public void joinTest(){  
    List<String> names = Lists.newArrayList("John", "Jane", "Adam", "Tom");  
    String result = Joiner.on(",").join(names);  

    assertEquals(result, "John,Jane,Adam,Tom");  
}  


/**
 * map转换为字符串
 */  
@Test  
public void whenConvertMapToString_thenConverted() {  
    Map<String, Integer> salary = Maps.newHashMap();  
    salary.put("John", 1000);  
    salary.put("Jane", 1500);  
    String result = Joiner.on(" , ").withKeyValueSeparator(" = ")  
                                    .join(salary);  
    System.out.println(result);  
}  

/**
 * list转String，跳过null
 */  
@Test  
public void whenConvertListToStringAndSkipNull_thenConverted() {  
    List<String> names = Lists.newArrayList("John", null, "Jane", "Adam", "Tom");  
    String result = Joiner.on(",").skipNulls().join(names);  
    System.out.println(result);  
    assertEquals(result, "John,Jane,Adam,Tom");  
}  

/**
 * list转String，将null变成其他值
 */  
@Test  
public void whenUseForNull_thenUsed() {  
    List<String> names = Lists.newArrayList("John", null, "Jane", "Adam", "Tom");  
    String result = Joiner.on(",").useForNull("nameless").join(names);  
    System.out.println(result);  
    assertEquals(result, "John,nameless,Jane,Adam,Tom");  
}  

/**
 * String to List
 */  
@Test  
public void whenCreateListFromString_thenCreated() {  
    String input = "apple - banana - orange";  
    List<String> result = Splitter.on("-").trimResults().splitToList(input);  
    System.out.println(result);  
    //assertThat(result, contains("apple", "banana", "orange"));  
}  

/**
 * String to Map
 */  
@Test  
public void whenCreateMapFromString_thenCreated() {  
    String input = "John=first,Adam=second";  
    Map<String, String> result = Splitter.on(",")  
                                         .withKeyValueSeparator("=")  
                                         .split(input);  

    assertEquals("first", result.get("John"));  
    assertEquals("second", result.get("Adam"));  
}  

/**
 * 多个字符进行分割
 */  
@Test  
public void whenSplitStringOnMultipleSeparator_thenSplit() {  
    String input = "apple.banana,,orange,,.";  
    List<String> result = Splitter.onPattern("[.|,]")  
                                  .omitEmptyStrings()  
                                  .splitToList(input);  
    System.out.println(result);  
}  

/**
 * 每隔多少字符进行分割
 */  
@Test  
public void whenSplitStringOnSpecificLength_thenSplit() {  
    String input = "Hello world";  
    List<String> result = Splitter.fixedLength(3).splitToList(input);  
    System.out.println(result);  
}  

/**
 * 限制分割多少字后停止
 */  
@Test  
public void whenLimitSplitting_thenLimited() {  
    String input = "a,b,c,d,e";  
    List<String> result = Splitter.on(",")  
                                  .limit(4)  
                                  .splitToList(input);  

    assertEquals(4, result.size());  
    System.out.println(result);  
}  
  ```
--------

### Cache
关于guava的cache，从spring 5开始就不再支持了，改用caffeine。但是这里还是写一下吧。caffeine是基于java8的，java8以下还是只能用guava。官方号称caffeine比guava的cache快。不过caffeine的风格基本照搬guava的cache

```java
  private LoadingCache<String, List<User>> userCache = CacheBuilder.newBuilder()
        //过期时间
        .expireAfterWrite(5, SECONDS)
        //最大长度
        .maximumSize(10).build(new CacheLoader<String, List<User>>() {
            @Override
            public List<User> load(String key) throws Exception {
              ....进行BD查询返回操作等
            }
        });
```

如果想定时刷新，使用执行器加重写reload方法
```java
private LoadingCache<String, List<User>> userCache = CacheBuilder.newBuilder()
      //过期时间
      .expireAfterWrite(5, SECONDS)
      //最大长度
      .maximumSize(10).build(new CacheLoader<String, List<User>>() {
          @Override
          public List<User> load(String key) throws Exception {
            ....进行BD查询返回操作等
          }

          @Override
          public ListenableFuture<List<User>> reload(String key, final List<User> oldValue)
                  throws Exception {
              return EXECUTOR.submit(() -> {
                  List<User> list = doFindUser();
              });
          }
      });

//再加个定时器
@PostConstruct
public void init() throws Exception {
    // 定时刷新 5秒
    EXECUTOR.scheduleWithFixedDelay(() -> USER_CACHE.refresh(StringUtils.EMPTY), 3, 5, TimeUnit.SECONDS);
    // 预加载
    doFindUser();
}
```
或者使用
```java
@PostConstruct
private void init() {
    refresh();
}

@Scheduled(fixedDelay = 30_000, initialDelay = 5_000)
private void refresh() {
       USER_CACHE.refresh(StringUtils.EMPTY);
}
```
