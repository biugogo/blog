---
title: Redis系列(10)---Redis Cluster
date: 2019-12-21 23:23:11
tags:
 -Redis
categories: DB
thumbnail: /gallery/marvel/1557842931503.jpg
---


# Redis系列(10)---Redis Cluster

## Redis Cluster能带来什么
1. 高可用性和高达1000个节点的线性扩展能力，不使用代理，异步复制，数据无合并
2. 可接受的写入安全性。把数据丢失控制在一个可接受的时间窗口
3. 自动的故障转移能力，在大多数节点可用，并且故障节点拥有备份节点时。同时支持*replicas migration*：自己没有Slave时，向其他有多个Slave的节点借取Slave的能力。


## Redis Cluster端口
Redis Cluster的每个node需要占用两个端口。默认6379和16379（前一个节点加10000）.6379用于客户端通信，16379 bus端口用于集群内部通信（点对点二进制通信端口，用于失败检测，配置更新，故障转移等）。如果bus端口不可用，Redis Cluster无法进行故障转移等功能。


## 分片规则
Redis Cluster不是使用一致性hash，而是使用hash slot---哈希槽。
### 普通hash、consistent hash(一致性hash)和hash slot（哈希槽）差别
1. 普通hash：hash计算的值，跟你想要key被hash到的节点数量是相关想。当节点数量发生变化，同一key的hash值会发生变化。例子：把int64的uid分配到5台机器上，选择uid%5来分配，机器变化为6台时，hash算法变为uid%6。hash值发生变化。

2. 一致性hash：节点变化不会影响key的hash值。
    * 构建一个哈希环（hash ring）（2的32次方-1）
    * 计算缓存对象的哈希值放入
    * 计算缓存节点（ip或者主机名）的哈希值放入
    * 从缓存对象所在的位置顺时针查找最近的缓存节点位置
    * **缺点**： 一致性哈希算法无法满足平衡性，所以需要引入虚拟节点的概念，通过为缓存节点创建N个虚拟节点（例如为ip或者主机名加上%N后缀），使缓存对象更均匀的分布在哈希环上。
3. hash slot：先规划好slot个数，hash会分配到slot上，不同主机分配不同slot范围。通过预先规划和变动槽对应的缓存节点，就能满足单调性和平衡性。
    * 规划槽的数量（常见1024（codis）或者16383（Redis-cluster））
    * 每个槽对应一个可重复的缓存节点

### 具体算法
1. 对Key使用CRC16求校验码
2. slot  = 校验码 %16384来计算键key属于哪个slot
3. 查看slot属于哪个节点

### 为什么是16384

我们知道，Redis Cluster无时无刻都在相互发送心跳包。

* 每秒会随机选取5个节点，找出最久没有通信的节点发送ping消息
* 每100毫秒(1秒10次)都会扫描本地节点列表，如果发现节点最近一次接受pong消息的时间大于cluster-node-timeout/2 则立刻发送ping消息

心跳的频繁程度，间接的导致Redis Cluster的节点数量不会太多，一般不超过1000。否则，集群中会充斥着大量的心跳信息，占用大量带宽降低集群性能。
<br>
占用最大结构是心跳包中有一个char数组组成的bitmap结构：
![心跳包结构](https://makefriends.bs2dl.yy.com/bm1577605736141.png)

当bitmap被置为1时，表示节点持有这个槽。假如分配65536个槽，我们需要65536位来表示槽信息，65536位=8192b=8kb。我们按照常理基本只有1000个节点，不需要slot数量太大。
所以选择 16384位=2048B=2Kb的数据结构，同时CRC16计算的值也会均匀的分布在16384个槽中。所以选择16384是在网络带宽以及Redis Cluster节点数的双重背景下，选择的一个比较折中的值。


## 一致性问题
和Sentinal一样，Redis Cluster无法保证数据强一致性（写入数据不丢失）。原因有2：
1. Master-Slave同步带来的同步数据丢失
    * Client写入Master成功，返回OK
    * Master还未成功同步写入数据给Slave，Master挂掉
    * Slave成为新的Master，丢失刚刚写的数据

2. 网络分区孤立带来的集群脑裂数据丢失
    * 有A,B,C,A1,B1,C1 6个节点，3主3备结构的Redis Cluster集群
    * 发生网路分区不可通信，A,C,A1,B1,C1 在一个分区，B被隔离到另一个分区。
    * Client继续像B写入消息，同时收到OK
    * B1成为了新Master，写入B的消息丢失。



## Redis Cluster Bus
Redis Cluster的node功能包括以下几点
1. 保存使数据（废话）
2. 集群状态，包含key的应该映射到哪个节点上
3. 自动发现其他节点
4. 发现故障节点，发起故障转移，保证集群出现故障时继续工作

每个节点都通过一个TCP二进制数据端口完成，我们叫它**Redis Client Bus** .所有节点都相互连接，节点间通过**流言协议（gossip protocal）**传播信息。如发现新节点、ping、传播信息等。同时Bus还被用于在集群中PUB/SUB消息，并在用户请求时安排手动故障转移。
<br>

N个节点的Cluster结构，每个node都持有N-1个TCP接收和发送长连接。如果通信超时，将会尝试重新创建这个长连接。虽然N个节点组成了拓扑网络，但是node之间还是使用gossip协议和一个配置更新协议去避免交换太多的消息。所以信息交换不是指数的。



## Redis Cluster 性能
Redis Cluster不会代理访问Key所在的Node，而是通过让Client重定向到正确的node。这个信息会被Client缓存起来，以至于client总是能访问到正确的node。节点间数据不会也不能产生交互（意味着不能使用pipline，lua来操作在不同node上的Key）。但是这样的好处也意味这，你使用N个节点组成的Redis Cluster，将会获得和N个单独的Redis一样的性能。

## Redis Cluster的Hash tags
如果我们想要把某些KEY 分配到固定的一个slot上。可以使用{}包裹Key的部分，来hash到同一个slot。比如想把key:apollo1和apollo2hash到同一个slot。把Key{apollo}1,{apollo}2框起来即可。

## 集群添加删除节点
Redis Cluster支持动态的添加删除节点，并且自动进行slot重新分配。slot调整发生在以下情景：

1. 从其他节点分配一些slot到新添加的空节点
2. 删除节点时，把删除节点负责的slot分配给其他节点
3. 在节点间重平衡slot

## MOVED转向和ASK转向以及Slot迁移
1. MOVED转向：查找的槽不是由该节点处理的话，节点将查看自身内部所保存的哈希槽到节点ID的映射记录，并向客户端回复一个MOVED错误。
```
GET x
-MOVED 3999 127.0.0.1:6381
```
错误信息包含键x所属的哈希槽3999，以及负责处理这个槽的节点的IP和端口号127.0.0.1:6381
2. ASK转向：
当节点需要让一个客户端长期地（permanently）将针对某个槽的命令请求发送至另一个节点时，节点向客户端返回MOVED转向。当节点需要让客户端仅仅在下一个命令请求中转向至另一个节点时，节点向客户端返回ASK转向。

<br>
<br>
<br>
<br>

ASK转向发生在slot迁移过程中。当我们想将slot(8)从A节点迁移到B节点。我们的步骤是：
1. 向B节点发送命令：**CLUSTER SETSLOT 8 IMPORTING A** ，代表slot(8)节点将会从A节点迁入
2. 向A节点发送命令：**CLUSTER SETSLOT 8 MIGRATING B** ，代表slot(8)节点将会从本节点迁出至B
3. Client访问Slot(8)的数据时，如果要处理的键已经存在于A，那么这个命令继续被原节点A处理；如果键未存在于slot(8)（比如说，要向槽添加一个新的键）,那么A会回复一个ASK转向错误，让新的B节点处理。节点A不再创建关于Slot(8)的任何新键.
4. 后台进程将会把Slot(8)中的键，原子性的从A节点迁移到B节点。一个外部客户端的视角来看，在某个时间点上，键key要么存在于节点A，要么存在于节点B，但不会同时存在于节点 A和节点B。
5. 迁移完成后，Client再次访问A上slot(8)中的键，会收到MOVED转向，代表slot迁移完成。

## 心跳检测
Ping和pong包存在一个header信息，这个Header信息在所有其他的Requset都有（比如说投票请求）。
Header里面包括以下信息：
1. Node ID：一个160位的伪随机字符串，在第一次创建节点时分配该字符串，并且在Redis Cluster节点的整个生命周期中都保持不变。
2. 当前currentEpoch和configEpoch。如果节点是从属节点，则configEpoch是其主节点的最后一个已知configEpoch。
3. 节点标志，指示该节点是从节点，主节点还是其他单位节点信息。
4. bitmap ：当前节点持有slot信息，如果节点是从属节点，bitmap是其主节点的bitmap
5. 发送方TCP基本端口
6. 集群状态（以自己角度）
7. 如果是个从节点，会带上它的主节点ID
8. 一个流言（ gossip section）协议部分：包含本节点对于集群中随机一些其他节点的看法，这个随机节点是数量跟集群规模有关。 流言协议允许以发送者的角度对其他节点的看法在集群中传播。流言协议内存包含：
    * Node Id
    * 这个Node的IP和port
    * node的状态

   




## 失败检测
Redis 集群失效检测是用来识别出大多数节点何时无法访问某一个主节点或从节点。当这个事件发生时，就提升一个从节点来做主节点；若如果无法提升从节点来做主节点的话，那么整个集群就置为错误状态并停止接收客户端的查询。
<br>
和Sentinal的主观失败和客观失败一样。每个节点都有一份跟其他已知节点相关的标识列表。其中有两个标识是用于失效检测，分别是 PFAIL 和 FAIL。PFAIL 表示可能失效（Possible failure），这是一个非公认的（non acknowledged）失效类型。FAIL 表示一个节点已经失效，而且这个情况已经被大多数主节点在某段固定时间内确认过的了。

1. PFAIL:当一个节点在超过 NODE_TIMEOUT 时间后仍无法访问某个节点，那么它会用 PFAIL 来标识这个不可达的节点。无论节点类型是什么，主节点和从节点都能标识其他的节点为PFAIL。
2. FAIL: 心跳时，每个节点向其他每个节点发送的gossip消息中有包含一些随机的已知节点的状态，最终每个节点都能收到一份其他每个节点的节点标识。下面的条件满足的时候，会使用这个机制来让PFAIL状态升级为FAIL状态。
    * 节点A标记节点B为PFAIL
    * 节点A通过gossip协议收集到集群中其他大部分主节点标识的节点B的状态信息
    * 大部分主节点标记节点B为PFAIL状态，或者在NODE_TIMEOUT * FAIL_REPORT_VALIDITY_MULT这个时间内是处于PFAIL状态。

    以上所有条件都满足了，那么节点A会标记节点B为FAIL，向所有可达节点发送一个FAIL消息。FAIL消息强制接收到这消息的节点把节点B标记为FAIL状态。FAIL标识基本都是单向的，也就是说，一个节点能从PFAIL状态升级到FAIL状态，但要清除FAIL标识只有以下两种可能方法：
    * 节点已经恢复可达的，并且它是一个从节。在这种情况下，FAIL 标识可以清除掉，因为从节点并没有被故障转移
    * 节点已经恢复可达的，而且它是一个主节点，但经过了很长时间（N * NODE_TIMEOUT）后也没有检测到任何从节点被提升了。

## 故障转移
在Redis Cluster主节点投票帮助下，故障转移的过程都是由从节点处理的。当以下条件满足时，一个从节点可以发起选举：
1. 该从节点的主节点处于FAIL状态。
2. 这个主节点负责的哈希槽数目不为零。
3. 从节点和主节点之间的重复连接（replication link）断线不超过一段给定的时间，这是为了确保从节点的数据是可靠的
4. 一个从节点想要被推选出来，那么第一步应该是提高它的 currentEpoch 计数，并且向主节点们请求投票。

从节点通过广播一个**FAILOVER_AUTH_REQUEST**数据包给集群里的每个主节点来请求选票。然后等待回复（最多等 NODE_TIMEOUT 这么长时间）。一旦一个主节点给这个从节点投票，会回复一个**FAILOVER_AUTH_ACK**，并且在 NODE_TIMEOUT * 2 这段时间内不能再给同个主节点的其他从节点投票。在这段时间内它完全不能回复其他授权请求。
从节点会忽视所有带有的时期（epoch）参数比 currentEpoch 小的回应（ACKs），这样能避免把之前的投票的算为当前的合理投票。一旦某个从节点收到了大多数主节点的回应，那么它就赢得了选举。否则，如果无法在 NODE_TIMEOUT 时间内访问到大多数主节点，那么当前选举会被中断并在 NODE_TIMEOUT * 4 这段时间后由另一个从节点尝试发起选举。
<br>

从节点并不是在主节点一进入FAIL状态就马上尝试发起选举，而是有一点点延迟。延迟计算：
```
DELAY = 500 milliseconds + random delay between 0 and 500 milliseconds + SLAVE_RANK * 1000 milliseconds.
```
1. 500 milliseconds 延迟是因为：我们需要主节点FAIL状态在集群中通过流言协议传播开，避免其他主节点不为之投票
2. 500 milliseconds 随机时延是因为Raft协议惯例，避免同时发起投票
3. SLAVE_RANK是从服务器当前数据同步的版本，当Master挂掉后，此Master的所有Slave相互交换信息，进行一个排名，同步数据最全的Salve排名最前。

一旦有从节点赢得选举，它就会开始用ping和pong数据包向其他节点宣布自己已经是主节点，并提供它负责的哈希槽，设置 configEpoch 为 currentEpoch（选举开始时生成的）。其他节点会检测到有一个新的主节点（带着更大的configEpoch）在负责处理之前一个旧的主节点负责的哈希槽，然后就升级自己的配置信息。

主节点接收到来自于从节点、要求以**FAILOVER_AUTH_REQUEST**请求的形式投票的请求。 要授予一个投票，必须要满足以下条件：

1. 在一个给定的时段（epoch）里，一个主节点只能投一次票，并且拒绝给以前时段投票：每个主节点都有一个 lastVoteEpoch 域，一旦认证请求数据包（auth request packet）里的 currentEpoch 小于 lastVoteEpoch，那么主节点就会拒绝再次投票。当一个主节点积极响应一个投票请求，那么 lastVoteEpoch 会相应地进行更新。
2. 一个主节点投票给某个从节点当且仅当该从节点的主节点被标记为 FAIL。
3. 如果认证请求里的 currentEpoch 小于主节点里的 currentEpoch 的话，那么该请求会被忽视掉。因此，主节点的回应总是带着和认证请求一致的 currentEpoch。如果同一个从节点在增加 currentEpoch 后再次请求投票，那么保证一个来自于主节点的、旧的延迟回复不会被新一轮选举接受。











