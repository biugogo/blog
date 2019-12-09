# Redis系列(5)---内存管理策略

Redis占用的系统内存，可能会稍大于我们设置的maxmemory的大小。

### 1.关于redis的内存管理配置：
1. 当remove发生时，Redis不会立即释放内存给OS。这不是redis的锅，时因为动态内存分配函数malloc()的问题。不立即回收remove的key占用的内存，是因为大多数释放的内存，跟未释放的内存在一片区域，内存碎片化，释放时，需要整理内存（通过安全重启的方式）。
2. 因为内存碎片的存在，分配内存时，你需要考虑按照峰值内存分配同时也得考虑内存利用率不可能达到100%。
3. 内存分配器可以重用remove释放的内存，而不会去向OS申请新的内存（如果可重用内存足够）

<br>
一份典型的Memory Info:

```c
used_memory:316355368                   --已经使用内存
used_memory_human:301.70M               
used_memory_rss:374353920               --Redis进程占据操作系统的内存（单位是字节）
used_memory_rss_human:357.01M
used_memory_peak:382487016              --内存峰值
used_memory_peak_human:364.77M
total_system_memory:67523944448         --系统总内存
total_system_memory_human:62.89G
used_memory_lua:61440                   --lua脚本占用内存
used_memory_lua_human:60.00K
maxmemory:4294967296                    --设置的redis最大内存（32位系统最大可以设置3G，64位系统如果设置为0表示没有限制）
maxmemory_human:4.00G
maxmemory_policy:allkeys-lru            --key淘汰策略allkeys-lru
mem_fragmentation_ratio:1.18            --内存碎片率（used_memory_rss / used_memory），一般大于1，且该值越大，内存碎片比例越大
mem_allocator:jemalloc-4.0.3            --Redis使用的内存分配器，在编译时指定；可以是 libc 、jemalloc或者tcmalloc，默认是jemalloc
```

### 2.Redis内存淘汰策略

当写入key到达设定的阀值，Redis提供了不同的几种策略，供我们选择：
1. **noeviction**: 触发内存阈值的写入操作(del 等少数命令除外)直接返回一个错误。
2. **allkeys-lru**: 最近最少使用过期策略
3. **volatile-lru**: 最近做少使用过期策略，不过只淘汰带有ttl的key
4. **allkeys-random**: 随机过期策略
5. **volatile-random**: 随机过期策略，不过只淘汰带有ttl的key
6. **volatile-ttl**: 淘汰最快过期的key

上述volatile淘汰策略中，如果没有带有ttl的key，Redis对外表现和noeviction一样。

### 3.Redis对于LRU算法的实现

Redis的LRU算法是一个近似LRU算法，如果实现一个标准的LRU算法（Hash+链表的方式），我们需要将所有KEY用一个链表装起来。这会消耗大量的内存。这对于节约内存到丧心病狂的Redis开发者来说，基本上是不可以接受的。所以Redis的LRU算法，是一个近似LRU算法。
<br>

我们可以通过调整参数**maxmemory-samples**的大小，来实现进行LRU精度的修改。官方给出的实验结果：
![Redis LRU实验图](https://redis.io/images/redisdoc/lru_comparison.png)

1. 亮灰色表示被排除的KEY
2. 灰色代表没有被排除的KEY
3. 绿色代表新增的KEY

这个实验，首先先填充满Redis，然后再添加50%的新key，让Redis可以过期最近最少使用的Key。在标准的LRU算法中，最近最少使用的KEY全部被排除。在Redis 2.8中，少量的新增KEY被删除了。Redis 3.0后改进了LRU算法，发现新增的KEY不再被删除了，并且随着**maxmemory-samples**参数的增加，实验结果越发接近标准的LRU算法。


#### 1.Redis LRU算法实现（2.8）:
https://yq.aliyun.com/articles/63034?spm=a2c4e.11153940.0.0.2cb0711c70GO5T

#### 2.Redis LRU算法实现（3.0后）：
https://yq.aliyun.com/articles/64435?utm_campaign=wenzhang&utm_medium=article&utm_source=QQ-qun&utm_content=m_7758
