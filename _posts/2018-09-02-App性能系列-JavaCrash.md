---
layout: post
title: "「App性能系列」Java Crash"
subtitle: 'Android应用 APP性能学习'
date:       2018-09-02
author: "Wangxiong"
header-style: text
tags:
  - Android
  - APP性能
---
日常开发中难免遇到Crash问题，随着APP的不断迭代，碎片化增多，开发过程的代码质量，Crash问题必然增多，常见的crash有：空指针、角标越界、类型转换异常等问题。

## 1. NullPointerException

NullPointerException问题是最常见的问题之一了，一般常见的有以下常见 ：

- 对象本身没有被初始化；
- 对象初始化了但是被置空或者被回收；
- 函数传入的参数本身就为空；
- Json解析的数据为空。

针对NullPointerException问题的需要多思考，传入的数据有可能为空时，对数据进行判空处理，在Activity这些可能被回收的场景，需要判断Activity/Fragment是否被销毁等。

## 2. IndexOutOfBoundsException

数组角标越界异常，一般常见的有以下场景：

- 经常在对ListView操作，因为外部持有了Adapter里的数据引用，当外部引用对数据进行了修改，但没有及时调用notifyDataSetChanged就会造成 crash，同时也可能出现The content of the adapter has changed but ListView did not receive a notification类型的crash问题。
- 多线程下对集合的操作也会造成这样的场景，在多线程下使用线程安全的集合代替。

## 3. TransactionTooLargeException

Intent传递大数据会触发TransactionTooLargeException异常，原因在于：Intent无法传递大数据是因为其内部使用了Binder通信机制，Binder事务缓冲区限制了传递数据的大小；Binder事务缓冲区的大小限定为1MB，是进程共享的。解决方法可以使用EventBus的黏性事件来解决。

