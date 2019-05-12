---
title: RabbitMQ
date: 2018-04-3 20:47:51
tags:
 -MQ
categories: MQ
thumbnail: https://raw.githubusercontent.com/apollochen123/image/master/%E9%BB%98%E8%AE%A42.jpg
---

# RabbitMQ

------------
## reference
https://spring.io/guides/gs/messaging-rabbitmq/   
https://blog.csdn.net/olaking/article/details/47009753    
https://github.com/springframeworkguru/spring-boot-rabbitmq-example/blob/master/src/main/java/guru/springframework/SpringBootRabbitMQApplication.java   
http://www.cnblogs.com/yangh965/p/5862390.html    
https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-messaging.html   
https://docs.spring.io/spring-data/mongodb/docs/2.0.5.RELEASE/reference/html/#query-by-example    


----
## AMQP
#### What is AMQP
AMQP(高级消息队列协议) 是一个异步消息传递所使用的应用层协议规范，作为线路层协议，而不是API（例如JMS），AMQP 客户端能够无视消息的来源任意发送和接受信息。

AMQP当中有四个概念非常重要
  1. virtual host，虚拟主机
  2. exchange，交换机
  3. queue，队列
  4. binding，绑定

一个虚拟主机持有一组交换机、队列和绑定。

1. **虚拟主机（virtual host）**：为什么需要多个虚拟主机呢？因为RabbitMQ当中，用户只能在虚拟主机的粒度进行权限控制。因此，如果需要禁止A组访问B组的交换机/队列/绑定，必须为A和B分别创建一个虚拟主机。每一个RabbitMQ服务器都有一个默认的虚拟主机/。
2. **交换机（Exchange）**：每条Producer发来的Message中，都有Header和body两个部分。交换机负责解析Message中的Header，找到其中的 **路由键（routing key）** 再根据 **绑定（binding）**,确定把消息分发到哪个queue。**消费者程序（Consumer）要负责创建你的交换机。** 交换机可以存在多个，每个交换机在自己独立的进程当中执行，因此增加多个交换机就是增加多个进程，可以充分利用服务器上的CPU核以便达到更高的效率。例如，在一个8核的服务器上，可以创建5个交换机来用5个核，另外3个核留下来做消息处理。类似的，在RabbitMQ的集群当中，你可以用类似的思路来扩展交换机一边获取更高的吞吐量.
<font size="5" color="red">  交换机类型：</font>
  * Direat：直接交换机，精确匹配routing key。将消息路由到一个或者多个队列当中。一般情况下，精确匹配只会匹配一个队列，但是不排除两个队列routing key相同的情况，交换机将会复制消息，传输到所有能匹配的队列。
  正常情况下：
  ![DirectExchange](https://github.com/apollochen123/image/blob/master/DirectExchange.png?raw=true)
  多路绑定
  ![DirectExchange-多路绑定](https://github.com/apollochen123/image/blob/master/DirctExchange_%E5%A4%9A%E8%B7%AF.png?raw=true)
  * fanout：广播交换机，不管消息的rountingKey是什么类型，只要绑定了该交换机的队列都会被传达消息
  ![fanoutExchange](https://github.com/apollochen123/image/blob/master/fanoutExchange.png?raw=true)
  * topic：主题式交换机，通过rountKey和本topic交换机的设置**表达式**匹配。如果能匹配得上，则将消息发送到相关队列。经典应用场景在于Subscribe/release模型。使用主题名字空间作为消息寻址方式。**"*"匹配一个或对多个词组，"#"匹配零个或1个词组，每个标记以"."分割** example:
      * "*.stock.#"能支持"usd.stock.a"和usd.stock，但是不匹配"stock.a".
      ![匹配模型](https://raw.githubusercontent.com/apollochen123/image/master/%E5%8C%B9%E9%85%8D%E6%A8%A1%E5%9E%8B.png)

  交换机有多种类型。他们都是做路由的，但是它们接受不同类型的绑定。为什么不创建一种交换机来处理所有类型的路由规则呢？因为每种规则用来做匹配分子的CPU开销是不同的。例如，一个“topic”类型的交换机试图将消息的路由键与类似“dogs.* ”的模式进行匹配。匹配这种末端的通配符比直接将路由键与“dogs”比较（“direct”类型的交换机）要消耗更多的CPU。如果你不需要“topic”类型的交换机带来的灵活性，你可以通过使用“direct”类型的交换机获取更高的处理效率。
  ![topicExchange](https://github.com/apollochen123/image/blob/master/topicExchange.png?raw=true)



3. **队列（Queues）**：消息（messages）的终点，可以理解成装消息的容器。消息就一直在里面，直到有客户端（也就是消费者，Consumer）连接到这个队列并且将其取走为止。不过，也可以将一个队列配置成这样的：一旦消息进入这个队列，此消息就被删除。 **队列是由消费者（Consumer）通过程序建立的，不是通过配置文件或者命令行工具。** 这没什么问题，如果一个消费者试图创建一个已经存在的队列，RabbitMQ会直接忽略这个请求。因此我们可以将消息队列的配置写在应用程序的代码里面。
4. **绑定（binding）**：交换机当中有一系列的绑定（binding），即路由规则（routes）。一个绑定就是一个类似这样的规则：将交换机“desert（沙漠）”当中具有路由键“阿里巴巴”的消息送到队列“hideout（山洞）”里面去。换句话说，一个绑定就是一个基于路由键将交换机和队列连接起来的路由规则。例如，具有路由键“audit”的消息需要被送到两个队列，“log-forever”和“alert-the-big-dude”。要做到这个，就需要创建两个绑定，每个都连接一个交换机和一个队列，两者都是由“audit”路由键触发。在这种情况下，交换机会复制一份消息并且把它们分别发送到两个队列当中。

#### Workflow with AMQP
1. 消费者创建消息队列
2. 消费者定义消息队列
3. 消费者定义特定类型的交换机
4. 消费者设定绑定规则
5. 等待消息
6. 生产者创建消息
7. 生产者将消息投递至信息通道中
8. 交换机获取消息
9. 消费者获取并处理消息，并发送反馈
10. 结束，关闭通道和连接

------------
#### 关于消息队列的持久化
队列、交换机和绑定如何持久化？
队列和交换机有一个创建时候指定的标志durable。durable的唯一含义就是具有这个标志的队列和交换机会在重启之后重新建立，它不表示说在队列当中的消息会在重启后恢复。
消息如何持久化？
但是首先需要考虑的问题是：是否真的需要消息的持久化？如果需要重启后消息可以回复，那么它需要被写入磁盘。但即使是最简单的磁盘操作也是要消耗时间的。所以需要衡量判断。
当你将消息发布到交换机的时候，可以指定一个标志“Delivery Mode”（投递模式）。根据你使用的AMQP的库不同，指定这个标志的方法可能不太一样。简单的说，就是将Delivery Mode设置成2，也就是持久的（persistent）即可。一般的AMQP库都是将Delivery Mode设置成1，也就是非持久的。所以要持久化消息的步骤如下：
1. 将交换机设成 durable。
2. 将队列设成 durable。
3. 将消息的 Delivery Mode 设置成2 。

绑定（Bindings）如何持久化？
绑定无法在创建的时候设置成durable。没问题，如果你绑定了一个durable的队列和一个durable的交换机，RabbitMQ会自动保留这个绑定。类似的，如果删除了某个队列或交换机（无论是不是durable），依赖它的绑定都会自动删除。

RabbitMQ 不允许你绑定一个非坚固（non-durable）的交换机和一个durable的队列。反之亦然。要想成功必须队列和交换机都是durable的。
一旦创建了队列和交换机，就不能修改其标志了。例如，如果创建了一个non-durable的队列，然后想把它改变成durable的，唯一的办法就是删除这个队列然后重现创建。因此，最好仔细检查创建的标志。


-----------------
## RabbitMQ 安装配置
参考：https://blog.csdn.net/hzw19920329/article/details/53156015

------
## Java整合RabbitMQ
channel工厂
```java
package com.yy.apollo.mq;

import java.io.IOException;

import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;

public class ChannelFactory {

    public static Channel getChannel(String channelName) {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");
        factory.setPort(5672);
        factory.setUsername("guest");
        factory.setPassword("guest");
        Channel channel = null;
        try {
            Connection connection = factory.newConnection();
            channel = connection.createChannel();
            channel.queueDeclare(channelName, false, false, false, null);
        } catch (IOException e) {
            e.printStackTrace();
        }
        return channel;
    }
}

```
生产者：
```java
package com.yy.apollo.mq;

import java.io.IOException;
import org.apache.commons.lang.SerializationUtils;
import com.rabbitmq.client.Channel;

public class Producer {

    private Channel channel;

    public void sendMessage(String queue,String message) {
        try {
            channel.basicPublish("", queue, null, SerializationUtils.serialize(message));
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public Producer(Channel channel) {
        super();
        this.channel = channel;
    }
}

```
消费者：
```Java
package com.yy.apollo.mq;

import java.io.IOException;

import org.apache.commons.lang3.SerializationUtils;

import com.rabbitmq.client.AMQP.BasicProperties;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Consumer;
import com.rabbitmq.client.Envelope;
import com.rabbitmq.client.ShutdownSignalException;

public class MyConsumer implements Runnable , Consumer{

    @Override
    public void run() {
        try {
            channel.basicConsume(queue, true, this);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private Channel channel;
    public MyConsumer(Channel channel, String queue) {
        super();
        this.channel = channel;
        this.queue = queue;
    }

    private String queue;
    @Override
    public void handleConsumeOk(String consumerTag) {
        System.out.println("Consumer: "+ consumerTag);
    }
    @Override
    public void handleCancelOk(String consumerTag) {

    }
    @Override
    public void handleCancel(String consumerTag) throws IOException {

    }
    @Override
    public void handleDelivery(String arg0, Envelope arg1, BasicProperties arg2, byte[] arg3) throws IOException {
        System.out.println(Thread.currentThread()+":"+"Consumer消费"+SerializationUtils.deserialize(arg3));
    }
    @Override
    public void handleShutdownSignal(String consumerTag, ShutdownSignalException sig) {

    }
    @Override
    public void handleRecoverOk(String consumerTag) {

    }

}

```
main
```java
package com.yy.apollo.mq;

import com.rabbitmq.client.Channel;

public class Main {
    public static void main(String[] args) {
        Channel send = ChannelFactory.getChannel("Apollo");
        Channel client = ChannelFactory.getChannel("Apollo");
        Channel client2 = ChannelFactory.getChannel("Apollo");
        Producer producer = new Producer(send);
        MyConsumer consumer = new MyConsumer(client,"Apollo");
        MyConsumer consumer2 = new MyConsumer(client2,"Apollo");

        Thread thread1 = new Thread(consumer,"11:");
        Thread thread2 = new Thread(consumer2,"22:");
        thread1.start();
        thread2.start();

        for(int i = 0;i<10000;i++) {
            producer.sendMessage("Apollo", "消息"+i);
            System.out.println("发送消息"+i);
        }
    }
}

```
pom配置
```
<dependency>
  <groupId>junit</groupId>
  <artifactId>junit</artifactId>
  <version>3.8.1</version>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>com.rabbitmq</groupId>
  <artifactId>amqp-client</artifactId>
  <version>3.0.4</version>
</dependency>

<dependency>
  <groupId>commons-lang</groupId>
  <artifactId>commons-lang</artifactId>
  <version>2.6</version>
</dependency>
<dependency>
  <groupId>org.apache.commons</groupId>
  <artifactId>commons-lang3</artifactId>
  <version>3.1</version>
</dependency>
```


-----------
## RabbitMQ 与spring boot整合
### Step by step
1. 添加springboot的RabiitMQ依赖
```
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```
2. 配置RabbitMQ
  这里使用纯java的方式配置，没用使用xml.代码配置详情见注释
  ```java
  package com.yy.Apollo.demointegration.rabbitmq;

  import java.util.HashMap;
  import java.util.Map;

  import org.springframework.amqp.core.Binding;
  import org.springframework.amqp.core.BindingBuilder;
  import org.springframework.amqp.core.DirectExchange;
  import org.springframework.amqp.core.Queue;
  import org.springframework.amqp.rabbit.connection.ConnectionFactory;
  import org.springframework.amqp.rabbit.listener.SimpleMessageListenerContainer;
  import org.springframework.amqp.rabbit.listener.adapter.MessageListenerAdapter;
  import org.springframework.beans.factory.annotation.Qualifier;
  import org.springframework.context.annotation.Bean;
  import org.springframework.context.annotation.Configuration;

  @Configuration
  public class MessageConfig {

      // 交换机
      private static String EXCHANGE = "DirectExchange";

      public static final String QUEUENAME1 = "queue1";
      public static final String QUEUENAME2 = "queue2";

      public static String getEXCHANGE() {
          return EXCHANGE;
      }

      public static void setEXCHANGE(String eXCHANGE) {
          EXCHANGE = eXCHANGE;
      }

      /**
       * 设置对了名为queue1的队列
       *
       * @return
       */
      @Bean("queue1")
      public Queue queue1() {
          return new Queue(QUEUENAME1, false);// 队列不持久
      }

      /**
       * 设置对了名为queue2的队列
       *
       * @return
       */
      @Bean("queue2")
      public Queue queue2() {
          return new Queue("queue2", false);// 队列不持久
      }

      /**
       * 配置交换机,这里是使用精确匹配方式
       *
       * @return
       */
      @Bean
      public DirectExchange exchange() {
          return new DirectExchange(EXCHANGE, false, false);
      }

      /**
       * 绑定交换机与队列，下列配置的意思是，如果到达交换机DirectExchange的消息中routeKey=qu.1，则把这条消息放入队列queue1
       *
       * @param queue
       * @param exchange
       * @return
       */
      @Bean
      public Binding binding1(@Qualifier(QUEUENAME1) Queue queue, DirectExchange exchange) {
          return BindingBuilder.bind(queue).to(exchange).with("qu.1"); // 如果消息的路由Key为qu.1，则交换机将消息发送到队列1
      }

      /**
       * 绑定交换机与队列，下列配置的意思是，如果到达交换机DirectExchange的消息中routeKey=qu.2，则把这条消息放入队列queue2
       *
       * @param queue
       * @param exchange
       * @return
       */
      @Bean
      public Binding binding2(@Qualifier(QUEUENAME2) Queue queue, DirectExchange exchange) {
          return BindingBuilder.bind(queue).to(exchange).with("qu.2"); // 如果消息的路由Key为qu.2，则交换机将消息发送到队列2
      }

      @Bean
      SimpleMessageListenerContainer container(ConnectionFactory connectionFactory,
              MessageListenerAdapter listenerAdapter) {
          // 生成一个Listener的容器
          SimpleMessageListenerContainer container = new SimpleMessageListenerContainer();
          // 使用springboot自动生成的connection工厂
          container.setConnectionFactory(connectionFactory);
          // 这个container负责监听queue1与quue2消息
          container.setQueueNames(QUEUENAME1, QUEUENAME2);
          // 其实我们的listener只是一个pojo类，需要listenerAdapter来让我们在pojo中定义的方法来执行。如果使用@RabbitListener可以不用设定。
          container.setMessageListener(listenerAdapter);
          return container;
      }

      /**
       * 这是代码配置listenerAdapter，可以使用RabbitListener代替本步骤
       *
       * @param messageRcv
       * @return
       */
      @Bean
      MessageListenerAdapter listenerAdapter(MessageRcv messageRcv) {
          MessageListenerAdapter adapter = new MessageListenerAdapter(messageRcv);
          Map<String, String> queueOrTagToMethodName = new HashMap<>();
          queueOrTagToMethodName.put(QUEUENAME1, "onMessageQueue1");//onMessageQueue1反射调用的方法
          queueOrTagToMethodName.put(QUEUENAME2, "onMessageQueue2");
          adapter.setQueueOrTagToMethodName(queueOrTagToMethodName);
          return adapter;
      }

  }
  ```

3. 消息接收类，分别两种方式，listenerAdapter+普通方法反射，和使用@RabbitListener(queues = MessageConfig.QUEUENAME1)
  ```java
  package com.yy.Apollo.demointegration.rabbitmq;

  import org.springframework.amqp.rabbit.annotation.RabbitListener;
  import org.springframework.stereotype.Component;

  @Component
  public class MessageRcv {

      public void onMessageQueue1(String message) {
          System.out.println("Queue1收到消息 : " + new String(message));
      }

      public void onMessageQueue2(String message) {
          System.out.println("Queue2收到消息 : " + new String(message));
      }

      @RabbitListener(queues = MessageConfig.QUEUENAME1)
      public void processMessage(String content) {
          System.out.println("@RabbitListener:"+content);
      }

      @RabbitListener(queues = MessageConfig.QUEUENAME2)
      public void processMessage2(String content) {
          System.out.println("@RabbitListener2:"+content);
      }

  }
  ```

4. 消息发送类
  ```java
  package com.yy.Apollo.demointegration.rabbitmq;

  import java.util.UUID;

  import org.springframework.amqp.rabbit.core.RabbitTemplate;
  import org.springframework.amqp.rabbit.support.CorrelationData;
  import org.springframework.beans.factory.annotation.Autowired;
  import org.springframework.stereotype.Component;

  @Component
  public class MessageSend {
      @Autowired
      private RabbitTemplate rabbitTemplate;

      public void sendMsg(String message, String queue) {
          String uuid = UUID.randomUUID().toString();
          CorrelationData correlationId = new CorrelationData(uuid);
          if (queue.equals("1")) {
              rabbitTemplate.convertAndSend(MessageConfig.getEXCHANGE(), "qu.1", message, correlationId);
          } else {
              rabbitTemplate.convertAndSend(MessageConfig.getEXCHANGE(), "qu.2", message, correlationId);
          }

      }
  }

  ```

5. 消息发送的controller
  ```java
  package com.yy.Apollo.demointegration.controller;

  @RestController
  @RequestMapping("/mongo")
  public class TestController {  
      @Autowired
      private MessageSend messageSend;

      @GetMapping("/sendMsg/{queue}/{message}")
      public String sendMsg(@PathVariable("message") String message,@PathVariable("queue")String queue) {
          messageSend.sendMsg(message,queue);
          System.out.println("Message!发送成功");
          return "OK";
      }
  }

  ```

6. yml配置
  ```yml
  server:
      port: 8828
  spring:
      name: demointegration
      rabbitmq:
          host: localhost
          port: 5672
          username: guest
          password: guest
  ```
