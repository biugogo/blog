# Redis系列(2)---Lua脚本

Redis 从2.6.0版本开始支持Lua脚本（5.1），并且默认加载了以下库：
  1. base
  2. table
  3. string
  4. math
  5. debug
  6. cjson (以非常快的速度处理JSON数据)
  7. cmsgpack

### 一个简单demo

```Lua
eval "return {KEYS[1],KEYS[2],ARGV[1],ARGV[2]}" 2 key1 key2 first second
```

1. 第一个参数-->脚本本身
2. 第二个参数-->脚本涉及redis的key的个数
3. 第三四个参数-->脚本涉及的redis key（对应KEYS[1],KEYS[2]）
4. 第五六个参数-->普通参数（对应ARGV[1],ARGV[2]）

使用这样的设计，有一个重要的功能就是，确保Redis Cluster可以通过KEYS去hash，将请求发送到正确的集群节点。
在集群中使用Lua，必须确保所有涉及的key会被hash到同一个slot上，这样的Lua脚本是RedisCluster兼容的。虽然这不是强制的，比如你在非集群环境中使用Lua，可以不遵守此项规定。


### Lua数据类型和Redis数据类型

![redis与Lua数据转换](https://makefriends.bs2dl.yy.com/bm1574922532966.jpg)

除了上述规则还有一些重要规则：
1. Lua只有一种数字类型---number，所以float和integer在Lua没有区别，如果你想用Lua返回float类型，请用string代替。
2. Lua的array没有nil，所以redis遇到数组中有nil将会停止转换
3. Lua的table返回只会带value，不会含有key（想要kv可以用json）

这些转换规则，在写Lua脚本的时候尤为注意。

### 脚本原子性

Redis使用单个Lua解释器去运行所有脚本，并且Redis也保证脚本会以原子性(atomic)的方式执行：当某个脚本正在运行的时候，不会有其他脚本或Redis命令被执行。

**脚本的运行开销非常少，但是执行一个运行缓慢的脚本并不是一个好主意，会阻塞其他客户端对redis的请求**

### 使用EVALSHA减少带宽

EVAL 命令要求你在每次执行脚本的时候都发送一次脚本主体(script body)，Redis 有一个内部的缓存机制，因此它不会每次都重新编译脚本。为了减少带宽的消耗，Redis实现了EVALSHA命令，它的作用和EVAL一样，都用于对脚本求值，但它接受的第一个参数不是脚本，而是脚本的SHA1校验和。
<br>
Redis保证所有被运行过的脚本都会被永久保存在脚本缓存当中。刷新脚本缓存的唯一办法是显式地调用SCRIPTFLUSH命令，清空运行过的所有脚本的缓存。缓存可以长时间储存而不产生内存问题的原因是，它们的体积非常小，而且数量也非常少，即使脚本在概念上类似于实现一个新命令，即使在一个大规模的程序里有成百上千的脚本，即使这些脚本会经常修改，即便如此，储存这些脚本的内存仍然是微不足道的。


### SCRIPT命令
1. SCRIPT FLUSH ：清除所有脚本缓存
2. SCRIPT EXISTS ：根据给定的脚本校验和，检查指定的脚本是否存在于脚本缓存
3. SCRIPT LOAD ：将一个脚本装入脚本缓存，但并不立即运行它
4. SCRIPT KILL ：杀死当前正在运行的脚本

### 脚本数据同步

#### Lua数据同步
Redis Lua数据同步有两种方式
1. **whole scripts replication** 脚本同步---从节点（或AOF）通过运行主上的脚本，来达到数据同步的作用。脚本同步的优点是，耗费更少的带宽和很少的CPU占用量（从网络上接收命令消耗资源比让CPU重跑一次脚本多）
2. **scripts effects replication** 脚本影响同步---Lua运行过程中，Redis收集所有对数据集的修改的命令，当脚本执行结束后，收集的命令将会被MULTI/EXEC事务包裹传递到从节点或者AOF。

同步方式在各个版本的变化：
1. Reids 3.2之前:只有 **whole scripts replication** 同步。
2. Reids 3.2-5.0之间:支持 **whole scripts replication**(默认)和**scripts effects replication**。可以通过在脚本运行写命令之前显示调用以下命令开启：
  ```
  redis.replicate_commands()
  ```
3. Reids 5.0之后:**scripts effects replication**是不再需要显示开启，默认开启。

#### 脚本编写规则

由于**whole scripts replication**的同步原理，Redis Lua需要遵循一些原则：
1. Redis的写命令必须写入一个定量，而不是随机量。
2. 脚本执行的操作不能依赖于任何隐藏(非显式)数据（Lua 没有访问系统时间或者其他内部状态的命令）
3. 不能依赖于脚本在执行过程中或脚本在不同执行时期之间可能变更的状态（随机命令报错，如RANDOMKEY、SRANDMEMBER、TIME等）
4. 不能依赖于任何来自I/O设备的外部输入
5. 不应该尝试去访问外部系统(比如文件系统)，或者执行任何系统调用
6. 不允许创建全局变量（非loacl）。如果一个脚本需要在多次执行之间维持某种状态，它应该使用Redis key来进行状态保存。
<br>
Redis 3.2之前Lua不提供系统时间或者其他外部状态。同时，在Redis的执行随机命令（RANDOMKEY, SRANDMEMBER, TIME）后，写操作将会导致Redis Lua报错，但是如果Lua中没有进行写操作，这些命令是可以使用的。
<br>
在Redis4.0之后，可以使用SMEMBERS，SPOP等操作在Lua中，因为在Set在Lua中做了特殊处理（会对无序的Set进行固定排序操作）。但是在Redis5中又放弃了这项操作（因为Redis5改用了command复制）

#### 选择性开启同步
在**scripts effects replication**模式下，我们可以使用一下命令选择性开启关闭同步。（这是一项专业要求很高的功能，使用不当会导致主从或者AOF之间数据不同步）
  ```
  redis.set_repl(redis.REPL_ALL) -- Replicate to AOF and replicas.
  redis.set_repl(redis.REPL_AOF) -- Replicate only to AOF.
  redis.set_repl(redis.REPL_REPLICA) -- Replicate only to replicas (Redis >= 5)
  redis.set_repl(redis.REPL_SLAVE) -- Used for backward compatibility, the same as REPL_REPLICA.
  redis.set_repl(redis.REPL_NONE) -- Don't replicate at all.
  ```

  一个例子

  ```
  redis.replicate_commands() -- Enable effects replication.
  redis.call('set','A','1')
  redis.set_repl(redis.REPL_NONE)
  redis.call('set','B','2')
  redis.set_repl(redis.REPL_ALL)
  redis.call('set','C','3')
  ```
  Run上面的脚本，只有A和C会在从节点和AOF中创建，B不会被同步。
<br>
适用场景：我们在脚本中求两个Set的交集，生成一个集合A，但是我们只想取集合A的子集B，然后把B同步给从节点和AOF子集，在主上删除A。这种情况下，A集合只是一个临时量，不需要从节点先创建再删除。这种场景尅使用选择性复制。



### 脚本超时

脚本执行最长时间限制默认是5秒，可以通过修改Lua-time-limit来改动。当一个脚本达到最大执行时间的时候，它并不会自动被Redis结束，因为Redis必须保证脚本执行的原子性，而中途停止脚本的运行意味着可能会留下未处理完的数据在数据集里面。所以脚本超时：
1. Redis记录一个脚本正在超时运行
2. Redis开始重新接受其他客户端的命令请求，但是只有SCRIPT KILL和SHUTDOWN NOSAVE两个命令会被处理，对于其他命令请求，Redis服务器只是简单地返回BUSY错误。
3. 可以使用SCRIPT KILL命令将一个仅执行只读命令的脚本杀死，因为只读命令并不修改数据
4. 如果脚本已经执行过写命令，那么唯一允许执行的操作就是SHUTDOWN NOSAVE,它通过停止服务器来阻止当前数据集写入磁盘.

### 一个使用Lua实现匹配的Demo


