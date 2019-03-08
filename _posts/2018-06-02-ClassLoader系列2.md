---
layout: post
title: "【ClassLoader系列】一、Android中的ClassLoader"
subtitle: 'ClassLoader学习，总结'
date:       2018-06-02
author: "Wangxiong"
header-style: text
tags:
  - Java
  - Android
  - ClassLoader
---
# Android中的ClassLoader

Android中的ClassLoader和Java中的ClassLoader是不同的。Java中的ClassLoader可以加载jar文件和class文件（本质是加载class文件），但是Android中无论是DVM还ART它们加载的不是class文件，而是dex文件。

## 1.Android中的ClassLoade的类型

Android中的ClassLoader和Java中的ClassLoader类似，分为两种：系统ClassLoader和自定义ClassLoader。系统的ClassLoader主要三种分别是BootClassLoader、PathClassLoader和DexClassLoader。

### 1.1BootClassLoader

Android系统启动时会使用BootClasssLoader来预加载常用的类。由Java代码实现，是ClassLoader的内部类，继承自ClassLoader，BootClasssLoader是个单例，访问修饰符是默认的只能在同一个包内访问，因此应用程序中无法直接调用。

### 1.2.DexClassLoader

DexClassLoader可以加载dex文件以及包含dex的压缩文件（APK，jar文件）。

```java
public class DexClassLoader extends BaseDexClassLoader {
    public DexClassLoader(String dexPath, String optimizedDirectory, String librarySearchPath, ClassLoader parent) {
        super((String)null, (File)null, (String)null, (ClassLoader)null);
        throw new RuntimeException("Stub!");
    }
}
```

DexClassLoader的构造方法

dexPath：dex文件的路径集成，多个路径用分隔符分割，默认是":"。

optimizedDirectory：解压的dex文件存储路径，这个路径必须是一个内部存储路径，一般使用当前应用的私有路径：/data/data/<Package Name>/...。

librarySearchPath：包含 C/C++ 库的路径集合，多个路径用文件分隔符分隔分割，可以为null。

parent：父加载器。

### 1.3PathClassLoader

Android系统使用PathClassLoader来加载系统和应用程序的类，PathClassLoader继承自BaseDexClassLoader，构造参数中没有optimizedDirectory，因为默认了参数optimizedDirectory值为：/data/dalivk-cahe，通常用来鸡杂已经安装的apk的dex文件（安装的apk的dex文件会存储在/data/dalivk-cahe中）。

## 2.ClassLoader的继承关系

和Java中的ClassLoader一样，虽然系统所提供的类加载器主要有3种类型，但是系统提供的ClassLoader相关类却不只3个。ClassLoader的继承关系如下图所示。

![image-20190308131412575](/Users/wangxiong/Library/Application Support/typora-user-images/image-20190308131412575.png)

- ClassLoader：一个抽象类，定义了ClassLoader的主要功能。BootClassLoader是它的内部类。
- SecureClassLoader：和JDK8中的SecureClassLoader代码是一样的，继承了ClassLoader，拓展了ClassLoader类加入了权限方面的功能，加强了ClassLoader的安全性。
- URLClassLoader类个JDK8中URLClassLoader类的代码是一样的，它继承自SecureClassLoader，用来通过URI路径从Jar文件和文件夹中加载类和资源。
- BaseDexClassLoader继承自ClassLoader，是ClassLoader的具体实现类。
- InMemoryClassLoader是Android8.0新增的类加载器，继承自BaseDexClassLoader，用来加载内存中的dex文件。
- PathClassLoader继承自ClassLoader，具体功能如上边所述
- DexClassLoader 继承自ClassLoader，具体功能如上边所述

## 3.BootClassLoader的创建

BootClassLoader是在Zygote进程的入口方法中创建的。

## 4.PatchClassLoader的创建

PatchClassLoader是在Zygote进程创建SystemServer进程是创建的。

zygote进程通过forkSystemServer方法fork自身创建子进程，当forkSystemServer方法返回的pid等于0，说明当前代码是在新创建SystemServer进程。

参考资料

[ClassLoader系列](http://liuwangshu.cn/application/classloader/2-android-classloader.html)
[Android动态加载之ClassLoader详解](http://www.jianshu.com/p/a620e368389a)
[热修复入门：Android 中的 ClassLoader](http://www.jianshu.com/p/96a72d1a7974)
[浅析dex文件加载机制](http://www.cnblogs.com/lanrenxinxin/p/4712224.html)

