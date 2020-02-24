---
title: Redis系列(5)---内存管理策略
date: 2019-12-07 19:13:50
tags:
 -Redis
categories: DB
thumbnail: /gallery/marvel/1557842931503.jpg
---


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

2.8中，Redis实现LRU算法的原理是，为每个Key设置一个24位的时钟标志。然后再LRU淘汰时，随机load一批Key，校验时钟标志（每次被访问会更新），然后淘汰掉这批随机Key里面最长未被访问的数据。如果使用传统LRU算法，我们不得不为每个Key增加两个指针，用来形成链表，两个指针就算在32位机器上，也会带来64位的消耗。

1. 首先是24位时钟标记的更新
```C++
/**24位*/
#define REDIS_LRU_BITS 24
unsigned lruclock:REDIS_LRU_BITS;

/**LRU最大时钟是24位能表示的最大数值 2的24次方秒，合194天左右**/
#define REDIS_LRU_CLOCK_MAX ((1<<REDIS_LRU_BITS)-1)
 /* LRU时钟精度 1秒*/
#define REDIS_LRU_CLOCK_RESOLUTION 1


.....

/**
 * 更新lru时钟，根据server.unixtime 交于 REDIS_LRU_CLOCK_MAX ，使lrulock在 0-REDIS_LRU_CLOCK_MAX之间循环增加
**/
void updateLRUClock(void) {
    server.lruclock = (server.unixtime/REDIS_LRU_CLOCK_RESOLUTION) &REDIS_LRU_CLOCK_MAX;
}

```
updateLRUClock(void)会在Redis的serverCron()中循环调用，来达到根据server.unixtime系统时间让server.lruclock变化的目的。这个循环调用默认是100ms一次。

2. LRU淘汰流程
```C++
            ......
            /* volatile-lru and allkeys-lru policy(如果淘汰策略是REDIS_MAXMEMORY_ALLKEYS_LRU和REDIS_MAXMEMORY_VOLATILE_LRU )*/
            else if (server.maxmemory_policy == REDIS_MAXMEMORY_ALLKEYS_LRU ||
                server.maxmemory_policy == REDIS_MAXMEMORY_VOLATILE_LRU)
            {/*这里就是我们配置的maxmemory_samples，值越大，随机取Key的次数越大， 样本越大，得到的最大过期Key越精确*/
                for (k = 0; k < server.maxmemory_samples; k++) {
                    sds thiskey;
                    long thisval;
                    robj *o;
                    /**随机取Key**/
                    de = dictGetRandomKey(dict);
                    thiskey = dictGetKey(de);
                    /* When policy is volatile-lru we need an additional lookup
                     * to locate the real key, as dict is set to db->expires. */
                    if (server.maxmemory_policy == REDIS_MAXMEMORY_VOLATILE_LRU)
                        de = dictFind(db->dict, thiskey);
                    o = dictGetVal(de);
                    /**得到key的时钟差值，越大代表最近最久未使用**/
                    thisval = estimateObjectIdleTime(o);

                    /* Higher idle time is better candidate for deletion */
                    if (bestkey == NULL || thisval > bestval) {
                        bestkey = thiskey;
                        bestval = thisval;
                    }
                }
            }
            ......
```
通过循环maxmemory_samples次，随机取Key，早出一个最久未使用的Key进行淘汰。我们在看看estimateObjectIdleTime函数
```C++
unsigned long estimateObjectIdleTime(robj *o) {
    if (server.lruclock >= o->lru) {
        return (server.lruclock - o->lru) * REDIS_LRU_CLOCK_RESOLUTION;
    } else {
        return ((REDIS_LRU_CLOCK_MAX - o->lru) + server.lruclock) *
                    REDIS_LRU_CLOCK_RESOLUTION;
    }
}
```
自己相减，求区间大小。这里server.lruclock可能会归零，所以多了一层else处理，可能会多加REDIS_LRU_CLOCK_MAX

3. 当Key被访问时，更新Key的24位时钟值。注意的是，为了避免fork子进程后额外的内存消耗，当Redis处于bgsave或aof rewrite时，lru访问时间是不更新的
```C++
robj *lookupKey(redisDb *db, robj *key) {
    dictEntry *de = dictFind(db->dict,key->ptr);
    if (de) {
        robj *val = dictGetVal(de);

        /* Update the access time for the ageing algorithm.
         * Don't do it if we have a saving child, as this will trigger
         * a copy on write madness. */
        if (server.rdb_child_pid == -1 && server.aof_child_pid == -1)
            val->lru = server.lruclock;
        return val;
    } else {
        return NULL;
    }
}
```


#### 2.Redis LRU算法实现（3.0后）：

如果研究Redis 2.8的LRU算法，很容易发现问题：
1. 时钟频率只能到秒级别
2. 如果抽取的N个样本凑巧都是很新的Key，可能会淘汰掉较新的Key。违背LRU算法
这两点，导致2.8的LRU算法，不是很接近于真正LRU算法的结果。
<br>

**Redis 3.0后 LRU算法的优化**：
1. LRU时钟的粒度从秒级提升为毫秒级
    ```C++
    /* Macro used to obtain the current LRU clock.
     * If the current resolution is lower than the frequency we refresh the
     * LRU clock (as it should be in production servers) we return the
     * precomputed value, otherwise we need to resort to a system call. */
    #define LRU_CLOCK() ((1000/server.hz <= LRU_CLOCK_RESOLUTION) ? server.lruclock : getLRUClock())

    unsigned int getLRUClock(void) {
        return (mstime()/LRU_CLOCK_RESOLUTION) & LRU_CLOCK_MAX;
    }
    ```
    提升LRU时钟的粒度，主要是为了在测试LRU算法性能时，能够在更短的时间内获取结果，更新LRU时钟的方法也有所变化，如果LRU时钟的时间粒度高于serverCron刷新的时间粒度，那么就主动获取    最新的时间，否则使用server缓存的时间。

2. 使用新的API来获取LRU替换时的采样样本，在Redis 2.8中每次选取淘汰样本时，都是调用dictGetRandomKey来随机获取一个key，会根据maxmemory-samples配置的大小，多次调用。Redis 3.0中，会一次性取maxmemory-samples个Key

3. 默认的LRU采样样本数从3提升为5

4. 使用eviction pool来选取需要淘汰的key。这是**最重要**的改进。当选出需要淘汰的Key时，我们不再直接淘汰，而是把它加入一个池子中，然后淘汰掉池子中最长未使用的Key。样的选取淘汰key的方式的好处是：假设说新随机选取的key的访问时间可能比历史随机选取的key的访问时间还要新，但是在Redis 2.8中，新选取的key会被淘汰掉，这和LRU算法利用的访问局部性原理是相违背的，在Redis 3.0中，这种情况被避免了。

代码：
```C++
#define EVICTION_SAMPLES_ARRAY_SIZE 16
void evictionPoolPopulate(dict *sampledict, dict *keydict, struct evictionPoolEntry *pool) {
    int j, k, count;
    dictEntry *_samples[EVICTION_SAMPLES_ARRAY_SIZE];
    dictEntry **samples;

    /* Try to use a static buffer: this function is a big hit...
     * Note: it was actually measured that this helps. */
    if (server.maxmemory_samples <= EVICTION_SAMPLES_ARRAY_SIZE) {
        samples = _samples;
    } else {
        samples = zmalloc(sizeof(samples[0])*server.maxmemory_samples);
    }

    count = dictGetSomeKeys(sampledict,samples,server.maxmemory_samples);
    for (j = 0; j < count; j++) {
        unsigned long long idle;
        sds key;
        robj *o;
        dictEntry *de;

        de = samples[j];
        key = dictGetKey(de);
        /* If the dictionary we are sampling from is not the main
         * dictionary (but the expires one) we need to lookup the key
         * again in the key dictionary to obtain the value object. */
        if (sampledict != keydict) de = dictFind(keydict, key);
        o = dictGetVal(de);
        idle = estimateObjectIdleTime(o);

        /* Insert the element inside the pool.
         * First, find the first empty bucket or the first populated
         * bucket that has an idle time smaller than our idle time. */
        k = 0;
        while (k < MAXMEMORY_EVICTION_POOL_SIZE &&
               pool[k].key &&
               pool[k].idle < idle) k++;
        if (k == 0 && pool[MAXMEMORY_EVICTION_POOL_SIZE-1].key != NULL) {
            /* Can't insert if the element is < the worst element we have
             * and there are no empty buckets. */
            continue;
        } else if (k < MAXMEMORY_EVICTION_POOL_SIZE && pool[k].key == NULL) {
            /* Inserting into empty position. No setup needed before insert. */
        } else {
            /* Inserting in the middle. Now k points to the first element
             * greater than the element to insert.  */
            if (pool[MAXMEMORY_EVICTION_POOL_SIZE-1].key == NULL) {
                /* Free space on the right? Insert at k shifting
                 * all the elements from k to end to the right. */
                memmove(pool+k+1,pool+k,
                    sizeof(pool[0])*(MAXMEMORY_EVICTION_POOL_SIZE-k-1));
            } else {
                /* No free space on right? Insert at k-1 */
                k--;
                /* Shift all elements on the left of k (included) to the
                 * left, so we discard the element with smaller idle time. */
                sdsfree(pool[0].key);
                memmove(pool,pool+1,sizeof(pool[0])*k);
            }
        }
        pool[k].key = sdsdup(key);
        pool[k].idle = idle;
    }
    if (samples != _samples) zfree(samples);
}
```

维护一个eviction pool带来的少量开销情况下，Redis 3.0后不再会淘汰掉新加入的Key，并且LRU算法的精确度又提升了一个等级。
![LRU精确度提升](https://makefriends.bs2dl.yy.com/bm1576068317864.png)