---
title: Maven package 可执行Jar
date: 2018-5-20 16:18:10
tags:
 -Maven
categories: Maven
thumbnail: https://raw.githubusercontent.com/apollochen123/image/master/%E9%BB%98%E8%AE%A42.jpg
---

# Maven package 可执行Jar
----
## Overview
最近遇到一个问题，用java写了一个简单的脚本想要自动化配置一些东西。但是当在IDE上面成功运行后，打包成jar文件却不能运行。找不到依赖的包。  换用了打包插件后可以运行了。这里总结一下run jar 文件。

----
## 问题再现
我们使用maven配置
```xml
<plugins>
     <plugin>
         <groupId>org.apache.maven.plugins</groupId>
         <artifactId>maven-jar-plugin</artifactId>
         <version>3.0.0</version>
         <configuration>
             <archive>
                 <manifest>
                 <addClasspath>true</addClasspath>
                 <classpathPrefix>libs/</classpathPrefix>
                 <mainClass>com.yy.apollo.autopush.App</mainClass>
                 <classpathLayoutType>simple</classpathLayoutType>
                 </manifest>
             </archive>
         </configuration>
     </plugin>
     <plugin>
         <groupId>org.apache.maven.plugins</groupId>
         <artifactId>maven-dependency-plugin</artifactId>
         <version>3.0.0</version>
         <executions>
             <execution>
                 <id>copy-dependencies</id>
                 <phase>prepare-package</phase>
                 <goals>
                     <goal>copy-dependencies</goal>
                 </goals>
                 <configuration>
                     <outputDirectory>${project.build.directory}/classes/lib</outputDirectory>
                 </configuration>
             </execution>
         </executions>
     </plugin>
</plugins>
```
也可以打包成功，插件maven-jar-plugin负责打包指定程序入口，插件maven-dependency-plugin负责拷贝jar包依赖。但是我们遇到了问题，打包后的jar包不能被依赖到，运行run jar 时会报出NoClassDefFoundError的错误。

```
Error: A JNI error has occurred, please check your installation and try again
Exception in thread "main" java.lang.NoClassDefFoundError: org/springframework/util/MultiValueMap
        at java.lang.Class.getDeclaredMethods0(Native Method)
        at java.lang.Class.privateGetDeclaredMethods(Unknown Source)
        at java.lang.Class.privateGetMethodRecursive(Unknown Source)
        at java.lang.Class.getMethod0(Unknown Source)
        at java.lang.Class.getMethod(Unknown Source)
        at sun.launcher.LauncherHelper.validateMainClass(Unknown Source)
        at sun.launcher.LauncherHelper.checkAndLoadMain(Unknown Source)
Caused by: java.lang.ClassNotFoundException: org.springframework.util.MultiValueMap
        at java.net.URLClassLoader.findClass(Unknown Source)
        at java.lang.ClassLoader.loadClass(Unknown Source)
        at sun.misc.Launcher$AppClassLoader.loadClass(Unknown Source)
        at java.lang.ClassLoader.loadClass(Unknown Source)
        ... 7 more
```
我们分析一下打包后的jar文件，发现里面的MANIFEST.MF文件：
![错误MF](https://github.com/apollochen123/image/blob/master/mavenRunJar/%E9%94%99%E8%AF%AFMF.jpg?raw=true)
这个问题是maven-jar-plugin 3.0.0 导致的，这里打包的路径变成了Maven repository的风格，导致ClassPath错误，找不到依赖包。<classpathLayoutType>simple</classpathLayoutType>在3.0.0版本的插件中不起效果。我们更换为3.1.0后。生效。

这个问题困扰了很久，这里总结一下maven打包可执行jar文件的几种方式：

## 第一种[maven-jar-plugin] + [maven-dependency-plugin]
pom配置：
```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <!--插件负责生成MANIFEST.MF文件，指定Main-Class，Class-Path等-->
            <artifactId>maven-jar-plugin</artifactId>
            <version>3.1.0</version>
            <configuration>
                <archive>
                    <manifest>
                        <!--是否在MANIFEST.MF中添加Class-Path，这里决定是否能找到依赖-->
                        <addClasspath>true</addClasspath>
                        <!--添加Class-Path的前缀信息，表示依赖在libs文件夹下-->
                        <classpathPrefix>libs/</classpathPrefix>
                        <!--程序入口，main class地址-->
                        <mainClass>com.yy.apollo.autopush.App</mainClass>
                        <!--Class-Path显示形式，可以有Repository形式和simple格式-->
                        <classpathLayoutType>simple</classpathLayoutType>
                    </manifest>
                </archive>
            </configuration>
        </plugin>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <!--插件自动拷贝dependency到目标仓库-->
            <artifactId>maven-dependency-plugin</artifactId>
            <version>3.0.0</version>
            <executions>
                <execution>
                    <id>copy-dependencies</id>
                    <!--package时执行本插件-->
                    <phase>package</phase>
                    <goals>
                        <goal>copy-dependencies</goal>
                    </goals>
                    <configuration>
                        <overWriteReleases>false</overWriteReleases>
                        <overWriteSnapshots>false</overWriteSnapshots>
                        <overWriteIfNewer>true</overWriteIfNewer>
                        <!--这个配置是关键项，就是配置拷贝到的目标目录-->
                        <outputDirectory>${project.build.directory}/libs</outputDirectory>
                    </configuration>
                </execution>
            </executions>
        </plugin>
```
打包后运行在target文件夹下出现jar包和libs文件夹（里面装的依赖jar）
![第一种打包图](https://github.com/apollochen123/image/blob/master/mavenRunJar/%E7%AC%AC%E4%B8%80%E7%A7%8D%E6%89%93%E5%8C%85%E5%9B%BE.jpg?raw=true)

#### 优点
1.比较简单透明，容易理解
#### 缺点
1.依赖关系不在最终的jar文件中，这意味着只有当libs文件夹对于jar才可访问和可见时，您的可执行jar文件才会运行

---------


## 第二种[maven-assembly-plugin]
pom配置
```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-assembly-plugin</artifactId>
    <version>2.5.5</version>
    <executions>
        <execution>
            <phase>package</phase>
            <goals>
                <goal>single</goal>
            </goals>
            <configuration>
                <archive>
                    <manifest>
                        <mainClass>com.yy.apollo.autopush.App</mainClass>
                    </manifest>
                </archive>
                <descriptorRefs>
                    <descriptorRef>jar-with-dependencies</descriptorRef>
                </descriptorRefs>
            </configuration>
        </execution>
    </executions>
</plugin>
```
这周方式，使用maven-assembly-plugin打包，只用配置好<mainClass>com.yy.apollo.autopush.App</mainClass>就可以了。会生成一个可以运行的jar包
![第二种](https://github.com/apollochen123/image/blob/master/mavenRunJar/%E7%AC%AC%E4%BA%8C%E7%A7%8D.jpg?raw=true)
这种方式，他同样在MANIFEST.MF文件中添加了Main-Class地址，但是依赖引入的实现，变成了把所有的jar包中的class文件拷贝出来，合并成为一个jar包。
![第二种jar包class文件](https://github.com/apollochen123/image/blob/master/mavenRunJar/%E7%AC%AC%E4%BA%8C%E7%A7%8D%E6%89%93%E5%8C%85class%E7%BB%93%E6%9E%84%E5%9B%BE.jpg?raw=true)

#### 优点
所有依赖在一个jar包当中，不再依赖lib文件夹
#### 缺点jar-with-dependencies
会生成一个带有后缀的jar，不怎么好看。同时打包所有引入的class文件，不管用到没有用到

-------------
## 第三种 [ maven-shade-plugin ]
pom配置
```xml
<plugin>
     <groupId>org.apache.maven.plugins</groupId>
     <artifactId>maven-shade-plugin</artifactId>
     <version>3.1.0</version>
     <executions>
         <execution>
             <goals>
                 <goal>shade</goal>
             </goals>
             <configuration>
                 <shadedArtifactAttached>true</shadedArtifactAttached>
                 <transformers>
                     <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                         <mainClass>com.yy.apollo.autopush.App</mainClass>
                     </transformer>
                 </transformers>
             </configuration>
         </execution>
     </executions>
 </plugin>
```
这种方式简单配置打出来的包和第二种类似，都是拷贝class文件到jar包中，但是多了一些附加功能。用到再具体学习吧.有点复杂。

----------
## 第四种 [ spring-boot-maven-plugin ]
pom配置
```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <executions>
        <execution>
            <goals>
                <goal>repackage</goal>
            </goals>
            <configuration>
                <classifier>spring-boot</classifier>
                <mainClass>com.yy.apollo.autopush.App</mainClass>
            </configuration>
        </execution>
    </executions>
</plugin>
```
注意的是，这种打包方式，打包命令为repackage 所以要使用clean package -P repackage打包
![springboot打包](https://github.com/apollochen123/image/blob/master/mavenRunJar/springboot%E6%96%B9%E5%BC%8F.jpg?raw=true)

#### 优点
并不需要你的项目带有Spring Boot application。普通项目也可以
#### 缺点
1. 需要maven3.2以上才能使用
2. 打成的包里很多无用的spring-boot依赖
