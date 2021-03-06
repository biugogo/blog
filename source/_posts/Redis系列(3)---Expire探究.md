---
title: Redis系列（3）---Expire探究
date: 2019-11-16 21:22:11
tags:
 -Redis
categories: DB
thumbnail: /gallery/marvel/1557842931503.jpg
---

# Redis系列（3）---Expire探究

---

### Redis Expire生命周期
1. 对于Key设置的Expire，在删除Key或者复写Key时，会被清除（DEL,SET,GETSET）
2. 对于Key设置的Expire，在对Value操作时，不会被清除。（INCR,LPUSH,HSET）
3. RENAME命令可以带会把旧Key的Expire带给新的命名。
4. Key没有Expire这个状态，一旦过期将会被删除

### 命令返回值

1. Expire：1代表设置成功，0代表Key不存在
2. ttl: -1代表没有过期时间，10代表还有10秒过期

### Redis依赖系统时间
Redis2.4中，可能有1秒左右误差，2.6之后，误差被控制在1毫秒之内。（2.6之前用的unix时间戳是秒，2.6改为了毫秒）。
redis服务器系统时间一定要准确，假如你修改了系统时间，会导致Expire系列命令失效（比如你设置一个Key ttl为1000秒，把服务器时间往前调2000秒，1000秒过后，这个Key不会被删除）

### Redis过期算法

Redis过期分为主动过期和被动过期两种。**被动过期**：这个Key被请求到，redis检查它过期了就会删除它。**主动过期**：Redis定期随机扫描一些Key，并删除其中过期的。这种方式的细节：
<br>
Redis默认会运行以下程序1秒10次

1. 随机从redis抽取20个带有过期时间的Key
2. 删除其中过期的Key
3. 如果其中有超过1/4的Key被删除，循环step.1

这个概率算法可以很容易算得redis中过期Key的数量的期望是每秒最大写入数据的1/4。

### Expire的同步

为了不牺牲主从（AOF）数据一致性，当一个Key过期，Redis的Slave（AOF）不会自己删除Key，而是等待Master节点删除过期Key，并把DEL命令同步给Slave（AOF）。所有的DEL操作发生在Master上。Slave上有完整的Expire信息，以便发生故障晋升为Master时，能够正确的进行过期操作。

### Expire的内存消耗

为Key设置expire，会带来额外的内存消耗。








