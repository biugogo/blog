---
title: Thrift
date: 2018-3-24 18:02:10
tags:
 -RPC
categories: RPC
thumbnail: https://raw.githubusercontent.com/apollochen123/image/master/%E9%BB%98%E8%AE%A42.jpg
---
# Thrift
-----------
## Overview
>The Apache Thrift software framework, for scalable cross-language services development, combines a software stack with a code generation engine to build services that work efficiently and seamlessly between C++, Java, Python, PHP, Ruby, Erlang, Perl, Haskell, C#, Cocoa, JavaScript, Node.js, Smalltalk, OCaml and Delphi and other languages.

简单说，就是一个跨平台的代码生成工具，生成代码是用来远程调用函数的框架.

------
参考博文：
* http://thrift-tutorial.readthedocs.io/en/latest/thrift-stack.html
* http://thrift.apache.org/static/files/thrift-20070401.pdf
* https://www.cnblogs.com/exceptioneye/p/4945073.html
* http://blog.163.com/kewangwu@126/blog/static/86728471201271353354581/
* https://www.jianshu.com/p/0f4113d6ec4b
------
## Quick Start
本文是基于window的，linux其实是类似的

1. 下载thirft.exe [官网下载](http://archive.apache.org/dist/thrift/),本文选择0.9.3版本 thrift-0.9.3.exe ，并将下载文件重命名为thrift.exe
2. 配置环境变量：
![配置环境变量](https://raw.githubusercontent.com/apollochen123/image/master/thrift%E7%9B%AE%E5%BD%95%E7%BB%93%E6%9E%84.jpg)

3. 查看是否安装成功
  ```
  thrift -version
  ```
4. 编写一个.thrift文件，example：Hello.thrift
```
namespace java com.yy.apolloTest
 service Hello {
  string hi(1:string para);
}
```
这里的namespace 类似java的包
5. 进入Hello.thrift所在目录，使用命令行把上面的Hello.thrift编译为java代码
```
thrift --gen java Hello.thrift
```
未报错，说明编译成功，你能在Hello.thrift所在目录看到gen-java文件生成，文件结构为上方namespace定义的结构
![文件结构](https://raw.githubusercontent.com/apollochen123/image/master/thrift%E6%96%87%E4%BB%B6%E7%BB%93%E6%9E%84.png)

6. 拷贝生成的java文件到项目工程
   ![图](https://raw.githubusercontent.com/apollochen123/image/master/thrift%E9%A1%B9%E7%9B%AE%E6%A0%91%E7%8A%B6%E5%9B%BE.png)


----
## Simple example with java
### 1.实现Hello.Iface，实现你需要被远程调用的方法
``` java
package com.yy.apollotest;

import java.nio.ByteBuffer;
import java.text.SimpleDateFormat;
import org.apache.thrift.TException;
import com.google.protobuf.InvalidProtocolBufferException;
import com.yy.apollotest.protoc.UserProtos;

public class HelloImpl implements Hello.Iface {

    SimpleDateFormat format = new SimpleDateFormat("yyyy.MM.dd:HH:mm:ss");

    @Override
    public String hi(String para) throws TException {
        return format.format(System.currentTimeMillis());
    }
}
```
### 2.实现Service
```java
package com.yy.apollotest.server;

import org.apache.thrift.protocol.TBinaryProtocol;
import org.apache.thrift.transport.TServerSocket;
import org.apache.thrift.transport.TTransportException;
import com.yy.apollotest.Hello;
import com.yy.apollotest.Hello.Processor;
import com.yy.apollotest.HelloImpl;
import org.apache.thrift.protocol.TBinaryProtocol.Factory;
import org.apache.thrift.server.TServer;
import org.apache.thrift.server.TThreadPoolServer;
import org.apache.thrift.server.TThreadPoolServer.Args;

public class Server {
    public void startServer() {
        System.out.println("server start");
        try {
            //使用Socket 协议传输
            TServerSocket serverTransport = new TServerSocket(9001);

            //处理器，包含需要被远程调用的方法
            Hello.Processor process = new Processor(new HelloImpl());

            //使用二进制编码
            Factory portFactory = new TBinaryProtocol.Factory(true, true);

            //配置Server的参数
            Args args = new Args(serverTransport);
            args.processor(process);
            args.protocolFactory(portFactory);
            //配置使用哪种Server，这里是使用线程池Server
            TServer server = new TThreadPoolServer(args);
            server.serve();

        } catch (TTransportException e) {
            e.printStackTrace();
        }
        System.out.println("server end");
    }

    public static void main(String[] args) {
        Server server = new Server();
        server.startServer();
    }
}

```
### 3. 实现client
```java
package com.yy.apollotest.client;

import java.io.IOException;
import java.nio.ByteBuffer;
import org.apache.thrift.TException;
import org.apache.thrift.protocol.TBinaryProtocol;
import org.apache.thrift.protocol.TProtocol;
import org.apache.thrift.transport.TSocket;
import org.apache.thrift.transport.TTransport;
import org.apache.thrift.transport.TTransportException;
import com.yy.apollotest.Hello;
import com.yy.apollotest.protoc.UserAdd;
import com.yy.apollotest.protoc.UserProtos;

public class Client {
    public void startClient() {
        System.out.println("client");
        TTransport transport;
        //访问9001端口的socket
        transport = new TSocket("localhost", 9001);
        //使用二进制编码协议
        TProtocol protocol = new TBinaryProtocol(transport);
        //创建client
        Hello.Client client = new Hello.Client(protocol);
        try {
            //打开连接
            transport.open();
            //远程调用hi()方法，print返回String
            System.out.println(client.hi("2018"));
        } catch (TTransportException e) {
            e.printStackTrace();
        } catch (TException e) {
            e.printStackTrace();
        }

        transport.close();
        System.out.println("client close");
    }

    public static void main(String[] args) {
        Client client = new Client();
        client.startClient();
    }
}

```
Console
```
client
2018.03.24:17:12:59
client close
```

### 4.**遇到的问题**：
生成的Hello.java在Eclipse中报错，自动编译的代码跑不过，需要mvn compile才能跑，项目会有叉，但是不影响运行。逼死强迫症系列。
在Idea中完美运行，没有问题
![错误](https://raw.githubusercontent.com/apollochen123/image/master/thrift%E7%94%9F%E6%88%90%E6%96%87%E4%BB%B6%E6%8A%A5%E9%94%99.png)

----------
## 简单原理分析
![原理图](https://raw.githubusercontent.com/apollochen123/image/master/thrift%E5%8E%9F%E7%90%86%E5%9B%BE%E8%A7%A3.png)

Thrift 是一个跨语言的序列化/RPC框架，它含有三个主要的组件：
1. protocol：protocol定义了消息是怎样序列化的
2. transport：transport定义了消息是怎样在客户端和服务器端之间通信的，server用于从transport接收序列化的消息，根据protocol反序列化之，调用用户定义的消息处理器，并序列化消息处理器的响应，然后再将它们写回transport
3. server：单线程Server还是多线程，线程池server，是阻塞方式还是非阻塞方式

### Protocol
thrift的序列化方式主要有：  
**TBinaryProtocol**：二进制协议-直接二进制格式编码数字值为二进制，而不是转换为文本。  
**TCompactProtocol**：紧凑的协议-非常有效，密集的数据编码.  
**TDenseProtocol**：密协议类似tcompactprotocol，从传送的内容中去除元信息，并将其添加回接收器
。密集的协议仍然是实验性的和java暂时不可用   
**TJSONProtocol**：使用Json编码数据  
**TSimpleJSONProtocol**：一个只用json写的协议，适合脚本语言解析
**TDebugProtocol**：使用人类可以看的文本传输，在debug时候
![优缺点](https://raw.githubusercontent.com/apollochen123/image/master/Thrift%E4%BC%A0%E8%BE%93%E6%A0%BC%E5%BC%8F%E7%9A%84%E4%BC%98%E7%BC%BA%E7%82%B9.png)

### Transport
TSocket：套接字-用于传输的阻塞套接字I/O。  
TFramedTransport：帧传输——在帧中发送数据，每个帧在前面加上一个长度。使用非阻塞服务器时需要进行此传输.   TFileTransport：文件传输-此传输将写入文件。而这个运输不包含java实现的，它应该是足够简单的实现。
TMemoryTransport：存储运输使用内存I/O的java实现使用一个简单的ByteArrayOutputStream实现。  
TZlibTransport：zlib压缩使用zlib运输执行。与其他交通工具一起使用。java实现不可用。

### Server
thrift的序列化方式主要有
* TSimpleServer   
TSimplerServer接受一个连接，处理连接请求，直到客户端关闭了连接，它才回去接受一个新的连接。正因为它只在一个单独的线程中以阻塞I/O的方式完成这些工作，所以它只能服务一个客户端连接，其他所有客户端在被服务器端接受之前都只能等待。TSimpleServer主要用于测试目的，不要在生产环境中使用它！
* TNonblockingServer    
TNonblockingServer使用非阻塞的I/O解决了TSimpleServer一个客户端阻塞其他所有客户端的问题。它使用了java.nio.channels.Selector，通过调用select()，它使得你阻塞在多个连接上，而不是阻塞在单一的连接上。当一或多个连接准备好被接受/读/写时，select()调用便会返回。TNonblockingServer处理这些连接的时候，要么接受它，要么从它那读数据，要么把数据写到它那里，然后再次调用select()来等待下一个可用的连接。通用这种方式，server可同时服务多个客户端，而不会出现一个客户端把其他客户端全部“饿死”的情况。然而，还有个棘手的问题：所有消息是被调用select()方法的同一个线程处理的。假设有10个客户端，处理每条消息所需时间为100毫秒，那么，latency和吞吐量分别是多少？当一条消息被处理的时候，其他9个客户端就等着被select，所以客户端需要等待1秒钟才能从服务器端得到回应，吞吐量就是10个请求/秒。如果可以同时处理多条消息的话，会很不错吧？
* THsHaServer   
THsHaServer（半同步/半异步的server）就应运而生了。它使用一个单独的线程来处理网络I/O，一个独立的worker线程池来处理消息。这样，只要有空闲的worker线程，消息就会被立即处理，因此多条消息能被并行处理。用上面的例子来说，现在的latency就是100毫秒，而吞吐量就是100个请求/秒。
* TThreadedSelectorServer   
Thrift 0.8引入了另一种server实现，即TThreadedSelectorServer。它与THsHaServer的主要区别在于，TThreadedSelectorServer允许你用多个线程来处理网络I/O。它维护了两个线程池，一个用来处理网络I/O，另一个用来进行请求的处理。当网络I/O是瓶颈的时候，TThreadedSelectorServer比THsHaServer的表现要好。为了展现它们的区别，我进行了一个测试，令其消息处理器在不做任何工作的情况下立即返回，以衡量在不同客户端数量的情况下的平均latency和吞吐量。对THsHaServer，我使用32个worker线程；对TThreadedSelectorServer，我使用16个worker线程和16个selector线程。
* TThreadPoolServer   
TThreadPoolServer与其他三种server不同的是： 有一个专用的线程用来接受连接。一旦接受了一个连接，它就会被放入ThreadPoolExecutor中的一个worker线程里处理。worker线程被绑定到特定的客户端连接上，直到它关闭。一旦连接关闭，该worker线程就又回到了线程池中。你可以配置线程池的最小、最大线程数，默认值分别是5（最小）和Integer.MAX_VALUE（最大）。

----------------
## Thrift数据结构
### 一个稍复杂的.thrift文件
```
namespace java com.yy.apollotest

enum RequestType {
   SAY_HELLO,   //问好
   QUERY_TIME,  //询问时间
}

struct Request {
   1: required RequestType type;  // 请求的类型，必选
   2: required string name;       // 发起请求的人的名字，必选
   3: optional i32 age;           // 发起请求的人的年龄，可选
}

exception RequestException {
   1: required i32 code;
   2: optional string reason;
}

// 服务名
service HelloWordService {
   string doAction(1: Request request) throws (1:RequestException qe); // 可能抛出异常。
}
```

### thrift支持的数据类型
1.基本类型（括号内为对应的Java类型）：
bool（boolean）: 布尔类型(TRUE or FALSE)
byte（byte）: 8位带符号整数
i16（short）: 16位带符号整数
i32（int）: 32位带符号整数
i64（long）: 64位带符号整数
double（double）: 64位浮点数
string（String）: 采用UTF-8编码的字符串

2.特殊类型（括号内为对应的Java类型）：
binary（ByteBuffer）：未经过编码的字节流

3.Structs（结构）：
struct定义了一个很普通的OOP对象，但是没有继承特性。
```
struct UserProfile {
1: i32 uid = 1,  //默认值为1
2: string name = "User1",
3: string blurb
}
```
4.collection
```
list<i32> subNodeList,
map<i32,string> subNodeMap,
set<i32> subNodeSet
```
