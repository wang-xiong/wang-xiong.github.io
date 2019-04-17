---
layout: post
title: "「Java基础」String类"
subtitle: 'Java基础回顾复习'
date:       2018-03-21
author: "Wangxiong"
header-style: text
tags:
  - Java
---
Java中的String类，java.lang.String，String对象表示一个字符串，一旦创建一个String对象，就不能改变其值，因为String对象是常量。

## 1.创建String的方式：

### 1.1 将一个字符串赋值给一个String的引用变量

```java
String s="Java";
```

### 1.2 new关键字构建一个String对象

```java
String s=new String("Java");
```

### 1.3 两种方式的区别

两种创建方式的含义不同，直接使用字符串赋值，得到的String对象并不总是最新的，如果该字符串之前已经创建过程，该对象可能来自于一个缓冲池。如果使用new的方式创建，JVM每次都会创建String的一个新的实例。因此使用字符串赋值的方式更好，可以节省了JVM需要构建新的实例的CPU周期。

```java
//1.输出true，==比较的是两个引用变量引用的地址是否相同，所以返回true		
String s1="Java";
String s2="Java";
System.out.println(s1==s2);
//2.输出false，因为new关键字每次都会创建一个新的String对象，所以返回false
String s1=new String("Java");
String s2=new String("Java");
//输出false
System.out.println(s1==s2);
//3.正常情况比较两个对象的值是否相同，使用equals方法
String s1="Java";
String s2=new String("Java");
String s3="C";
System.out.println(s1.equals(s2));//输出true
System.out.println(s1.equals(s3));//输出false
```

## 2. StringBuffer和StringBuilder

因为String对象是不可以变的，因此添加或者删除字符时，每次都会创建一个新的String对象。所以最好使用StringBuffer和StringBuilder操作，操作完成后再转换成String对象。

StringBuffer的方式是同步的，适合多线程下使用，同步的代价就是性能下降。

StringBuilder的方式是异步的，性能高，线程不安全。