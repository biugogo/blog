# Redis系列(1)---管道

### RTT(Round Trip Time)

Redis是一个基于TCP的服务，所以对于一次请求大致可以分为两个step:

1. Client发送一个command给Redis Server,然后等待server回包（通常会阻塞等待）。
2. Server处理完command,返回response给Clinet.

Client和server之间的网络通信。在同机房时，会很快，但是当他们再不同机房时，通信的开销会变得很大。
当然，不管他们的网络通信延迟多少，command都会从client传输到server，在从server取回回包。这个通信间隔我们称作RTT (Round Trip Time).当Clinet批量发送命令时，假设RTT为100ms（当然生产不会这么长），即使server每秒可以出来单client十万个请求，但是RTT存在，也只能处理10个。当我们连接本地（127.0.0.1）时，RTT也会在0.05ms左右，但是这还是很长，在你需要啊批量处理很多command时。

### Pipelining

Redis解决RTT的方案是提供一个pipelining管道，Client一次发送一批command给Server，server按发送的顺序一次批量回包（灵感来源于POP3协议）.
<br>
*注意:* **当你用pipelining发送命令时，redis将会强制在server内存对你发送的命令进行排序和处理，所以使用pipelining发送命令也不是一次发送越多越好。这会耗费server的内存。官方建议是一次不超过10k**



### Pipeinling带来的性能提升不只是减少了RTT

管道带来的提升，并不简单是减少了RTT。使用管道，可以极大的提升server的QPS。这是因为执行命令的开销很小，但是一次IO的消耗却很大。系统进行IO的read和write的调用时，意味着用户态到内核态的切换，这个上下文切换的消耗是十分巨大的。（跟业务相似，瓶颈再IO）。     
<br>
使用管道可以有效减少用户态到内核态的上下文切换（pipeing只进行一次read和一次wirte）.官方给出的数据，使用管道基本可以提升10倍的性能。当一次发送80个command时，比不使用pipeining性能可以提升到10倍。
![来自官方的测试](https://redis.io/images/redisdoc/pipeline_iops.png)
<br>

*当Client和Server在同一台机器上时*






