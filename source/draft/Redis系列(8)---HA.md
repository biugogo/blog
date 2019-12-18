# Redis系列(8)---HA

## Overview

Redis官方推荐的HA（high availability）方案的Redis Sentinal。如果你使用的是Redis2.6(Sentinal版本为Sentinal 1)，你最好应该使用Redis2.8版本的Sentinal 2，因为Sentinal 1有很多的Bug，已经被官方弃用，所以强烈建议使用Redis2.8以及Sentinal 2。

## Sentinal的功能和注意事项

作为官方推荐的高可用方案，Redis Sentinal具有以下功能：
1. 监控：Sentinal会不断检查Master和Slave是否按预期工作。
2. 通知：能够通过提供的API，通知系统管理者监控的Redis实例出现问题
3. 自动故障转移：如果Master出现问题，Sentinal 能够开启故障转移流程，选举新的Master，并通知客户端Redis实例发生变化
4. 配置提供：Sentinal提供服务发现，客户端可以连接Sentinal获得实时更新的Redis节点信息。

当然使用Sentinal之前，你应该了解
1. Sentinal是一个集群方案，这意味着你需要部署3个以上Sentinal节点。
2. Sentinal应该部署在3个独立的机器上，避免硬件故障
3. Sentinal+Redis集群，因为异步主从同步的问题，不能保证在故障转移期间的数据写入不会失败。
4. Sentinal需要Client的支持（比如节点发现，接收节点变更信息等）
5. 上线前，请进行充足的测试，确保能正常进行故障监控和转移（废话）。

## Sentinal基础信息
### 一份简单的配置
```
# 
Sentinal monitor myMaster 127.0.0.1 6379 2
# myMaster主观下线超时时间
Sentinal down-after-milliseconds myMaster 60000
# 
Sentinal Failover-timeout myMaster 180000
Sentinal parallel-syncs myMaster 1
```
1. 第一行：Sentinal monitor \<Master-group-name> \<ip> \<port> \<quorum>
    * Master-group-name：监控的Master节点名称
    * ip:Master节点Ip
    * port:Master节点端口
    * quorum：至少有quorum个Sentinal主观的认为这个Master有故障，才会对这个Master进行下线以及故障转移。这里仅仅是开启检测，并不是执行。执行故障转移需要大于一半节点投票同意，才会执行故障转移。

2. 第二行: Sentinal \<down-after-milliseconds> \<MasterName> \<timeout>
    * Sentinal超过timeout这个时间都无法连通Master包括Slave（Slave不需要客观下线，因为不需要故障转移）的话，就会主观认为该Master已经下线（实际下线需要客观下线的判断通过才会下线）
3. 第三行：Sentinal Failover-timeout \<MasterName> \<timeout>
    * 同一个Sentinal对同一个Master两次Failover之间的间隔时间
    * 当一个Slave从一个错误的Master那里同步数据开始计算时间，直到Slave被纠正为向正确的Master那里同步数据时间。
    * 当想要取消一个正在进行的Failover所需要的时间。  
    * 当进行Failover时，配置所有Slaves指向新的Master所需的最大时间。不过，即使过了这个超时，Slaves依然会被正确配置为指向Master，但是就不按parallel-syncs所配置的规则来了。
4. 第四行：Sentinal parallel-syncs \<MasterName> \<cout>
    * 指定了在发生主备切换时最多可以有多少个Slave同时对新的Master进行同步。数字越小，同步时间越长。但是Slave在同步Master时会处于不可用状态。所以可以根据业务需要寻找一个合适的值。



### 主观下线和客观下线
1. **主观下线**: Subjectively Down，简称 SDOWN。如果单个Sentinal节点在down-after-milliseconds给定的毫秒数之内未能收到某Redis节点返回PONG命令（或者返回错误），那么这个Sentinal节点会认为该Redis节点主观下线。
2. **客观下线**：Objectively Down， 简称 ODOWN。当大于quorum个Sentinal对某个Redis节点做出主观下线判定，这个节点状态变为客观下线，并开启故障转移。

主观下线到客观下线的流程：

1. 当Sentinal监视的某个Redis节点主观下线后
2. 该Sentinal会询问其它该Redis节点的Sentinal，通过发送Sentinal is-master-down-by-addr ip port current_epoch runid命令询问其他节点是否同意下线。
    * ip：主观下线的服务id
    * port：主观下线的服务端口
    * current_epoch：Sentinal的纪元
    * runid：*表示检测服务下线状态，如果是Sentinal运行id，表示用来选举领头Sentinal
3. 其他Sentinal节点接收询问后，提取参数，根据ip和端口，回复包含三个参数:
    * down_state（1表示已下线，0表示未下线）
    * leader_runid（领头Sentinal id）
    * leader_epoch（领头Sentinal纪元）
4. 领头Sentinal接收到超过quorum个回复告诉它某个Master已经down掉了，那么它会判定该Redis节点客观下线ODOWN，这个流程是通过gossip（流言）协议完成的。

### Sentinal内部定时任务
1. 每10秒每个Sentinal会对Master和Slave执行info命令，这个任务达到两个目的：发现Slave节点和确认主从关系。
2. 每2秒每个Sentinal通过Master节点上发布订阅频道__sentinal__:hello进行信息交换(对节点的"看法"和自身的信息)，达成共识。
3. 每1秒每个Sentinal对其他Sentinal和Redis节点执行ping操作（相互监控），这个其实是一个心跳检测，是失败判定的依据。

### 节点自动发现机制和节点删除
1. **Sentinal各个节点**: Sentinal各个节点都相互通信，但是不需要繁琐的写入配置文件。Sentinal通过Master节点的发布/订阅机制去自动发现其它也监控了本Master的Sentinal节点（向名为__sentinal__:hello的管道中发送消息来实现）。
2. **Master和Slave**：同样也不需要配置某个Master的所有Slave的地址，Sentinal会通过询问Master来得到这些Slave的地址的。

每个Sentinal通过向每个Master和Slave的发布/订阅频道__sentinal__:hello每秒发送一次消息，来宣布它的存在。每个Sentinal也订阅了每个Master和Slave的频道__sentinal__:hello的内容，来发现未知的Sentinal，当检测到了新的Sentinal，则将其加入到自身维护的Master监控列表中。

#### 删除Sentinal
删除一个Sentinal显得有点复杂：因为Sentinal永远不会删除一个已经存在过的Sentinal，即使它已经与组织失去联系很久了。所以你需要：
1. 停止所要删除的Sentinal
2. 发送一个Sentinal RESET * 命令给所有其它的Sentinal实例，如果你想要重置指定Master上面的Sentinal，只需要把*号改为特定的名字。注意，需要一个接一个发，每次发送的间隔不低于30秒。
3. 检查一下所有的Sentinals是否都有一致的当前Sentinal数。使用Sentinal Master Mastername来查询。

#### 删除旧的Master或者不可达Slave
Sentinal永远会记录好一个Master的Slaves，即使Slave已经与组织失联好久了。这是很有用的，因为Sentinal集群必须有能力把一个恢复可用的Slave进行重新配置。想要永久地删除掉一个Slave(有可能它曾经是个Master)，你只需要发送一个Sentinal RESET Master命令给所有的Sentinals，它们将会更新列表里能够正确地复制Master数据的Slave。

## 故障转移流程
### 1.判定主观下线
* Sentinal会向Master发送Ping，收到的返回结果不是有效回复（+PONG、-LOADING、-MASTERDOWN），经过down-after-milliseconds时间内连续收到无效回复，便会标记对应Redis实例的flags属性为SRI_S_DOWN，认为其主观下线。



### 2. 客观下线
* 当一台Sentinal维护的flags属性变为SRI_S_DOWN时，由于在Sentinal集群中且每个Sentinal节点判断Master下线的时间间隔可能不一样，所以它必须要去询问其他Sentinal节点这台监督的Master节点是否下线。
    
* 通过遍历自己维护的Sentinals dict向其他的Sentinal节点发送命令：
    ```
    sentinal is-master-down-by-addr <Master-ip> <Master-port> <current_epoch> <leader_id>
    ```
    其中leader_id的参数在第一次询问客观下线时，默认 '*' 号。

* 命令返回
    ```
    <down_state> <leader_runid> <leader_epoch> 
    ```

    * down_state：Master的下线状态，0未下线，1已下线。当返回为已下线时，会同步更新flags的对应的第5位标志位**SRI_Master_DOWN**为1。
    * leader_runid：第一次发送询问时我们传的'\*' ,所以收到的回复也是'\*'
    * leader_epoch: 当前投票纪元，leader_runid为'\*'时，该值总为0

* 分析询问结果是否超过 **quorum** 个Sentinal判断Master已下线后，进入客观下线状态。flag状态**SRI_O_DOWN**变为1，进入选举Sentinal leader流程。

在Sentinal监督的Master由主观下线状态到客观下线的过程，从命令广播和判断Master客观下线这个共识，Sentinal并没有采用什么特殊的算法。客观下线这一状态是在通过命令
**Sentinal is-master-down-by-addr**不断交互慢慢达成的一个共识。**Sentinal is-master-down-by-addr**命令大量充斥在Sentinal的网络结构中，当某个Sentinal节点的客观条件得到满足时，选举故障转移的领头选举便也开始了。从整个集群看，当开始进入Leader选举状态时，集群中可能还有Sentinal节点并没有判断出该Master已经掉线。



### 3. Sentinal Leader选举
* 当Sentinel集群中的某个节点判定Master客观下线，开启Sentinal Leader选举流程。这里有一些前提：
    * Master必须满足客观下线
    * Master没有在故障转移中
    * Master是不是距离上次尝试故障转移时间间隔小于2倍故障转移超时

* 如果满足以上前提，设置Flag状态为**SRI_FAILOVER_IN_PROGRESS**；更新故障转移新纪元。设置Leader选举开始时间（**这里会加上一个随机数，避免同时发起成为leader询问，Raft协议标准**）

* 对外发送命令，寻求其他节点把自己设置为leader节点，这时候leader_id是自己
    ```
    sentinal is-master-down-by-addr <Master-ip> <Master-port> <current_epoch> <leader_id> 
    ```

* 接收到询问命令后有两种情况：
    * 接收节点在当前投票纪元中没有设置leader，便设置将其设置为leader
    * 接收节点在当前投票纪元中已设置了leader，便将已设置的leader返回

* 接收到回复后，发起询问的Sentinel节点会将leader runid存起来，以便统计票数。
通过一轮询问，询问的Sentinel节点就将会获得其他Sentinel的投票结果。

* 一轮询问后，选出票数最多的runid，投出自己的一票，如果有winer，就将该票投给winer，如果没有就把票投给自己。

* 当winer的选票大于所有节点的一半并且大于quorum时，该winner就是被选举出来的leader


整个拉票的过程是异步的，并且如果有节点掉线的话，或者票数最多的节点满足不了上述的要求的话，那么当前纪元时产生不了最终的leader的，只能等待超时，然后开启下一轮的新纪元，直到该次故障转移leader被选举出来。选票产生的结果在当前纪元内才有效。



### 4. 选择Slave

1. 选择不健康的Slave，以下状态的 Slave 是不健康的：
    * 主观下线的Slave
    * 大于等于5秒没有回复过Sentinel节点ping响应的Slave
    * 与Master失联超过down-after-milliseconds * 10秒的 Slave

2. 对健康的 Slave 进行排序
    * 选择 priority（从节点优先级，可配置，默认100）最高的从节点，如果有优先级相同的节点，进行下一步。
    * 选择复制偏移量最大的从节点（复制的最完整），如果有复制偏移量相等的节点，进行下一步。
    * 选择runid最小的从节点。

### 5. 通知选中的Slave成为Master
* 选则出合适的Slave后，向其发送**Slaveof no one**的命令将其升级为Master。
* Sentinel向选择的Slave发送的info命令来获知Slave的角色是否已被改变。
* 如果等待超时，选择的Slave INFO信息还没有改变，则宣告故障转移超时失败，重置故障转移，进入新一轮的投票选举。
* 成功转换为新Master时，会强制更新**hello msg**的pub周期，然后广播。


### 6. 通知其他Slave新的Master
* 向其他Slaves发送Slave OF \<new Master address>命令，复制新Master。
* 通过info命令的探测得知，每个Slave是否已重新配置了新的Master

故障转移结束。


## 关于Redis配置版本号
configuration epoch是当前Redis主从架构的配置版本号，无论是sentinel集群选举 leader还是进行故障转移的时候，要求各sentinel节点得到的configuration epoch都是相同的，sentinel is-master-down-by-addr 命令中就必须有当前配置版本号这个参数，在选举 eader过程中，如果本次选举失败，那么进行下一次选举，就会更新配置版本号，也就是说，每次选举都对应一个新的configuration epoch，在故障转移的过程中，也要求各个sentinel节点使用相同的 configuration epoch。

在故障转移成功之后，sentinel leader 会更新生成最新的Master配置，configuration epoch 也会更新，然后同步给其他的 sentinel 节点，这样保证 sentinel 集群中保存的 Master <-> Slave 配置都是最新的，当 client 请求的时候就会拿到最新的配置信息。

## Redis Sentinel 可能出现的问题以及解决办法
1. 问题1：异步复制导致的数据丢失

    因为Master -> Slave的复制是异步的，所以可能有部分数据还没复制到Slave，Master 就宕机了，此时这部分数据就丢失了。

2. 问题2：Redis服务脑裂导致的数据丢失

    Master所在机器突然网络故障，跟其他Slave机器不能连接，但是实际上Master还运行着。此时哨兵可能就会认为Master宕机了，然后开启选举，将其他Slave切换成了Master，这个时候，集群里就会有两个Master，也就是所谓的脑裂。此时虽然某个 Slave被切换成了Master，但是client还没来得及切换到新的Master，还继续写向旧 Master的数据就丢失了。因为旧Master再次恢复的时候，会被作为一个Slave挂到新的Master上去，自己的数据会清空，重新从新的Master复制数据。

通过以下两个配置可以减少数据丢失
```
min-slaves-to-write <number of Slaves>
min-slaves-max-lag <number of seconds>
```
1. **min-slaves-to-write**: Master必须至少有一个Slave在进行正常复制，否则就拒绝写请求，此时Master丧失可用性。
2. **min-slaves-max-lag**: Slave在少于min-slaves-max-lag秒内需要回复。

两个条件共同作用至少有min-slaves-to-write个从服务器， 并且这些服务器的延迟值都少于min-slaves-max-lag秒， 那么主服务器就会执行客户端请求的写操作。尽管不能保证写操作的持久性，但起码丢失数据的窗口会被严格限制在指定的秒数中。