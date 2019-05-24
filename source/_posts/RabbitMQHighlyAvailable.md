# RabbitMQ高可用队列
------

## Overview 

本文主要翻译[官方文档](https://www.rabbitmq.com/ha.html#unsynchronised-mirrors)作为记录理解。

1. 什么是Mirrored Queue和Mirrored Queue如何工作 
2. 如何配置Mirrored Queue
3. Mirrored Queue数据地址和数据迁移
4. Master选举和同步
5. 高可用和普通队列在节点挂掉后行为

----

###  Mirrored Queue是什么

默认情况下，消息Queue不像Exchange和banding那样会复制到所有节点，RabbitMQ cluster的队列是只会在一个节点上的（队列在哪个声明就会在哪个节点上）。但是可以通过配置Mirrored Queue让Queue在多个节点上形成镜像。

每组Mirrored Queue由一个master和大于一个mirror（镜像）。每个Queue都有自己的master。所有对Queue的操作都是首先应用在matser上，再传播到mirror。这些操作包括消息的发布和传递，消费者的Ack回包等。

Mirrored Queue是在cluster模式下使用的，所以尽量保证所有的节点延迟很小最好是在一个局域网当中。

当消息被发布到Mirrored Queue时，会被复制到所有的mirrors。消费者不管他们连接到哪个节点都会连接到master，当消费者与master ack后，mirror就好删除这条消息。Mirrored Queue只会提高Queue的高可用，但是不会分配负载。

如果master挂掉了,最老的mirror如果是同步的，就会被选举成新的master，如果它不是同步的，它也可以成为master，这个取决于队列配置。

### 如何配置Mirroring

Mirrored Queue配置是通过配置policies实现的。当一个Queue满足多个policy时，会选择优先级最大的policy。Policies可以在任何时间更改。你可以先创建一个没有Mirrored的queue然后再通过Policies让它变成Mirrored的。

非Mirrored Queus和没有Mirror的MirroredQueue的差别在于，非Mirrored Queus因为不需要镜像组件，可以
有更高的消息吞吐量。

#### Mirrored Queus三种类型
1. **exactly**：固定数量节点策略    
设置ha-mode=axactly同时，需要设置ha-params=count。count就是节点数量，如果count=1，代表只有一个master的Mirrored Queus；如果count=2代表1个master和1个mirror。如果这里这里节点数不够count，会在所有的节点上创建Queue。如果已经有超过count的mirror queue存在，会删除多余节点上的queue。     
*注意使用ha-promote-on-shutdown=always在exactly模式下会有危险，因为Queue会跨节点迁移，使之变得不同步*

配置eg：
![img](/gallery/rabbitMQ/haExactly.png)
![img](/gallery/rabbitMQ/haExactlyDetails.png)     
    
   * Pattern:匹配正则表达式
   * Apply to:匹配队列还是交换机，这里是匹配队列
   * Definition:配置详情，这里意思是固定数量的Mirrored Queus，队列最大长度500000。Master节点选择策略是最轻负载策略。

2. **all**：全体节点策略
设置ha-mode=all，Queue会复制到全部节点，如果一个新的节点加入到集群，Queue也会自动复制到这个节点上。这个设置是非常保守的，如果不是特别重要的数据，没有必要这样，这会增加整个集群的负载，包括网络IO和磁盘IO等。通常建议复制到N/2 + 1个节点即可，N代表集群总节点数。

![img](/gallery/rabbitMQ/haAll.png)

3. **nodes**：指定名称节点
设置ha-mode=nodes同时需要配置ha-params=*node names*。node name即rabbit@hostname形式的状态。如果集群中没有node name，不会有mirror生成，但是一旦指定name的node接入集群，Queue会自动在接入节点上创建。

#### 多少个mirror合理？
首先，配置ha-mode=all是十分保守的，会加重整个集群负载，增加不必要的网络消耗和磁盘消耗，通常建议
```
mirro数=N/2+1个 //N代表节点总数
```
当然，如果有些数据是瞬态的，并且时效性很强的，设置更少的mirror或者不设置mirror也是可以的。

------------

### Mirrored Queue Master节点选择和数据迁移









