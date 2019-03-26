---
layout: post
title: "「IPC系列」三、常用的IPC"
subtitle: 'Android IPC学习'
date:       2018-06-12
author: "Wangxiong"
header-style: text
tags:
  - Android
  - IPC
---
Android系统IPC进程间的通信方式有多种，使用Bundle、使用共享文件、AIDL、Messenger、ContentProvider、Socket

## 1. 使用Bundle

Android中的三大组件Activity，Service，Receiver可以在Bundle添加数据，通过Intent传递数据，因为Bundle实现了Serializable接口。但是有大小限制，不能超过1M。

## 2. 使用共享文件

并发读写过程容易出现问题

## 3. 使用AIDL

### 3.1 AIDL概述

AIDL(Android Interface Definition Language)Android接口定义语言，是用于定义服务端和客户端的通信接口的一张语言描述，可以用来生成IPC的代码。实质是通过定义AIDL文件，根据AIDL文件生成一个IInterface的实例代码。

### 3.2 AIDL使用

1. 进程间通信的数据序列化
2. 服务端定义AIDL接口文件包括传递数据的AIDL文件，向外暴露调用的AIDL接口，服务端实现暴露的AIDL接口。
3. 客户端拷贝服务端的所有AIDL文件，保持目录结构一致。客户端绑定服务端Service，进行调用。

### 3.3 注册回调函数

在声明回调接口AIDL文件，并在暴露方法的AIDL文件添加注册和解除注册方法。在服务端中实现，在客户端中注册。但是Binder把客户端的对象序列化传给服务端，但是对服务端回调函数经过序列化后，前后两个回调是两个完全不相关的对象。

为了能够正确无误的注册和反注册回调函数，系统提供了RemoteCallBackList类，RemoteCallBackList类内部有一个ArrayMap用于保存所有的AIDL回调接口。

```java
ArrayMap<IBinder, Callback> mCallbacks  = new ArrayMap<IBinder, Callback>();
```

其中CallBack封装了真正的远避免程回调函数，因为即使回掉函数经过序列化和反序列化后生成不同的对象，但这些对象的底层Binder对象是同一个。所以可以遍历RemoteCallbackList的方式删除注册的回调函数。此外，当客户端进程终止后，RemoteCallbackList 会自动移除客户端所注册的回调接口。而且 RemoteCallbackList 内部自动实现了线程同步的功能，所以我们使用它来注册和解注册时，不需要进行线程同步。

### 3.4 避免ANR

客户端在调用远程方法时，被调用的方法是运行在服务端的Binder线程池中，同时客户端线程会被挂起，如果服务端执行方法耗时，就会导致客户端线程阻塞，可能造成ANR，所以如果确认远程方法是耗时的，就要避免在UI线程中调用远程方法。同理，服务端也不能调用客户端的耗时方法。

### 3.5 Binder连接池

服务端一对多，使用Bindler连接池管理所有的AIDL，服务端只创建一个Service，每个客户端请求连接时，带上自己的唯一标识，服务端根据标识返回对应的Binder给客户端。