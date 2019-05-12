---
title: SSDB初步学习
date: 2018-4-16 14:14:10
tags:
 -DB
categories: DB
thumbnail: https://raw.githubusercontent.com/apollochen123/image/master/%E9%BB%98%E8%AE%A42.jpg
---

# SSDB简单了解和配置+java简单接入
-----
# Overview
一个高性能的支持丰富数据结构的 NoSQL 数据库, 用于替代 Redis.

-------------

### 安装与配置
因为是调研，我就在window上搭了一个主从模型。官方的强制建议是，window下，只适用于开发：
>强烈推荐你把 SSDB 部署在 Linux 操作系统上.不要在生产环境中使用 Windows 操作系统来运行 SSDB 服务器. 如果你确实必须使用 Windows 操作系统, 请在上面运行一个 Linux 虚拟机, 然后再让 SSDB 运行于这个虚拟机之中.

#### window下安装
1. clone ssdb window下[github仓库地址](https://github.com/ideawu/ssdb-bin)
2. cd 到 clone路径
3. D:\ssdb\ssdb-bin> ssdb-server-1.9.4.exe  ssdb.conf -s start

启动图：
![启动图](https://github.com/apollochen123/image/blob/master/SSDB/%E5%90%AF%E5%8A%A8%E5%9B%BE.png?raw=true)

#### linux下安装
参见：
http://ssdb.io/docs/zh_cn/install.html
#### 配置
SSDB的配置信息主要集中在其目录下 ssdb.conf 文件中
官方配置参考：http://ssdb.io/docs/zh_cn/config.html
主从，主主，多主配置方案：http://ssdb.io/docs/zh_cn/replication.html

**注意：** SSDB 的配置文件使用一个 TAB 来表示一级缩进, 不要使用空格来缩进, 无论你用1个, 2个, 3个, 4个, 5个, 6个, 7个, 8个或者无数个空格都不行!
```
# ssdb-server config
# MUST indent by TAB!
# relative to path of this file, directory must exists

# 这个目录下，有两个文件夹，一个是data，一个是meta.数据存放在var/data目录下
work_dir = ./var
pidfile = ./var/ssdb.pid

server:
    ip: 127.0.0.1
    port: 9002
    #只读模式
    #readonly: yes|no
	# bind to public ip
	#ip: 0.0.0.0

	#配置黑名单，白名单iptable
	# format: allow|deny: all|ip_prefix
	# multiple allows or denys is supported
	#deny: all
	#allow: 127.0.0.1
	#allow: 192.168

#使用主从，主主，多主的配置
replication:
	slaveof:
		# to identify a master even if it moved(ip, port changed)
		# if set to empty or not defined, ip:port will be used.
		#id: svc_2
		# sync|mirror, default is sync
		#type: sync
		#ip: 127.0.0.1
		#port: 8889

#日志输出配置
logger:
	level: debug
	output: log.txt
	rotate:
		size: 1000000000

leveldb:
	# in MB
	cache_size: 500
	# in KB
	block_size: 32
	# in MB
	write_buffer_size: 64
	# in MB
	compaction_speed: 1000
	# yes|no
	compression: no
```
### 主从配置：
主配置：不必更改
从配置: 只需修改replication下slaveof配置
    ```
    replication:
	   slaveof:
		# to identify a master even if it moved(ip, port changed)
		# if set to empty or not defined, ip:port will be used.
		id: svc_2
		# sync|mirror, default is sync
		type: sync
		#主库host
		ip: 127.0.0.1
		#主库ip
		port: 9001

    ```

### java使用ssdb4j原生接入
官方对java支持不怎么友好，简陋的java客户端驱动---在maven仓库里都没有库，官方给出说法：
![官方jar说法](https://github.com/apollochen123/image/blob/master/SSDB/%E5%AE%98%E6%96%B9javaMaven%E8%A7%A3%E7%AD%94.png?raw=true)

所以选择目前使用最多的，但是也很简陋（稍微好一点）的ssdb4j.maven依赖如下
```
<dependency>
  <groupId>org.nutz</groupId>
  <artifactId>ssdb4j</artifactId>
  <version>10.0</version>
</dependency>
```
java代码：
```java
package com.yy.apollo.demo;

import java.io.IOException;

import org.nutz.ssdb4j.SSDBs;
import org.nutz.ssdb4j.spi.Response;
import org.nutz.ssdb4j.spi.SSDB;

public class Simple {
    public static void main(String[] args) throws IOException {
        SSDB a = SSDBs.replication("127.0.0.1", 9001, "127.0.0.1", 9002, 10000, null);
        a.set("Key1116", "1116");
        Response response = a.get("Key1116");
        System.out.println(response.asString());
        a.close();

    }
}

```
一个简单的主写，从读模型。这里window的主从有个问题，等我set值过后，得重启从server才能拿到刚刚set到主的数据。可能是window下的问题。
结果：
```
1116
```


### java使用jedis接入
可以使用jedis接入，虽然还不知道具体兼容问题，但是大致能用.按照官方的使用方式，应该问题不大。
```java
package com.yy.apollo.demo;
import redis.clients.jedis.Jedis;

public class JedisTest {

    public static void main(String[] args) {
        Jedis jedis = new Jedis("127.0.0.1",9002,1000);
        jedis.set("Jedis1", "I am jedis");
        System.out.println(jedis.get("Jedis1"));
        jedis.close();
    }
}
```
结果：
```
I am jedis
```
----------
### 优势
1. 便宜，便宜，便宜。重要的事说3次
2. SSDB 拥有 Redis 的主要优点 - 高性能, 丰富数据结构, 并且拥有 Redis 所不具备的能力 - 大数据存储能力. SSDB 服务器的单机存储能力是 Redis 的 100 倍! 因为 SSDB 能将数据存储在硬盘中.
3. 持久化的队列服务

### 局限
1. 社区不够活跃。资料较少.
2. >ssdb的主从复制效率很低。binlog和数据是分开存储的，日志冗余较多，由于ssdb本身要在多线程条件下才能发挥出更好的性能，为了使多个线程在写入binlog时能保证操作顺序和原子性，ssdb的binlog数据结构上用了一把全局锁，可想而知，这里的锁竞争会很影响性能。
作者：李竟成
链接：https://www.zhihu.com/question/40733101/answer/270737849
来源：知乎
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
3. >前面回答里提到的zrevrange 和 zrevrangebyscore慢，而zrange 和 zrangebyscore 还能接受，其实就是说逆序遍历比顺序遍历慢得多，其根本原因就在于逆序遍历的时候，会多一个“记录头部”定位的过程，需要不断尝试去定位到两条记录的“分界点”，而顺序遍历的时候则不需要，因为读完一条记录直接就到了下一条记录的“分界点”，并且像rocksdb之类的存储引擎都会把数据长度保存在记录的元信息里，只需要按长度读取数据就可以了。redis则不存在类似问题，因为它是完全基于指针和偏移量在内存中进行寻址来读取数据的，寻址效率高了好多个数量级。
作者：李竟成
链接：https://www.zhihu.com/question/40733101/answer/270737849
来源：知乎
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
4. 不支持Sentinel模式
![不支持哨兵](https://github.com/apollochen123/image/blob/master/SSDB/%E4%B8%8D%E6%94%AF%E6%8C%81sentinel.png?raw=true)
5. 官方和ssdb4j的jar包，都没有提供容错机制，如主从切换。 ssdb4j支持java端主从访问，但是不能支持主主，多主方式访问。
6. 目前用于redis的sentinel可能无法使用。得研究新的接入方案。

### 关于性能
整理了一些别人性能的测试

https://www.cnblogs.com/smark/p/3909291.html
从测试结果看来差距还是非常明显,并不象官网那样说得这么理想.虽然SSDB效率上不如REDIS,但其基于磁盘存储有着其最大的优势,毕竟很多业务数据远超过服务器内存的容量.
SSDB的测试结果不理想也许是硬件环境受限,如果有个SSD硬盘的测试环境估计也得到一个更好的结果,但在测试过程中发现一个问题就是SSDB占用的CPU资源也是非常大的,在以上测试过程SSDB的并发效率比不上REDIS,同时CPU损耗上基本要比REDIS高出一倍的样子.
以上测试结果紧紧是是一些情况下的性能测试对比,不能完全表述出两者在应用的差距的结果,如果需要用到这些产品的同学不防在实施前进行一些测试为实施选择提供一个更可靠的结果.

https://www.cnblogs.com/python3-study/p/7244401.html
从官方数据看，SSDB的性能很突出，与Redis基本相当了，Redis是内存型，容量问题是弱项，并且内存成本太高，SSDB针对这个弱点，使用硬盘存储，使用Google高性能的存储引擎LevelDB，适合大数据量处理并把性能优化到Redis级别，具有Redis的数据结构、兼容Redis客户端，还给出了从Redis迁移到SSDB的方案。
那么接下来我在一台测试服务器上分别对Redis与SSDB做性能测试，但是结果是SSDB比Redis差了很多，与SSDB官网上显示的对比数据相差较大
预料到SSDB会弱于Redis，但没想到差这么多，可能是测试数量不同，或者是我的服务器硬件配置不利于SSDB等原因导致的

https://my.oschina.net/fuckphp/blog/270938
