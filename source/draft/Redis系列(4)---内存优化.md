# Redis系列（4）---内存优化

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


# Redis数据结构--SDS
