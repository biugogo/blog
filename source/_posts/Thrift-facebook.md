---
title: Thrift Annotation and Maven Pugin
date: 2018-4-24 21:02:10
tags:
 -RPC
categories: RPC
thumbnail: https://raw.githubusercontent.com/apollochen123/image/master/%E9%BB%98%E8%AE%A42.jpg
---

# Thrift Annotation and maven pugin
----------
## Overview
最近项目里用到了Thrift，但是传统的方式，会生成很多冗余代码。我们用不上也不需要看，但是对于thrift
必须用到的。并且开发很是麻烦，还得切换命令行什么的。这里记录一个demo。使用maven自动生成thrift接口类并且使用thrift Annotation 和 thrift service。这里仅仅是一个例子，距离生成使用thrift还是有一定差距，毕竟还得有服务发现注册等等组件。
这里的server和client已经实现了多线程pool等，但是在例子中没有使用，请根据实际生产配置。

-----------
## Step1 使用maven构建thrift文件
pom配置
```
<profiles>
  <profile>
    <id>genThrift</id>
    <build>
      <plugins>
        <plugin>
          <groupId>com.facebook.mojo</groupId>
          <artifactId>swift-maven-plugin</artifactId>
          <version>0.12.2-SNAPSHOT</version>
          <executions>
            <execution>
              <goals>
                <goal>generate</goal>
              </goals>
            </execution>
          </executions>
          <configuration>
            <idlFiles>
              <!--这里是.thrift存放的位置 -->
              <directory>src/main/thrift</directory>
              <includes>
                <include>
                  *.thrift
                </include>
              </includes>
              <excludes>
              </excludes>
            </idlFiles>
            <usePlainJavaNamespace>true</usePlainJavaNamespace>
            <!-- 配置java文件输出的目录 -->
            <outputFolder>src/main/java</outputFolder>
            <!-- 不添加thriftException，使用自定义exception -->
            <addThriftExceptions>false</addThriftExceptions>
            <!-- 自定义配置Annotation，定义的annotation会自动追加到生成的接口上 -->
            <serviceAnnotations>
              <serviceAnnotation>com.yy.apollo.demo.ServiceCenterExporter</serviceAnnotation>
              <serviceAnnotation>com.yy.apollo.demo.ServiceCenterReference</serviceAnnotation>
            </serviceAnnotations>
          </configuration>
        </plugin>
      </plugins>
    </build>
  </profile>
</profiles>
```
maven命令
```
mvn clean install -P genThrift
```
eclipse中配置图
![Thrift-maven配置](https://github.com/apollochen123/image/blob/master/thrift/thrift-maven%E9%85%8D%E7%BD%AE%E5%9B%BE.jpg?raw=true)

## Step2 Demo的.thrift文件
```
namespace java.swift com.yy.apollo.demo.thrift
namespace java com.yy.apollo.demo.thrift

enum RankType{
    DAY,
    WEEK,
    MONTH,
    ALL
}
enum RankSort{
    ASC,
    DESC
}

enum ErrorCode{
    APP_INFO_NOT_EXIST,
    SYS_ERROR,
    OPTIONS_ERROR
}
exception RankException{
    1: required ErrorCode code;
    2: optional string msg;
}
struct RankItem{
    1: optional string id;
    2: optional i64 score;
    3: optional i64 rank;
    4: optional RankType type;
    5: optional string rankName;
}

typedef list<RankItem> RankItemList
typedef list<string> RankNameList
typedef list<RankType> RankTypeList

struct AppInfo{
    1: required string appId;
    2: required string rankId;
}

struct RankInfo{
    1: required RankType type;
    2: optional string name;
}

struct PutScoreOptions{
    1: optional i64 time;
    2: optional RankNameList rankNameList;
    3: optional RankTypeList excludeRankTypeList;
    4: optional bool includeItemIdRank;
}

struct PutScoreResult{
    1: optional RankItemList itemIdRankList;
    2: optional map<string,RankItemList> customNameMap
}


struct GetRangeOptions{
    1: optional i64 time;
    2: optional RankNameList rankNameList;
    3: optional bool reversed;
}

struct GetRangeResult{
    1: optional RankItemList rankList;
    2: optional map<string,RankItemList> nameRankListMap;
}

struct GetRankOptions{
    1: optional i64 time;
    2: optional string rankName;
    3: optional i32 preN;
    4: optional i32 nextN;
    5: optional bool reversed;
}

struct GetRankResult{
    1: optional RankItem rank;
    2: optional RankItemList preList;
    3: optional RankItemList nextList
}

service WolfkillRankService{
    void ping();

    /**
    * 将itemId对应的itemScore放入排行榜中
    *
    * @param appInfo 排行榜应用信息
    * @param itemId 排行项Id
    * @param itemScore 排行项积分
    * @param options 可选参数
    *
    * @throws RankException 失败则抛出排行榜异常
    */
    PutScoreResult putScore(1:AppInfo appInfo,2:string itemId,3:i64 itemScore, 4:PutScoreOptions options) throws (1:RankException rankException);

    /**
    * 查询排行榜区间
    *
    * @param appInfo 排行榜应用信息
    * @param type 排行榜类型
    * @param start 开始下标（0开始）
    * @param stop 结束下标
    * @param options 可选参数
    *
    * @throws RankException 失败则抛出排行榜异常
    */
    GetRangeResult getRange(1:AppInfo appInfo,2:RankType type,3:i64 start,4:i64 stop,5:GetRangeOptions options) throws (1:RankException rankException);

    /**
    * 查询排行信息
    *
    * @param appInfo 排行榜应用信息
    * @param type 排行榜类型
    * @param itemId 排行项Id
    * @param options 可选参数
    *
    * @throws RankException 失败则抛出排行榜异常
    */
    GetRankResult getRank(1:AppInfo appInfo,2:RankType type,3:string itemId,4:GetRankOptions options) throws (1:RankException rankException);
}
```
## step3 生成的thrift服务类
```java
package com.yy.apollo.demo.thrift;

import com.facebook.swift.codec.*;
import com.facebook.swift.service.*;
import com.google.common.util.concurrent.ListenableFuture;
import java.io.*;
import java.util.*;

@com.yy.apollo.demo.ServiceCenterExporter  
@com.yy.apollo.demo.ServiceCenterReference  
@ThriftService("WolfkillRankService")
public interface WolfkillRankService
{
	@com.yy.apollo.demo.ServiceCenterExporter  
	@com.yy.apollo.demo.ServiceCenterReference  
    @ThriftService("WolfkillRankService")
    public interface Async
    {
        @ThriftMethod(value = "ping")
        ListenableFuture<Void> ping();

        @ThriftMethod(value = "putScore")
        ListenableFuture<PutScoreResult> putScore(
            @ThriftField(value=1, name="appInfo") final AppInfo appInfo,
            @ThriftField(value=2, name="itemId") final String itemId,
            @ThriftField(value=3, name="itemScore") final long itemScore,
            @ThriftField(value=4, name="options") final PutScoreOptions options
        );

        @ThriftMethod(value = "getRange")
        ListenableFuture<GetRangeResult> getRange(
            @ThriftField(value=1, name="appInfo") final AppInfo appInfo,
            @ThriftField(value=2, name="type") final RankType type,
            @ThriftField(value=3, name="start") final long start,
            @ThriftField(value=4, name="stop") final long stop,
            @ThriftField(value=5, name="options") final GetRangeOptions options
        );

        @ThriftMethod(value = "getRank")
        ListenableFuture<GetRankResult> getRank(
            @ThriftField(value=1, name="appInfo") final AppInfo appInfo,
            @ThriftField(value=2, name="type") final RankType type,
            @ThriftField(value=3, name="itemId") final String itemId,
            @ThriftField(value=4, name="options") final GetRankOptions options
        );
    }
    @ThriftMethod(value = "ping")
    void ping();


    @ThriftMethod(value = "putScore",
                  exception = {
                      @ThriftException(type=RankException.class, id=1)
                  })
    PutScoreResult putScore(
        @ThriftField(value=1, name="appInfo") final AppInfo appInfo,
        @ThriftField(value=2, name="itemId") final String itemId,
        @ThriftField(value=3, name="itemScore") final long itemScore,
        @ThriftField(value=4, name="options") final PutScoreOptions options
    ) throws RankException;

    @ThriftMethod(value = "getRange",
                  exception = {
                      @ThriftException(type=RankException.class, id=1)
                  })
    GetRangeResult getRange(
        @ThriftField(value=1, name="appInfo") final AppInfo appInfo,
        @ThriftField(value=2, name="type") final RankType type,
        @ThriftField(value=3, name="start") final long start,
        @ThriftField(value=4, name="stop") final long stop,
        @ThriftField(value=5, name="options") final GetRangeOptions options
    ) throws RankException;

    @ThriftMethod(value = "getRank",
                  exception = {
                      @ThriftException(type=RankException.class, id=1)
                  })
    GetRankResult getRank(
        @ThriftField(value=1, name="appInfo") final AppInfo appInfo,
        @ThriftField(value=2, name="type") final RankType type,
        @ThriftField(value=3, name="itemId") final String itemId,
        @ThriftField(value=4, name="options") final GetRankOptions options
    ) throws RankException;
}
```
## Step4 创建server
```java
package com.yy.apollo.demo.server;

import java.util.Arrays;

import com.facebook.nifty.core.RequestContext;
import com.facebook.swift.codec.ThriftCodecManager;
import com.facebook.swift.service.ThriftEventHandler;
import com.facebook.swift.service.ThriftServer;
import com.facebook.swift.service.ThriftServerConfig;
import com.facebook.swift.service.ThriftServiceProcessor;
import com.yy.apollo.demo.service.ExampleService;
import com.yy.apollo.demo.thrift.WolfkillRankService;

public class Server {

    public static void main(String[] args) {
        WolfkillRankService wolfkillRankService = new ExampleService();
        ThriftServiceProcessor processor = new ThriftServiceProcessor(new ThriftCodecManager(),
                Arrays.asList(new myhander()), wolfkillRankService);
        @SuppressWarnings("resource")
        ThriftServer server = new ThriftServer(processor, new ThriftServerConfig().setPort(9001)).start();
    }

    static class myhander extends ThriftEventHandler {
        public Object getContext(String methodName, RequestContext requestContext) {
            System.out.println("methodName:" + methodName + "       requestContext:" + requestContext);
            return requestContext;
        }
    }
}
```

## Step5 创建 client
```java
package com.yy.apollo.demo.client;

import java.util.concurrent.ExecutionException;

import com.facebook.nifty.client.FramedClientConnector;
import com.facebook.swift.service.ThriftClientManager;
import com.google.common.net.HostAndPort;
import com.yy.apollo.demo.thrift.AppInfo;
import com.yy.apollo.demo.thrift.WolfkillRankService;

public class ThriftClient {

    public static void main(String[] args) {
        ThriftClientManager clientManager = new ThriftClientManager();
        FramedClientConnector connector = new FramedClientConnector(HostAndPort.fromParts("localhost", 9001));
        try {
            WolfkillRankService wolfkillRankService = clientManager.createClient(connector, WolfkillRankService.class).get();
            wolfkillRankService.ping();
            System.out.println(wolfkillRankService.putScore(new AppInfo(), "itemId", 888, null));
        } catch (InterruptedException | ExecutionException e) {
            e.printStackTrace();
        }
    }
}
```

## 执行结果
先运行server，在运行client
### console for client

```
PutScoreResult{itemIdRankList=[RankItem{id=itemId, score=888, rank=0, type=null, rankName=null}], customNameMap=null}
```

### console for Server
```
methodName:WolfkillRankService.ping       requestContext:com.facebook.nifty.core.NiftyRequestContext@1201209d
Pong
methodName:WolfkillRankService.putScore       requestContext:com.facebook.nifty.core.NiftyRequestContext@7e12da9a
```
