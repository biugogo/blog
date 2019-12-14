# Redis系列(6)---分布式锁

## 分布式锁应该有的特性
作为一个分布式锁，应该具有以下3个特性：
1. **安全性**：安全性也叫互斥性，任何时候，只有一个客户端能够拥有锁
2. **无死锁**：客户端最终一定可以获取锁，即使客户端在释放锁之前崩溃掉了
3. **容错性**：只要Redis大多数结点在运行，client可以持续的申请和释放锁

## 单例分布式锁

### 常见的分布式锁实现

一个常用的实现---**单Redis实例实现**：

1. **申请锁时**，使用以下命令：
    ```java
    SET resource_name my_random_value NX PX 30000
    ```
2. **释放锁时**，使用以下Lua：
    ```java
    if redis.call("get",KEYS[1]) == ARGV[1] then
        return redis.call("del",KEYS[1])
    else
        return 0
    end   
    ```
Java代码完整实现：
```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import redis.clients.jedis.Jedis;
import java.util.UUID;
import java.util.concurrent.Callable;
import java.util.concurrent.ThreadLocalRandom;
import java.util.concurrent.TimeUnit;
import java.util.function.Supplier;

public class RedisLock {

    private static final Logger LOG = LoggerFactory.getLogger(RedisLock.class);
    /**
     * UNLOCK_LUA
     */
    private final static String UNLOCK_LUA = "" +
            "if redis.call(\"get\",KEYS[1]) == ARGV[1] " +
            "then " +
            "   return redis.call(\"del\",KEYS[1]) " +
            "else " +
            "   return 0 " +
            "end";
    // @formatter:on

    /**
     * 等待锁时间
     */
    private final static long WAIT_LOCK_TIME = 100;

    private final Supplier<Jedis> jedisSupplier;

    public RedisLock(Supplier<Jedis> jedisSupplier) {
        this.jedisSupplier = jedisSupplier;
    }
    /**
     * 等锁随机时间
     */
    private long randomSleepingTime() {
        return ThreadLocalRandom.current().nextLong(WAIT_LOCK_TIME);
    }

    /***
     * 尝试获得分布式锁, 如果锁可用, 返回true
     * @param lockKey   锁的键值
     * @param value     锁的值,推荐使用随机字符.
     * @param timeout   超时时间.单位:毫秒.
     * @return true/false
     */
    public boolean tryLock(String lockKey, String value, long timeout) {
        try (Jedis jedis = jedisSupplier.get()) {
            Object result = jedis.set(lockKey, value, "NX", "PX", timeout);
            return "OK".equalsIgnoreCase(result + "");
        }
    }

    /***
     * 如果分布式锁不可用, 当前线程将继续请求, 直到锁可用.
     * @param lockKey   锁的键值
     * @param timeout   超时时间.单位:毫秒.
     * @return 锁的值.
     */
    public String lock(String lockKey, long timeout) {
        String lockVal = UUID.randomUUID().toString();
        for (; ; ) {
            if (!tryLock(lockKey, lockVal, timeout)) {
                try {
                    TimeUnit.MILLISECONDS.sleep(randomSleepingTime());
                } catch (InterruptedException e) {
                    LOG.error("[RedisLock] sleep error.", e);
                }
            } else {
                break;
            }
        }
        return lockVal;
    }

    /***
     * 释放锁.
     * @param lockKey   锁的键值
     * @param value     锁的值.必须要和上锁的值一致.
     */
    public void unlock(String lockKey, String value) {
        try (Jedis jedis = jedisSupplier.get()) {
            jedis.eval(UNLOCK_LUA, 1, lockKey, value);
        }
    }

    /**
     * 查看是否有锁
     * @param lockKey
     * @return
     */
    public boolean existLock(String lockKey) {
        try (Jedis jedis = jedisSupplier.get()) {
            return jedis.exists(lockKey);
        }
    }

    public void runWithLock(String lockKey, Runnable runnable) {
        String lockValue = null;
        try {
            lockValue = lock(lockKey, 3000);
            runnable.run();
        } finally {
            if (lockValue != null) {
                unlock(lockKey, lockValue);
            }
        }
    }

    public <T> T callWithLock(String lockKey, Callable<T> callable) throws Exception {
        String lockValue = null;
        try {
            lockValue = lock(lockKey, 3000);
            return callable.call();
        } finally {
            if (lockValue != null) {
                unlock(lockKey, lockValue);
            }
        }
    }
}

```
### 常用实现的不足
1. **容错性不足**：
    作为一个分布式锁，这个锁只能由单例Redis去实现。但是单例Redis实现，无法保证Redis挂掉后还能持续获得锁，容易出现单点故障导致整个系统崩溃。
2. **滥用**
    前文说到，我们只能由单例Redis去实现，但是通常在生产环境我们使用sentinel模式来保证Redis的HA。所以我们错误的认为在sentinel模式下上述实现也是可用的。设想以下情景：
    * Client A重Master申请到了一个锁。
    * 在Master将锁同步给Slave之前，Master挂掉了。
    * 发生故障转移，Slave成为Master。
    * Client B申请锁，此时从Slave晋升为Master的结点，很自然的没有锁信息，再给B一个锁。此时ClientA、ClinetB同时拥有了锁。不能保证互斥性。

这就有个很纠结的问题，要么放弃容错性，要么放弃绝对的互斥性。当然在大多数业务中，我们在sentinel模式下使用上述Redis锁都是可用的（反正我们生产业务是这么用的，当然与钱挂钩的重要业务还是慎重使用）

## RedLock算法
首先准备N个在不同机房，不同机器上Redis实例，保证他们的独立性。官方给出建议是，5个

1. 获取本地时间
2. 使用相同的Key和随机Value，尝试从N个Redis实例中申请锁。然后设置一个超时时间，作用是一旦未能成功获取到锁的情况下快速释放锁。例如申请一个10秒的锁，我们通常设置超时时间为5-50ms。这样可以避免客户端与一个已经故障的Master通信占用太长时间，通过快速失败的方式尽快的与集群中的其他节点完成锁操作。
3. 获取锁成功的标志是：获取到**超过一半**实例的锁，同时申请时长（申请时长=当前时间-step1的时间)小于锁超时时间（这里是10秒）。
4. 如果成功获得到锁，客户端持有锁的时间窗口=锁超时时间（这里是10秒）-申请时长。
5. 如果申请失败（没有获得到一半以上的实例的锁），我们需要释放申请到的所有锁来保证快速失败


RedLock是否具备分布式锁的特性：
1. RedLock安全性：如果我们获取到锁，那么我们一定拿到了一半以上的锁，并且在TTL-获取锁时长-时钟飘逸时间内，其他实例是不可能获取到锁的
2. RedLock无死锁：我们获取锁，都会被最终释放。如果释放锁失败，ttl时间后，锁还是会变得可用。
3. RedLock容错性：只要存在一半以上Redis实例,Client就有持续获取锁的能力。假如其中节点挂掉超过一半，整个系统无法获取锁。
