---
title: Eclipse with lombok
date: 2018-03-23 10:47:51
tags:
 -Plug-in
categories: Plug-in
thumbnail: https://raw.githubusercontent.com/apollochen123/image/master/%E9%BB%98%E8%AE%A42.jpg
---

# Eclipse with lombok

-----------------
## What is lombok
 官方介绍：
 > Project Lombok is a java library that automatically plugs into your editor and build tools, spicing up your java.
Never write another getter or equals method again. Early access to future java features such as val, and much more.

其实lombok就是一个常用java代码补全工具，比如setter and getter方法，还有construtor。或者是每个类的log获取。都可以使用lombok自动补全

--------------
## How to use ？

### demo
定义一个pojo
```
public class Mountain{
    private String name;
    private double longitude;
    private String country;
}
```
要使用这个对象,必须还要写一些getter和setter方法,可能还要写一个构造器、equals方法、或者hash方法.这些方法很冗长而且没有技术含量,我们叫它样板式代码.
   lombok的主要作用是通过一些注解，消除样板式代码，像这样：
```
@Data
public class Mountain{
    private String name;
    private double longitude;
    private String country;
}
```
然后就可以看到：
![lombok图片](https://raw.githubusercontent.com/apollochen123/image/master/lombok.jpg)

---------------------
## How to install：
1. 访问lombok官方网站下载jar包：https://projectlombok.org/download
2. java -jar 刚刚下载的jar包
3. 在弹出窗口按照指引 install
