---
title: Maven deploy两次sources.jar发布到release失败问题
date: 2018-9-7 16:09:06
tags:
 -Maven
categories: Maven
thumbnail: https://raw.githubusercontent.com/apollochen123/image/master/%E9%BB%98%E8%AE%A42.jpg
---

# Maven deploy 两次sources.jar，导致发布到release失败问题
-----------
## Overview
最近在使用mvn clean install deploy 发布jar包到Nexus release仓库时，发现报错，但是在Nexus仓库中去找jar包，却是已经发布成功了。很是困惑。

参考：
1. https://maven.apache.org/plugins/maven-deploy-plugin/
2. https://maven.apache.org/plugin-developers/cookbook/attach-source-javadoc-artifacts.html
3. http://maven.apache.org/ref/3.5.4/maven-core/lifecycles.html
4. http://www.cnblogs.com/zhongshengzhen/p/nexus_maven.html

----------

## 问题重现
pom配置
```xml
</plugins>
 <plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-source-plugin</artifactId>
    <version>3.0.1</version>
    <executions>
        <execution>
            <!-- This id must match the -P release-profile id value or else sources will be "uploaded" twice, which causes Nexus to fail -->
            <id>attach-sources</id>
            <!--<phase>deploy</phase>-->
            <goals>
                <goal>jar</goal>
            </goals>
        </execution>
    </executions>
 </plugin>
</plugins>

<profiles>
    <profile>
        <id>snapshot</id>
        <distributionManagement>
            <repository>
                <id>jrepo2-public</id>
                <url>http://**/repositories/snapshots</url>
            </repository>
        </distributionManagement>
    </profile>

    <profile>
        <id>releases</id>
        <distributionManagement>
            <repository>
                <id>jrepo2-public</id>
                <url>http://**/repositories/releases</url>
            </repository>
        </distributionManagement>
    </profile>
</profiles>
```
使用mvn clean intall deploy -P releases命令

![问题重现](https://raw.githubusercontent.com/apollochen123/image/master/mvn%E4%B8%A4%E6%AC%A1deploy%E7%9B%B8%E5%90%8Cjar%E6%8A%A5%E9%94%99/%E9%97%AE%E9%A2%98%E9%87%8D%E7%8E%B0.jpg)

-----------

### 问题分析
-------

* 问题的发生点在于：Nexus，release仓库不能覆盖配置导致的。这也是为什么snapshot可以成功跑过，但是release不行的原因。
![不能覆盖](https://raw.githubusercontent.com/apollochen123/image/master/mvn%E4%B8%A4%E6%AC%A1deploy%E7%9B%B8%E5%90%8Cjar%E6%8A%A5%E9%94%99/%E4%BB%93%E5%BA%93%E4%B8%8D%E8%83%BD%E8%A6%86%E7%9B%96.jpg)

------

* 但是我们研究一下，为什么会有两次的Installing，两次deploy： ****-sources.jar
![两次sources.jar](https://raw.githubusercontent.com/apollochen123/image/master/mvn%E4%B8%A4%E6%AC%A1deploy%E7%9B%B8%E5%90%8Cjar%E6%8A%A5%E9%94%99/%E4%B8%A4%E6%AC%A1sourcesjar.jpg)
sources.jar 生成的配置
  ```xml
  <executions>
      <execution>
          <!-- This id must match the -P release-profile id value or else sources will be "uploaded" twice, which causes Nexus to fail -->
          <id>attach-sources</id>
          <!--<phase>deploy</phase>-->
          <goals>
              <goal>jar</goal>
          </goals>
      </execution>
  </executions>
  ```
这个配置会生成sources.jar
我们去官网查一下attach-sources的Lifecycles
![source生命周期](https://raw.githubusercontent.com/apollochen123/image/master/mvn%E4%B8%A4%E6%AC%A1deploy%E7%9B%B8%E5%90%8Cjar%E6%8A%A5%E9%94%99/sources-lifecle.png)
所以，执行package以后的所有生命周期，都会执行这个生成sources.jar的操作。install和deploy都会执行。也就能够解释了。

-------

* 为什么生成两个相同的source.jar都会被上传。我们先用snapshot仓库测试一下
![两次sources.jar](https://raw.githubusercontent.com/apollochen123/image/master/mvn%E4%B8%A4%E6%AC%A1deploy%E7%9B%B8%E5%90%8Cjar%E6%8A%A5%E9%94%99/%E4%B8%A4%E6%AC%A1sourcesjar.jpg)
两个一样的文件为什么会被上传两次？我们看一下deploy文档
![deploy文档](https://raw.githubusercontent.com/apollochen123/image/master/mvn%E4%B8%A4%E6%AC%A1deploy%E7%9B%B8%E5%90%8Cjar%E6%8A%A5%E9%94%99/%E6%A0%A1%E9%AA%8Cmd5.png)
这个文档提醒了我，为什么deploy会上传两个名字相同的包，因为名字一样，md5是不一样的。
所有deploy是认为生成的两个名字相同的jar，不是同一个jar（事实上也不是，md5不一样）
可以在本地文件系统验证，不修改连续install两次，看md5是否改变
![两次md5](https://raw.githubusercontent.com/apollochen123/image/master/mvn%E4%B8%A4%E6%AC%A1deploy%E7%9B%B8%E5%90%8Cjar%E6%8A%A5%E9%94%99/%E4%B8%A4%E6%AC%A1%E4%B8%8D%E5%90%8Cmd5.jpg)

--------------

## 结论
```xml
<execution>
    <id>attach-sources</id>
    <goals>
        <goal>jar</goal>
    </goals>
</execution>
```
生成source在install和deploy阶段被执行两次，生成两个sources.jar包。名字相同但是Md5不同。所以deploy会认为这是两个jar。都会上传。但是Nexus仓库会识别到这两个jar名字一样，是同一个包，加上又设置了不能覆盖，所以报400错误。

----
正确做法
父类添加pluginManagement
```xml
<pluginManagement>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-source-plugin</artifactId>
            <version>3.0.1</version>
            <executions>
                <execution>
                    <!-- This id must match the -P release-profile id value or else sources will be "uploaded" twice, which causes Nexus to fail -->
                    <id>attach-sources</id>
                    <goals>
                        <goal>jar</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</pluginManagement>
```
子类添加
```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-source-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```
然后部署命令使用
```
mvn clean deploy -P release
```
