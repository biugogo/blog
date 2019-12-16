# Redis系列(8)---HA

## Overview

Redis官方推荐的HA（high availability）方案的Redis Sentinal。如果你使用的是redis2.6(Sentinal版本为Sentinal 1)，你最好应该使用redis2.8版本的Sentinal 2，因为Sentinal 1有很多的Bug，已经被官方弃用，所以强烈建议使用redis2.8以及Sentinal 2。

## Sentinal功能和注意事项

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
2. 该Sentinal会询问其它该Redis节点的Sentinal，通过发送Sentinal is-Master-down-by-addr ip port current_epoch runid命令询问其他节点是否同意下线。
    * ip：主观下线的服务id
    * port：主观下线的服务端口
    * current_epoch：Sentinal的纪元
    * runid：表示用来选举领头Sentinal
3. 其他Sentinal节点接收询问后，提取参数，根据ip和端口，回复包含三个参数:
    * down_state（1表示已下线，0表示未下线）
    * leader_runid（领头Sentinal id）
    * leader_epoch（领头Sentinal纪元）
4. 领头Sentinal接收到超过quorum个回复告诉它某个Master已经down掉了，那么它会判定该Redis节点客观下线ODOWN，这个流程是通过gossip（流言）协议完成的。

### sentinal内部定时任务
1. 每10秒每个Sentinal会对Master和Slave执行info命令，这个任务达到两个目的：发现Slave节点和确认主从关系。
2. 每2秒每个Sentinal通过Master节点上发布订阅频道__Sentinel__:hello进行信息交换(对节点的"看法"和自身的信息)，达成共识。
3. 每1秒每个Sentinal对其他Sentinal和redis节点执行ping操作（相互监控），这个其实是一个心跳检测，是失败判定的依据。

### 节点自动发现机制和节点删除
1. **Sentinal各个节点**: Sentinal各个节点都相互通信，但是不需要繁琐的写入配置文件。Sentinal通过Master节点的发布/订阅机制去自动发现其它也监控了本Master的Sentinel节点（向名为__Sentinel__:hello的管道中发送消息来实现）。
2. **Master和Slave**：同样也不需要配置某个Master的所有Slave的地址，Sentinel会通过询问Master来得到这些Slave的地址的。

每个Sentinel通过向每个Master和Slave的发布/订阅频道__Sentinel__:hello每秒发送一次消息，来宣布它的存在。每个Sentinel也订阅了每个Master和Slave的频道__Sentinel__:hello的内容，来发现未知的Sentinel，当检测到了新的Sentinel，则将其加入到自身维护的Master监控列表中。

#### 删除Sentinal
删除一个Sentinel显得有点复杂：因为Sentinel永远不会删除一个已经存在过的Sentinel，即使它已经与组织失去联系很久了。所以你需要：
1. 停止所要删除的Sentinel
2. 发送一个Sentinel RESET * 命令给所有其它的Sentinel实例，如果你想要重置指定Master上面的Sentinel，只需要把*号改为特定的名字。注意，需要一个接一个发，每次发送的间隔不低于30秒。
3. 检查一下所有的Sentinels是否都有一致的当前Sentinel数。使用Sentinel Master Mastername来查询。

#### 删除旧的Master或者不可达Slave
sentinel永远会记录好一个Master的Slaves，即使Slave已经与组织失联好久了。这是很有用的，因为sentinel集群必须有能力把一个恢复可用的Slave进行重新配置。想要永久地删除掉一个Slave(有可能它曾经是个Master)，你只需要发送一个SENTINEL RESET Master命令给所有的sentinels，它们将会更新列表里能够正确地复制Master数据的Slave。

## 故障转移流程


### 主观下线
### 客观下线
### 主节点选举

### 通知发布

### Slave复制
### 配置持久化

## 内部算法