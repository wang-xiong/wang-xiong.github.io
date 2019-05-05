---
layout: post
title: "【Java虚拟机系列】四、JVM、Dalik、ART"
subtitle: 'Java虚拟机原理学习'
date:       2018-04-06
author: "Wangxiong"
header-style: text
tags:
  - Java
  - JVM
---

## 1. JVM与Dalik

| **JVM**                                                      | Dalivk                                                       |
| :----------------------------------------------------------- | ------------------------------------------------------------ |
| Java虚拟机基于栈，基于栈的机器必须使用指令来载入和操作栈上的数据 | Dalivk虚拟机基于寄存器                                       |
| Java虚拟机运行的是Java字节码。（java类会被编译成.class文件，打包到jar中，java虚拟机会从.class文件或者.jar获取相应的字节码） | Dalivk运行的是.dex字节码格式。（java类被编译成.class文件后，会通过一个dx工具将所有的.class文件打包成.dex文件，然后dalivk虚拟机会从其中读取指令和数据） |
|                                                              | 一个应用对应一个Dalivk虚拟机实例，独立运行                   |
| Jvm在运行时会为每个类装载字节码                              | Dalvik程序只包含dex文件，dex文件包含所有的类                 |

DVM执行的是.dex文件，jvm执行的是.class文件，dvm是基于寄存器的虚拟机，jvm是基于虚拟栈的虚拟机，寄存器的存取速度比栈快的多，dvm可以根据硬件进行最大优化，比较适合移动设备。.class文件会存在很多冗余信息，dex工具会去掉冗余信息，并把所有.class文件整合到.dex文件中，减少I/O操作，提高类的查找速度。

## 2. DVM与ART

Dalvik运行环境使用JIT(just-in-time)来进行转译，每次运行的时候都需要通过JIT转换为机器码，这会影响应用的运行效率。

ART则是使用AOT(Ahead-of-time)进行处理，并在用程序安装完毕时进行应预先的基础性编译，这样就减去了JIT运行时机器码的转化时间，应用的启动和执行都变得更加快速。



