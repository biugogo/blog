---
title: EffectiveJava(2)--基础方法
date: 2019-6-1 12:47:54
tags:
 -EffectiveJava
categories: Basic
thumbnail: /gallery/lol/1557842817807.jpg
---

# EffectiveJava(2)--基础方法

## Overview
1. hashCode()和equals()方法
2. wait()和notify()
3. toString()

-----

### 1.hashCode()和equals()方法

#### equals()原则
1. 自反性(reflexive): 对于任何非null的引用值x x . equals(x)必须返回true.
2. 对称性(symmetric)：对于任何非null的引用值x 和y,当且仅当y.equals(x)返回true 时， x.equals(y)必须返回true.
3. 传递性(transitive): 对于任何非null的引用值x 、y 和z,如果x.equals(y)返回true,并且y.equals(z)也返回true,那么x.equals(z)也必须返回true 0
4. 一致性(consistent): 对于任何非null的引用值x 和y,只要equals 的比较操作在对象中所用的信息没有被修改，多次调用x.equals(y)就会一致地返回true,
或者一致地返回false
5. 对于任何非null 的引用值x, x.equals (null)必须返回false 。

这些原则跟你想象的逻辑很符合，不需要特别记忆。

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

    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        BuilderDemo demo = (BuilderDemo) o;
        return Objects.equals(requiredParam1, demo.requiredParam1) &&
                Objects.equals(requiredParam2, demo.requiredParam2) &&
                Objects.equals(optionalParam1, demo.optionalParam1) &&
                Objects.equals(optionalParam2, demo.optionalParam2);
    }
}
```

#### 覆盖equals时总要覆盖hashCode
1. 相等的对象必须具有相等的散到码hashCode。
2. 相等的hashCode不一定是相等对象。（当然尽量让对象拥有不同的hashCode，可以提高hash table的性能）

这个记忆点可以很简单，想想hashMap的结构，散列+链表。为什么需要链表，就是因为不同对象能别hash到一个链表里。相等对象如果没有相同的hashCode，用hashCode去hashMap里取值时，也就会变得不确定性。

```java
    @Override
    public int hashCode() {
        return Objects.hash(requiredParam1, requiredParam2, optionalParam1, optionalParam2);
    }
```

------------


### 2.wait和notify

1. 始终应该使用wait循环模式来调用wait 方法；永远不要在循环之外调用wait方法,理由:
    1. 另一个线程可能已经得到了锁，并且从一个线程调用notify方法那一刻起，到等待线程苏醒过来的这段时间中，得到锁的线程已经改变了受保护的状态。
    2. 条件并不成立，但是另一个线程可能意外地或恶意地调用了notify方法
    3. 通知线程（ notifying thread ）在唤醒等待线程时可能会过度“大方” 。例如，即使只有某些等待线程的条件已经被满足，但是通知线程可能仍然调用notifyAll方法
    4. 在没有通知的情况下，等待线程也可能（但很少）会苏醒过来。这被称为"伪唤醒"

    ```java
     synchronized (object) {
         while (condition do not hold) {
             object.wait();
         }
     }
    ```

2. notify还是notifyAll
    1. 保守建议：始终使用notifyAll，它总会产生正确的结果，因为它可以保证你将会唤醒所有需要被唤醒的线程。这些线程醒来之后，会检查它们正在等待的条件，如果发现条件并不满足，就会继续等待。
    2. 优化建议：从优化的角度来看，如果处于等待状态的所有线程都在等待同一个条件，而每次只有一个线程可以从这个条件中被唤醒，那么你就应该选择调用notify方法

3. 一个使用notify和wait的例子,打印1-100,线程1打印双数，线程2打印单数:
    ```java
    public class WaitAndNotifyDemo {
    private static volatile int count = 1;

    private static class Printer implements Runnable {
        private String name;
        private final Object object;

        public Printer(String name, Object object) {
            this.name = name;
            this.object = object;
        }

        @Override
        public void run() {
            try {
                while (count < 100) {
                    synchronized (object) {
                        while (count % 2 != 0) {
                            object.wait();
                        }
                        if (count % 2 == 0) {
                            System.out.println(name + ":" + count);
                            count++;
                            object.notify();
                        }
                    }
                }
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

    private static class Printer2 implements Runnable {
        private String name;
        private final Object object;

        public Printer2(String name, Object object) {
            this.name = name;
            this.object = object;
        }

        @Override
        public void run() {
            try {
                while (count < 100) {
                    synchronized (object) {
                        while (count % 2 == 0) {
                            object.wait();
                        }
                        if (count % 2 != 0) {
                            System.out.println(name + ":" + count);
                            count++;
                            object.notify();
                        }
                    }
                }
            } catch (Exception e) {
                e.printStackTrace();
            }
        }

    }

    public static void main(String[] args) {
        Object lock = new Object();
        Printer p = new Printer("p1", lock);
        Printer2 p2 = new Printer2("p2", lock);

        ExecutorService executorService = Executors.newFixedThreadPool(2);
        executorService.execute(p);
        executorService.execute(p2);
    }
    }
    ```

-------

### 3.始终要覆盖toString
1. 提供好的toString实现可以便类用起来更加舒适，使用了这个类的系统也更易于调试
2. 在实际应用中,toString方法应该返回对象中包含的所有值得关注的信息，

#### 3种toString写法

demo类
```java
public class ToStringTest {
    private Integer a;
    private Long b;
    private String c;
    private String d;
    private String e;
}
```

1. "+"连接

```java
    public String toString() {
        return "ToStringTest{" +
                "a=" + a +
                ", b=" + b +
                ", c='" + c + '\'' +
                ", d='" + d + '\'' +
                ", e='" + e + '\'' +
                '}';
    }
```

2. StringBuffer(StringBuilder)
```java
    public String toString() {
        final StringBuilder sb = new StringBuilder("ToStringTest{");
        sb.append("a=").append(a);
        sb.append(", b=").append(b);
        sb.append(", c='").append(c).append('\'');
        sb.append(", d='").append(d).append('\'');
        sb.append(", e='").append(e).append('\'');
        sb.append('}');
        return sb.toString();
    }
```
3. StringJoiner
```java
    public String toString() {
        return new StringJoiner(", ", ToStringTest.class.getSimpleName() + "[", "]")
                .add("a=" + a)
                .add("b=" + b)
                .add("c='" + c + "'")
                .add("d='" + d + "'")
                .add("e='" + e + "'")
                .toString();
    }
```

性能测试结果是：
StringBuffer(StringBuilder)≈"+"，Java8的StringJoiner大约是前2者的3倍。结论是用"+"就好，看起来更加简易。（大多数编译器也会将"+"编译成StringBuilder的形式）


----






