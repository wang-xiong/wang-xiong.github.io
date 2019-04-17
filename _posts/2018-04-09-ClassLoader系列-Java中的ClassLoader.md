---
layout: post
title: "「ClassLoader系列」一、Java中的ClassLoader"
subtitle: 'ClassLoader学习，总结'
date:       2018-04-09
author: "Wangxiong"
header-style: text
tags:
  - Java
  - ClassLoader
---

## 1. Java中的ClassLoader

类加载器的功能就是查找Class文件到Java虚拟机中。

## 2. Java中的ClassLoader类型

Java的类加载器主要有两种类型，系统类加载器和自定义类加载器。

系统类加载器：Bootstrap ClassLoader、Extendsions ClassLoader、APP ClassLoader。

### 2.1 Bootstap ClassLoader

用C++代码实现的类加载器，用于加载Java虚拟机运行所需要的系统类。这些系统类默认都在$JAVA_HOME/jre/lib目录中，Java虚拟机的启动就是通过Bootstap ClassLoader创建一个初始类完成的。Bootstap ClassLoader并不继承java.lang.ClassLoader。

### 3.2 Extensions ClassLoader

用于加载Java的拓展类，拓展类的jar包一般都会放在$JAVA_HOME/jre/lib/ext目录下。用于提供除系统类之外的额外功能。

### 2.3 App ClassLoader

负责加载当前应用程序ClassPath目录下的所有jar和Class文件。

自定义的类加载器

除了系统提供的类加载器，还可以自定义类加载器。通过继承java.lang.ClassLoader类来实现自己的类加载器，除了Bootstrap ClassLoader，Extensions ClassLoader和App ClassLoader也继承了java.lang.ClassLoader类。

## 3. ClassLoader的继承关系

ClassLoader是一个抽象类，定义了ClassLoader的主要功能。

SecureClassLoader继承了ClassLoader，但SecureClassLoader并不是SecureClassLoader的实现类，而是拓展了ClassLoader类加入了权限方面的功能，加强了ClassLoader的安全性。

URLClassLoader继承了SecureClassLoader，用来通过URL路径从jar文件和文件夹中加载类和资源。

ExtClassLoader和AppClassLoader都继承自URLClassLoader，它们都是Launcher的内部类，Launcher是Java虚拟机的入口应用。

## 4. 双亲委托机制

类加载器查找Class采用的是双亲委托机制，即首先判断该Class是否已经加载，如果没有则委托给父类加载器进行查找，依次进行递归直到最顶层的Bootstrap ClasssLoader，如果Bootstrap ClassLoader找到该Class，就直接返回，如果没有，则继续依次向下查找，最后交给自身去查找。如下图所示

![classloader-1.png](https://upload-images.jianshu.io/upload_images/10547376-81dff278cd367af2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**双亲委托机制好处：**

1.避免重复加载类，如果已经加载过一次Class，就不需要再加载，而是直接从缓存中直接读取。

2.更加安全，如果不使用双亲委托模式，就可以自定义一个String类来替代系统的String类，这显然会造成安全隐患，采用双亲委托模式会使得系统的String类在Java虚拟机启动时就被加载，也就无法自定义String类来替代系统的String类，除非我们修改类加载器搜索类的默认算法。还有一点，只有两个类名一致并且被同一个类加载器加载的类，Java虚拟机才会认为它们是同一个类，想要骗过Java虚拟机显然不会那么容易。

## 5. 自定义ClassLoader

1.编写测试类ClassLoaderTest.java，放到D:/lib下

2.执行Javac ClassLoaderTest.java生成ClassLoaderTest.class文件

3.编写自定义ClassLoader，复写findClass方法，并在findClass方法中调用defineClass方法。

