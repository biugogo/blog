---
title: Common Lock
date: 2018-2-07 16:14:10
tags:
 -lock
categories: lock
thumbnail: https://raw.githubusercontent.com/apollochen123/image/master/%E9%BB%98%E8%AE%A42.jpg
---

# Common Lock
## Overview
最近在学习concurrent包的一些用法，看到了lock，然后源码里提到了CLH等概念，就学习了一下。参考了以下博客的作者的博文，很多都是搬过来的，感兴趣的同学可以参考原博客：
[博客1](https://coderbee.net/index.php/concurrent/20131115/577)
[博客2](https://www.cnblogs.com/yuyutianxia/p/4296220.html)
[博客3](http://blog.csdn.net/gatieme/article/details/52098615)

------------------
## Basic Knowledge
* 1.1 SMP(Symmetric Multi-Processor)
 对称多处理器结构，指服务器中多个CPU对称工作，每个CPU访问内存地址所需时间相同。其主要特征是共享，包含对CPU，内存，I/O等进行共享。SMP能够保证内存一致性，但这些共享的资源很可能成为性能瓶颈，随着CPU数量的增加，每个CPU都要访问相同的内存资源，可能导致内存访问冲突，可能会导致CPU资源的浪费。常用的PC机就属于这种。

* 1.2 NUMA(Non-Uniform Memory Access)
 非一致存储访问，将CPU分为CPU模块，每个CPU模块由多个CPU组成，并且具有独立的本地内存、I/O槽口等，模块之间可以通过互联模块相互访问,访问本地内存的速度将远远高于访问远地内存(系统内其它节点的内存)的速度，这也是非一致存储访问的由来。NUMA较好地解决SMP的扩展问题,当CPU数量增加时，因为访问远地内存的延时远远超过本地内存，系统性能无法线性增加。

 对此感兴趣的朋友可以参考[博客3](http://blog.csdn.net/gatieme/article/details/52098615)。详细介绍了SMP，NUMA，MPP，COMA，CCNUMA
------------
## CommonLock
这里主要用java实现了一下简单自旋锁，票锁，CLH锁和MCS锁。比较粗显，欢迎大家指正。

-------
### SpinLock
**自旋锁** 是指当一个线程尝试获取某个锁时，如果该锁已被其他线程占用，就一直循环检测锁是否被释放，而不是进入线程挂起或睡眠状态。自旋锁适用于锁保护的临界区很小的情况，临界区很小的话，锁占用的时间就很短。

**缺点** ：CAS操作需要硬件的配合； 保证各个CPU的缓存（L1、L2、L3、跨CPU Socket、主存）的数据一致性，通讯开销很大，在多处理器系统上更严重； 没法保证公平性，不保证等待进程/线程按照FIFO顺序获得锁。
#### diagram
![image](https://raw.githubusercontent.com/apollochen123/image/master/spinLock.png)
#### Simple Java Implements
```java
public static class SpinLock
{
    private AtomicReference<Thread> lockOwner = new AtomicReference<>();

    public void lock()
    {
        Thread thread = Thread.currentThread();
        //如果当前锁被占用，则自旋,这里用AtomicReference是为了使用它的原子性的compareAndSet方法（CAS操作），解决了多线程并发操作导致数据不一致的问题，确保其他线程可以看到锁的真实状态。
        while (!lockOwner.compareAndSet(null, thread))
        {
            System.out.println("正在等待拿锁");
            sleep(1000);//降低负载1秒取一次锁,自旋锁很占cpu资源，如果一直自旋会占满cpu的所有资源
        }

    }

    public void unlock()
    {
        Thread thread = Thread.currentThread();
        //解除锁
        lockOwner.compareAndSet(thread, null);
        System.out.println("锁已释放");
    }
}

public static void spinLockTest()
{
    SpinLock lock = new SpinLock();
    ExecutorService executor = Executors.newFixedThreadPool(2);
    executor.submit(() -> {
        lock.lock();
        System.out.println("66666666");
        sleep(2000);
        lock.unlock();
    });
    executor.submit(() -> {
        lock.lock();
        System.out.println("77777777");
        lock.unlock();
    });
    shutdown(executor);
}
```
------
### TicketLock
**Ticket Lock** 是为了解决上面的公平性问题，类似于现实中银行柜台的排队叫号：锁拥有一个服务号，表示正在服务的线程，还有一个排队号；每个线程尝试获取锁之前先拿一个排队号，然后不断轮询锁的当前服务号是否是自己的排队号，如果是，则表示自己拥有了锁，不是则继续轮询。当线程释放锁时，将服务号加1，这样下一个线程看到这个变化，就退出自旋。

**缺点**： Ticket Lock 虽然解决了公平性的问题。 但是多处理器系统上，每个进程/线程占用的处理器都在读写同一个变量serviceNum，每次读写操作都必须在多个处理器缓存之间进行缓存同步，这会导致繁重的系统总线和内存的流量，大大降低系统整体的性能。
#### diagram
![image](https://raw.githubusercontent.com/apollochen123/image/master/tiketLock.png)
#### Simple Java Implements
```java
public static class TicketLock
{
    private AtomicInteger serviceNum = new AtomicInteger();
    private AtomicInteger ticketNum = new AtomicInteger();

    public int lock()
    {
        int myTicketNum = ticketNum.getAndIncrement();
        while (serviceNum.get() != myTicketNum)
        {
            System.out.println("正在等待");
            //只要不是自己的服务号，不断轮询
            sleep(1000);//降低cpu负载和增加输出可读性sleep一秒
        }
        return myTicketNum;
    }

    public void unlock(int myTicket)
    {
        int next = myTicket + 1;
        //只有正在服务ticket才能释放锁
        serviceNum.compareAndSet(myTicket, next);
        System.out.println("锁已经释放");
    }
}

public static void ticketLockTest()
{
    TicketLock lock = new TicketLock();
    ExecutorService executor = Executors.newFixedThreadPool(2);
    executor.submit(() -> {
        int ticket = lock.lock();
        System.out.println("66666666");
        sleep(2000);
        lock.unlock(ticket);
    });
    executor.submit(() -> {
        int ticket = lock.lock();
        System.out.println("77777777");
        lock.unlock(ticket);
    });
    shutdown(executor);
}
```
-------
### MCSLock
MCS Spinlock 是一种基于链表的可扩展、高性能、公平的自旋锁，申请线程只在本地变量上自旋，直接前驱负责通知其结束自旋，从而极大地减少了不必要的处理器缓存同步的次数，降低了总线和内存的开销。

#### diagram
##### lock:
![image](https://raw.githubusercontent.com/apollochen123/image/master/MCSLock.png)
##### unlock
![image](https://raw.githubusercontent.com/apollochen123/image/master/MCSUnlock.png)

#### Simple Java Implements
```java
public static class MCSLock
{
    public static class MCSNode
    {
        volatile MCSNode next;
        volatile boolean isBlock = true; // 默认是在等待锁
    }

    volatile MCSNode queue;//指向最后一个申请锁的node

    //原子更新器更新MCSLock类里面的MCSNode类型的queue变量
    private static final AtomicReferenceFieldUpdater<MCSLock, MCSNode> UPDATER = AtomicReferenceFieldUpdater.newUpdater(MCSLock.class,
        MCSNode.class, "queue");

    public void lock(MCSNode currentThreadNode)
    {
        //拿到前节点
        MCSNode pre = UPDATER.getAndSet(this, currentThreadNode);
        if(pre!=null){ //如果前节点不为空，则锁已经被占用
            pre.next = currentThreadNode; //设置per下一节点为当前节点
            while(currentThreadNode.isBlock){ //当前节点进入自旋

            }
        }
        //前节点为null，只有当前线程在申请锁
        else{  
            currentThreadNode.isBlock = false;
        }
    }

    public void unlock(MCSNode currentThreadNode){
        //如果当前节点取得了lock，则node一定是fasle。如果前一节点使用pre.next释放lock，则无意义返回
        if(currentThreadNode.isBlock){
            return ;
        }
        //如果当前节点的下一节点为null，有两种可能。1.确实是没有后节点，此时queue等于当前节点 2.MCSNode pre = UPDATER.getAndSet(this, currentThreadNode)，但是还没执行 pre.next = currentThreadNode;
        if(currentThreadNode.next == null){
            if(UPDATER.compareAndSet(this, currentThreadNode, null)){
                return ;
            }
            else{
                while(currentThreadNode.next == null){};
            }
        }
        //让后节点退出自旋
        currentThreadNode.next.isBlock=false;
        //gc
        currentThreadNode.next = null;
    }

}

public static void mcsLockTest()
{
    MCSLock lock = new MCSLock();
    ExecutorService executor = Executors.newFixedThreadPool(3);
    executor.submit(() -> {
        MCSLock.MCSNode mCSNode = new MCSLock.MCSNode();
        lock.lock(mCSNode);
        System.out.println("66666666");
        sleep(2000);
        lock.unlock(mCSNode);
    });
    executor.submit(() -> {
        MCSLock.MCSNode mCSNode = new MCSLock.MCSNode();
        lock.lock(mCSNode);
        System.out.println("77777777");
        sleep(2000);
        lock.unlock(mCSNode);
    });
    executor.submit(() -> {
        MCSLock.MCSNode mCSNode = new MCSLock.MCSNode();
        lock.lock(mCSNode);
        System.out.println("8888888");
        sleep(2000);
        lock.unlock(mCSNode);
    });
    shutdown(executor);

}
```
--------
### CLHLock
CLH(Craig, Landin, and Hagersten locks): 是一个自旋锁，能确保无饥饿性，提供先来先服务的公平性。
CLH锁也是一种基于链表的可扩展、高性能、公平的自旋锁，申请线程只在本地变量上自旋，它不断轮询前驱的状态，如果发现前驱释放了锁就结束自旋。 当一个线程需要获取锁时：
1. 创建一个的QNode，将其中的locked设置为true表示需要获取锁
2. 线程对tail域调用getAndSet方法，使自己成为队列的尾部，同时获取一个指向其前趋结点的引用myPred
3. 该线程就在前趋结点的locked字段上旋转，直到前趋结点释放锁
4. 当一个线程需要释放锁时，将当前结点的locked域设置为false，同时回收前趋结点

#### diagram
##### lock:
![image](https://raw.githubusercontent.com/apollochen123/image/master/CLHlock.png)
##### unlock:
![image](https://raw.githubusercontent.com/apollochen123/image/master/CLHUnlock.png)
#### Simple Java Implements
```java
public static class CLHLock
{
    public static class CLHNode
    {
        volatile boolean isBlock = true;
        volatile CLHNode pre; //前一节点
    }

    private static final AtomicReferenceFieldUpdater<CLHLock, CLHNode> UPDATER = AtomicReferenceFieldUpdater.newUpdater(CLHLock.class,
        CLHNode.class, "tail");

    private volatile CLHNode tail;

    public void lock(CLHNode currentThreadNode)
    {
        CLHNode preNode = UPDATER.getAndSet(this, currentThreadNode);//拿到tail节点，再更新tail节点等于本地线程变量currentThreadNode
        System.out.println("拿到tail节点:" + preNode + "再更新tail节点等于本地线程变量currentThreadNode:" + currentThreadNode);
        if (preNode != null) //已有线程占用了锁，进入自旋
        {
            while (preNode.isBlock)
            {
                sleep(1000);
                System.out.println("正在自旋:"+Thread.currentThread());
            }
            ;
        }
    }

    public void unlock(CLHNode currentThreadNode)
    {
        //比较tail节点是否等于currentThreadNode，如果不相等，则设置当前线程node的isBlock为false，如果相等，则设置tail为null
        System.out.println("tail:"+tail);
        System.out.println("currentThreadNode:"+currentThreadNode);
        if (!UPDATER.compareAndSet(this, currentThreadNode, null))
        {
            currentThreadNode.isBlock = false;
        }
    }

}

public static void cLHLockTest()
{
    CLHLock lock = new CLHLock();
    ExecutorService executor = Executors.newFixedThreadPool(3);
    executor.submit(() -> {
        CLHLock.CLHNode cLHNode = new CLHLock.CLHNode();
        lock.lock(cLHNode);
        System.out.println("66666666");
        sleep(2000);
        lock.unlock(cLHNode);
    });
    executor.submit(() -> {
        CLHLock.CLHNode cLHNode = new CLHLock.CLHNode();
        lock.lock(cLHNode);
        System.out.println("77777777");
        sleep(2000);
        lock.unlock(cLHNode);
    });
    executor.submit(() -> {
        CLHLock.CLHNode cLHNode = new CLHLock.CLHNode();
        lock.lock(cLHNode);
        System.out.println("8888888");
        sleep(2000);
        lock.unlock(cLHNode);
    });
    shutdown(executor);
}

```

-------
#### Additional
代码里的shutdown和slee方法如下：
```java

    public static void sleep(long i)
    {
        try
        {
            Thread.sleep(i);
        }
        catch (InterruptedException e)
        {
            e.printStackTrace();
        }
    }

    public static void shutdown(ExecutorService executor)
    {
        try
        {
            System.out.println("attempt to shutdown executor");
            executor.shutdown();
            executor.awaitTermination(10, TimeUnit.SECONDS);
        }
        catch (InterruptedException e)
        {
            System.err.println("tasks interrupted");
        }
        finally
        {
            if (!executor.isTerminated())
            {
                System.err.println("cancel non-finished tasks");
            }
            executor.shutdownNow();
            System.out.println("shutdown finished");
        }
    }
```
