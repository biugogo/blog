---
title: RabbitMQ Highly Available
date: 2019-05-25 17:51:56
tags:
 -Rabbit
categories: Framework
thumbnail: /gallery/doraemon/1557842807990.jpg
---

# RabbitMQ高可用队列
------

## Overview 

本文主要翻译[官方文档](https://www.rabbitmq.com/ha.html#unsynchronised-Mirrors)作为记录理解。

1. Mirrored Queue是什么
2. 如何配置Mirrored Queue
3. Mirrored Queue Master节点选择和数据迁移
4. Mirrored Queue故障迁移细节

----

###  Mirrored Queue是什么

默认情况下，Msssage Queue不像Exchange和Banding那样会复制到所有节点，RabbitMQ Cluster的队列是只会在一个节点上的（队列在哪个声明就会在哪个节点上）。但是可以通过配置Mirrored Queue让Queue在多个节点上形成镜像。

每组Mirrored Queue由一个Master和一个或多个Mirror（镜像）。每个Queue都有自己的Matser。所有对Queue的操作都是首先应用在Matser上，再传播到Mirror。这些操作包括Msssage的发布和传递，消费者的Ack回包等。

Mirrored Queue是在cluster模式下使用的，所以尽量保证所有的节点延迟很小最好是在一个局域网当中。

当Msssage被发布到Mirrored Queue时，会被复制到所有的Mirrors。消费者不管他们连接到哪个节点都会连接到Master，当消费者与Master Ack后，Mirror就会删除这条Msssage。Mirrored Queue只会提高Queue的高可用，但是不会分配负载。

如果Master挂掉了,最老的Mirror如果是同步的，就会被选举成新的Master，如果它不是同步的，它也可以成为Master，这个取决于队列配置。

**Mirrored Queue和non-Mirrored的主要差别在于，如果Master节点不可用， non-Mirrored的表现取决于他的持久化配置，持久化的non-Mirrored Queue会在节点重启后继续可用，在挂掉时间内，Queue接受到message会打印一条log，非持久化的non-Mirrored将永远消失。Mirrored Queue在节点全部挂掉前，都可以进行故障迁移。**

non-Mirrored日志example
```
operation queue.declare caused a channel exception not_found: home node 'rabbit@hostname' of durable queue 'queue-name' in v
```

### 如何配置Mirroring

Mirrored Queue配置是通过配置policies实现的。当一个Queue满足多个policy时，会选择优先级最大的policy。Policies可以在任何时间更改。你可以先创建一个没有Mirrored的Queue然后再通过Policies让它变成Mirrored的。

非Mirrored Queus和没有Mirror的MirroredQueue的差别在于，非Mirrored Queus因为不需要镜像组件，可以有更高的Msssage吞吐量。

#### Mirrored Queus三种类型
1. **exactly**：固定数量节点策略    
设置ha-mode=axactly同时，需要设置ha-params=count。count就是节点数量，如果count=1，代表只有一个Master的Mirrored Queus；如果count=2代表1个Master和1个Mirror。如果这里这里节点数不够count，会在所有的节点上创建Queue。如果已经有超过count的Mirror Queue存在，会删除多余节点上的Queue。     
*注意使用ha-promote-on-shutdown=always在exactly模式下会有危险，因为Queue会跨节点迁移，使之变得不同步*
![img](/gallery/rabbitMQ/haExactly.png)
![img](/gallery/rabbitMQ/haExactlyDetails.png)     
    
   * Pattern:匹配正则表达式
   * Apply to:匹配队列还是交换机，这里是匹配队列
   * Definition:配置详情，这里意思是固定数量的Mirrored Queus，队列最大长度500000。Master节点选择策略是最轻负载策略。

2. **all**：全体节点策略
设置ha-mode=all，Queue会复制到全部节点，如果一个新的节点加入到集群，Queue也会自动复制到这个节点上。这个设置是非常保守的，如果不是特别重要的数据，没有必要这样，这会增加整个集群的负载，包括网络IO和磁盘IO等。通常建议复制到N/2 + 1个节点即可，N代表集群总节点数。
![img](/gallery/rabbitMQ/haAll.png)

3. **nodes**：指定名称节点
设置ha-mode=nodes同时需要配置ha-params=*node names*。node name即rabbit@hostname形式的状态。如果集群中没有node name，不会有Mirror生成，但是一旦指定name的node接入集群，Queue会自动在接入节点上创建。

#### 多少个Mirror合理？
首先，配置ha-mode=all是十分保守的，会加重整个集群负载，增加不必要的网络消耗和磁盘消耗，通常建议
```
mirro数=N/2+1个 //N代表节点总数
```
当然，如果有些数据是瞬态的，并且时效性很强的，设置更少的Mirror或者不设置Mirror也是可以的。

------------

### Mirrored Queue Master节点选择和数据迁移

#### Master节点选择策略
为了保证message FIFO的特性，每个Queue都有一个Master，所有Queue的操作都是先对Master生效，然后传递到其他Mirror。Queue的Master节点选取有三种策略。
1. min-Masters：负载最低节点策略
2. client-local：客户端连接哪个节点就哪个策略
3. random：随机

一般选取min-Masters策略,在管理网页配置
```
Queue-Master-locator=min-Masters即可
```
#### Master节点迁移
当我们更改Master选择策略的时候，可能会造成节点迁移（当现主节点不满足新的策略时）。为了保证不丢失Msssage，rabbitMQ的旧Master会持续服务，直到有一个新的Mirror完全同步。这个过程一旦开始，就像会像Matser节点挂掉，从新选主的的过程，consumer会和Master断开连接，进行重连。

*比如Queue有A主，B从两个节点，现在策略指定Queue迁移至C,D两个节点，中间过程可能会有ACD三个节点，当CD中有某个同步完成，A节点会自动下线。最后留下CD俩个节点，同步完成*。


----

### Mirrored Queue故障迁移细节
上文我们已经知道，每个Mirrored Queue都有一个Master和一个或几个Mirror，在不同的节点上，所有的操作都是在Master上进行的，再由Master广播给Mirror，包括Msssage消费也是从Master节点消费的。

#### Mirror挂掉
一旦Mirror挂掉，出了一些记录外，不会发生任何事，Master依旧是Master，client也不会有任何动作，不会被告知失败。当然Mirror的失败也不会被立即检测到（因为心跳间隔的缘故）

#### Master挂掉
1. 运行时间最长的Mirror将被提拔成Master，这里假设这个运行时间最长的Mirror是完全同步的和Master。如果所有Mirror都不是同步的，只存在Master的这部分Msssage将会丢失。
2. 被选举的这个Mirror会对所有已经发送，但是还没有ACK的Msssage重新排队，这里包括客户端已经发出的ACK Msssage，但是这个ACK可能在到达Master前或者Master同步给Mirror时被丢包了。因此，被选举成新Master的Mirror不得不从新发送所有没有ack的Msssage。
3. Consumers that have requested to be notified when a queue fails over will be notified of cancellation.(原文，不是很理解)
4. 因为Mirror重新发送没有ACK的所有Msssage，client将会收到重复的Msssage。
5. 在Mirror被选举成Master的选举过程，不会发生Msssage丢失。Client发送Message到连接的节点会被路由到Master，然后复制到所有节点，就算Master失败，这些Msssage还是会被发送到其他Mirror。一旦Mirror升级为新Master,这条Msssage已经在新Master Queue中了。
6. 如果Mirrored Queue设置了noAck=true，Msssage会发生丢失（就算Master节点没有发生故障，也会丢）
7. Mirrored Queues也支持publisher confirms模式。message只有被所有Mirror确认后才会被认为成功发布。
8. Mirrored Queues也支持transactions模式，只有当所有Mirror确认了transaction，clinet才会收到tx.commit-ok的回包。

















