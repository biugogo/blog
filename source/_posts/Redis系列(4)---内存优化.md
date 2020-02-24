---
title: Redis系列（4）---内存与性能优化
date: 2019-12-01 20:33:10
tags:
 -Redis
categories: DB
thumbnail: /gallery/marvel/1557842931503.jpg
---

# Redis系列（4）---内存与性能优化

## Overview
1. Redis内存优化之-字符串SDS
2. Redis内存优化之-压缩列表ziplist
3. Redis内存优化之-跳表skiplist
4. Redis内存优化之-字典
5. 特殊内存编码阈值

## 1.Redis内存优化之-字符串SDS
Redis 中的字符串，没有使用C语音的字符串（'\0'结尾），而是自己实现了一种SDS结构。
```C++
struct sdshdr {
    int len;// 记录 buf 数组中已使用字节的数量,等于 SDS 所保存字符串的长度
    int free;//记录 buf 数组中未使用字节的数量
    char buf[]; // 字节数组，用于保存字符串
};
```
![sdshdr结构图](https://makefriends.bs2dl.yy.com/bm1575623104345.png)

buf属性是一个char类型的数组,SDS遵循C字符串以空字符结尾的惯例，保存空字符的1字节空间不计算在SDS的len属性里面，并且为空字符分配额外的1字节空间，这一惯例的好处是，SDS可以直接重用一部分 C字符串函数库里面的函数。

### SDS优势

1. **常数复杂度获取字符串长度**：C字符串并不记录自身的长度信息，获取长度需要重头到尾遍历字符串。SDS直接调用len即可。
2. **杜绝缓冲区溢出**：C字符串长度不会改变，SDS如果发现空间不足，不会产生溢出，而是自动扩容。
3. **减少修改字符串时带来的内存重分配次数**： C字符串的长度和底层数组的长度之间存在着这种关联性，所以每次增长或者缩短一个C字符串，程序都总要对保存这个C字符串的数组进行一次内存重分配操作。为了避免C字符串的这种缺陷，SDS使用**空间预分配**和**惰性空间释放**来改进这种频繁的内存重分配
    * 空间预分配：一种优化字符串增长的方式，每次空间扩展时，会额外分配一些内存空间。分配策略有两种**SDS的len小于1MB**和**SDS的len大于1MB**。在**SDS的len小于1MB**时，预留扩充后多一倍的内存;**SDS的len大于1MB**时，额外分配一1MB。举个例子：一个字符串，如果扩充后len等于10KB，那么SDS会额外分配10KB给free；如果扩充后len等于2MB，那么SDS会额外分配1MB的free给SDS
    * 惰性空间释放：当SDS的API需要缩短SDS保存的字符串时，程序并不立即使用内存重分配来回收缩短后多出来的字节， 而是使用free属性将这些字节的数量记录起来，并等待将来使用。
4. **二进制安全**：C字符串中的字符必须符合某种编码（比如ASCII），并且除了字符串的末尾之外，字符串里面不能包含空字符，所以C字符串不能保存像图片、音频、视频、压缩文件这样的二进制数据。SDS没有这些限制。
5. **兼容部分C字符串函数**：遵循C字符串以空字符结尾的惯例。




## 2.Redis内存优化之-压缩列表ziplist

Redis的压缩列表，节约内存而开发的， 由一系列特殊编码的连续内存块组成的顺序型（sequential）数据结构。在hash，list，zset的实现中都使用了压缩列表来减少内存开销。
详细学习压缩列表可以参考 [redis设计与实现-压缩列表](http://redisbook.com/preview/ziplist/list.html).

![压缩列表结构](https://makefriends.bs2dl.yy.com/bm1575782231110.png)

压缩列表entry的结构，详细可以参考[redis设计与实现-压缩列表结点](http://redisbook.com/preview/ziplist/node.html).
![压缩列表node结构](https://makefriends.bs2dl.yy.com/bm1575782724100.png)

1. 节点的previous_entry_length属性以字节为单位，记录了压缩列表中前一个节点的长度。
2. 节点的encoding属性记录了节点的content属性所保存数据的类型以及长度。
3. 节点的content属性负责保存节点的值，节点值可以是一个字节数组或者整数，值的类型和长度由节点的encoding属性决定

#### **压缩列表的优势**：
1. 节约内存：作为一种特殊的连续内存编码，压缩列表对内存的使用控制到了位的级别。
2. 降低内存碎片：因为它的连续，可以有效降低内存碎片，提升内存利用效率。

#### **压缩列表的劣势**：
没有绝对好的数据结构，压缩列表也不是毫无缺点。
1. O(n)的CRUD是时间复杂度，意味着压缩列表不能太长。在一定N内的时间复杂度我们可以容忍，所以我们需要在时间复杂度（CPU）和内存开销中找到一个平衡点（这也是redis开放一系列ziplist参数的原因。）
2. 连锁更新（cascade update）：在特殊情况下，可能触发连续多次空间扩展操作。细节参考 [redis设计与实现-连锁更新](http://redisbook.com/preview/ziplist/node.html).

## 3.Redis内存优化之-跳表skiplist
跳表是以空间换时间的数据结构。关于跳表的学习可以参考[这位大佬的博客](https://juejin.im/post/5d90e4a15188252d3a6a60b8)。这篇博客比CSDN上面一堆抄来抄去的博客好太多，通俗易懂。
![跳表](https://makefriends.bs2dl.yy.com/bm1575786723023.JPG)

观察这个结构，很容易看得出，跳表就是为有序链表加N层索引。跳表可以很好的满足redis的Sort Set的需求，log(n)的查询，添加，删除复杂度，以及m*log(n)的range操作。

## 4.Redis内存优化之-字典
Redis的字典和各种编程语言中的字典设计是类似。但是与众不同的是，redis的每个字典，包含了两个hash表。
两个hash表，是为了redis在hash表长度非常大，需要做渐进式rehash而设计的。
```C++
//字典结构
typedef struct dict {
    dictType *type;// 类型特定函数
    void *privdata;// 私有数据
    dictht ht[2];// 哈希表，两个
    int rehashidx; // rehash索引,当rehash不在进行时，值为 -1
} dict;
```
为什么需要两个hash表，是因为：如果ht[0]里只保存着四个键值对，那么服务器可以在瞬间就将这些键值对全部rehash到ht[1]；但是，如果哈希表里保存的键值对数量不是四个，而是四百万、四千万甚至四亿个键值对，那么要一次性将这些键值对全部rehash到ht[1]的话，由于庞大的计算量可能会导致服务器在一段时间内停止服务。为了避免rehash对服务器性能造成影响，服务器不是一次性将ht[0]里面的所有键值对全部rehash到ht[1]，而是分多次、渐进式地将ht[0]里面的键值对慢慢地rehash到ht[1]。

以下是哈希表渐进式rehash的详细步骤：
1. 为ht[1]分配空间，让字典同时持有ht[0]和ht[1]两个哈希表。
2. 在字典中维持一个索引计数器变量rehashidx，并将它的值设置为0，表示rehash工作正式开始。
3. 在rehash进行期间，每次对字典执行添加、删除、查找或者更新操作时，程序除了执行指定的操作以外，还会顺带将ht[0]哈希表在rehashidx索引上的所有键值对rehash到ht[1]，当rehash工作完成之后，程序将rehashidx属性的值增一。
4. 随着字典操作的不断执行，最终在某个时间点上，ht[0]的所有键值对都会被rehash至ht[1]，这时程序将rehashidx属性的值设为-1，表示rehash操作已完成。

渐进式rehash的好处在于它采取分而治之的方式，将rehash键值对所需的计算工作均滩到对字典的每个添加、删除、查找和更新操作上，从而避免了集中式rehash而带来的庞大计算量。






## 5.特殊内存编码阈值

从Redis 2.2开始，Redis开始使用一些特殊的数据结构以达到节约内存的目的。这个数值在理想情况下，可以提升5-10倍的内存利用效率。付出的代价很小，仅仅是增加一点CPU的开销。（一种以时间换空间的做法）
<br>
作为使用者，这些结构都是透明的，你需要做的只是根据你的CPU好坏，调整一些 redis.conf 中的参数值：

```js
hash-max-zipmap-entries 512 (hash-max-ziplist-entries for Redis >= 2.6)
hash-max-zipmap-value 64  (hash-max-ziplist-value for Redis >= 2.6)
list-max-ziplist-entries 512
list-max-ziplist-value 64
zset-max-ziplist-entries 128
zset-max-ziplist-value 64
set-max-intset-entries 512
```
#### 5.1 关于Lists

1. **list-max-ziplist-entries** 512

    如果Lists中，元素的数量少于512时，Lists会采用ziplist实现。当元素数量大于512时，会转变为linked list

2. **list-max-ziplist-value** 64

    如果Lists中，元素的最大长度小于64时，Lists会采用ziplist实现。当元素数量大于512时，会转变为linked list.

#### 5.2 关于zset
1. **zset-max-ziplist-entries** 128

    如果Sort Set中，元素的数量少于512时，Sort Set会采用ziplist实现。当元素数量大于512时，会转变为跳表（Skip List）结构

2. **zset-max-ziplist-value** 64

    如果Sort Set中，元素的最大长度小于64时，Sort Set会采用ziplist实现。当元素数量大于512时，会转变为跳表（Skip List）结构.

#### 5.3 关于Set
1. **set-max-intset-entries** 128
    当set中，元素的类型全是int类型时，并且长度小于128，我们使用intset存储.
    整数集合是Redis用于保存整数值的集合抽象数据结构，它可以保存类型为int16_t、int32_t或者int64_t的整数值，并且保证集合中不会出现重复元素。
    ```C++
    typedef struct intset {
        uint32_t encoding;// 编码方式INTSET_ENC_INT16，INTSET_ENC_INT32，INTSET_ENC_INT64
        uint32_t length; // 集合包含的元素数量
        int8_t contents[];// 保存元素的数组
    } intset;
    ```
    intset是有升级操作的，当一个intset中全是int16时，intset使用int16来装元素，此时插入一个大于int16 Max的整数时，整个集合中的元素会升级为int32。同理，对于int64也是这样。
    <br>
    因为每次向整数集合添加新元素都可能会引起升级，而每次升级都需要对底层数组中已有的所有元素进行类型转换，所以向整数集合添加新元素的时间复杂度为 O(N) 。
    <br>
    整数集合不支持降级操作，一旦对数组进行了升级，编码就会一直保持升级后的状态。

#### 5.4 关于Hash
1. **hash-max-zipmap-entries** 512 (hash-max-ziplist-entries for Redis >= 2.6)
    当redis的hash元素数量小于512时，我们使用ziplist去代替真正的hash。

2. **hash-max-zipmap-value** 64 (hash-max-ziplist-value for Redis >= 2.6)
    当hash的各字段长度都小于64时，使用ziplist代替真正的hash

当使用ziplist带代替hash时，没添加一组kv，会想ziplist中插入连续两项。
<br>

当我们想使用hash存储1000个用户的姓名，地址，电话时时，设想两种存储结构：
1. 用3个hash分别存储姓名，地址，电话。
2. 1000个hash，每个hash存储姓名，地址，电话。
<br>

两种结构都可以达到我们的目的，但是耗费的内存却是不同的，使用方案2能够有效减少内存使用。因为1000个hash全部都是使用ziplist实现的。这个内存节约量大约是5-10倍的量（根据redis官方给出的数据以及一个example的实验结果，可以参考:[官方文档](https://redis.io/topics/memory-optimization)）.




