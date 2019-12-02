# Redis系列（3）---Expire探究

---

### Redis Expire生命周期
1. 对于Key设置的Expire，在删除key或者复写key时，会被清除（DEL,SET,GETSET）
2. 对于Key设置的Expire，在对value操作时，不会被清除。（INCR,LPUSH,HSET）
3. RENAME命令可以带会把旧Key的Expire带给新的命名。
4. Key没有Expire这个状态，一旦过期将会被删除

### 命令返回值

1. Expire：1代表设置成功，0代表key不存在
2. ttl: -1代表没有过期时间，10代表还有10秒过期

### Redis依赖系统时间
Redis2.4中，可能有1秒左右误差，2.6之后，误差被控制在1毫秒之内。（2.6之前用的unix时间戳是秒，2.6改为了毫秒）。
redis服务器系统时间一定要准确，假如你修改了系统时间，会导致expire系列命令失效（比如你设置一个Key ttl为1000秒，把服务器时间往前调2000秒，1000秒过后，这个key不会被删除）

### Redis过期算法

Redis过期分为主动过期和被动过期两种。**被动过期**：这个key被请求到，redis检查它过期了就会删除它。**主动过期**：Redis定期随机扫描一些Key，并删除其中过期的。这种方式的细节：
<br>
Redis默认会运行以下程序1秒10次

1. 随机从redis抽取20个带有过期时间的Key
2. 删除其中过期的key
3. 如果其中有超过1/4的key被删除，循环step.1

这个概率算法可以






