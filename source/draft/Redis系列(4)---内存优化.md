# Redis系列（4）---内存与性能优化

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


## 1.Redis内存优化之-字符串SDS
Redis 中的字符串，没有使用C语音的字符串（'\0'结尾），而是自己实现了一种SDS结构。
```C++
struct sdshdr {
    // 记录 buf 数组中已使用字节的数量
    // 等于 SDS 所保存字符串的长度
    int len;
    // 记录 buf 数组中未使用字节的数量
    int free;
    // 字节数组，用于保存字符串
    char buf[];
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



