# Redis系列(8)---HA

## Overview

Redis官方推荐的HA（high availability）方案的Redis Sentinal。如果你使用的是redis2.6(sentinel版本为sentinel 1)，你最好应该使用redis2.8版本的sentinel 2，因为sentinel 1有很多的Bug，已经被官方弃用，所以强烈建议使用redis2.8以及sentinel 2。

## sentinal功能和注意事项

作为官方推荐的高可用方案，Redis Sentinal具有以下功能：
1. 监控：Sentinel会不断检查Master和Slave是否按预期工作。
2. 通知：能够通过提供的API，通知系统管理者监控的Redis实例出现问题
3. 自动故障转移：如果Master出现问题，Sentinal 能够开启故障转移流程，选举新的Master，并通知客户端Redis实例发生变化
4. 配置提供：Sentinal提供服务发现，客户端可以连接Sentinal获得实时更新的Redis节点信息。

当然使用Sentinal之前，你应该了解
1. Sentinal是一个集群方案，这意味着你需要部署3个以上Sentinal节点。
2. Sentinal应该部署在3个独立的机器上，避免硬件故障
3. Sentinal+Redis集群，因为异步主从同步的问题，不能保证在故障转移期间的数据写入不会失败。
4. Sentinal需要Client的支持（比如节点发现，接收节点变更信息等）
5. 上线前，请进行充足的测试，确保能正常进行故障监控和转移（废话）。

## XXX
### 一份简单的配置
```
# 
sentinel monitor mymaster 127.0.0.1 6379 2
# mymaster主观下线超时时间
sentinel down-after-milliseconds mymaster 60000
# 
sentinel Failover-timeout mymaster 180000
sentinel parallel-syncs mymaster 1
```
1. 第一行：sentinel monitor \<master-group-name> \<ip> \<port> \<quorum>
    * master-group-name：监控的Master节点名称
    * ip:Master节点Ip
    * port:Master节点端口
    * quorum：至少有quorum个sentinel主观的认为这个master有故障，才会对这个master进行下线以及故障转移。这里仅仅是开启检测，并不是执行。执行故障转移需要大于一半节点投票同意，才会执行故障转移。

2. 第二行: sentinel \<down-after-milliseconds> \<masterName> \<timeout>
    * sentinel超过timeout这个时间都无法连通master包括Slave（Slave不需要客观下线，因为不需要故障转移）的话，就会主观认为该master已经下线（实际下线需要客观下线的判断通过才会下线）
3. 第三行：sentinel Failover-timeout \<masterName> \<timeout>
    * 同一个sentinel对同一个master两次Failover之间的间隔时间
    * 当一个Slave从一个错误的master那里同步数据开始计算时间，直到Slave被纠正为向正确的master那里同步数据时间。
    * 当想要取消一个正在进行的Failover所需要的时间。  
    * 当进行Failover时，配置所有Slaves指向新的master所需的最大时间。不过，即使过了这个超时，Slaves依然会被正确配置为指向master，但是就不按parallel-syncs所配置的规则来了。
4. 第四行：sentinel parallel-syncs \<masterName> \<cout>
    * 指定了在发生主备切换时最多可以有多少个Slave同时对新的master进行同步。数字越小，同步时间越长。但是Slave在同步Master时会处于不可用状态。所以可以根据业务需要寻找一个合适的值。



### 主观下线和客观下线
Sentinal






### 节自动发现和删除

### 删除旧的master和slave




## 故障转移流程

### 主观下线
### 客观下线
### 主节点选举

### 通知发布

### slave复制
### 配置持久化

## 内部算法