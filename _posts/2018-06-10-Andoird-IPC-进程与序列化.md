---
layout: post
title: "「IPC系列」一、进程与序列化"
subtitle: 'Android IPC学习'
date:       2018-06-10
author: "Wangxiong"
header-style: text
tags:
  - Android
  - IPC
---
默认情况下，Android系统中同一个应用的所有组件均运行在同一进程的相同线程中，新启动的应用组件会创建进程或者在已经存在的进程中启动并使用相同的执行线程。但是也可以指定组件在单独的进程中运行，并创造额外的线程。

## 1. 指定进程

四大组件都支持使用android:process属性指定该组件在那个进程运行，通过此方式可以指定多个组件共享同一个进程。此外改属性也可以使不同的应用组件在相同的进程中运行，但是前提是应用要共享相同的Linux用户ID并使用相同的证书签名。application元素也支持android:process属性设置默认的进程。

## 2. 进行重要程度ADJ

系统内存不足时可能会杀死关闭进程，被终止的进程的所有组件都会消耗，当这些组件再次运行时，系统将为它们重新启动进程。杀死哪些进程是根据进程的重要程度，即ADJ。吃重要程度一共分为五个级别，第一级最为重要。

### 2.1 前台进程

即用户当前操作的必需进程，如果一个进程满足以下任意一个条件，即为前提进程。

- 托管用户正在交互的Activity（已调起Activity的onResume方法）。
- 托管某个Service，该Service绑定到用户正在交互的Activity。
- 托管正在前天运行的Service（服务已经调起startForeground）。
- 托管正在执行生命周期回调的Service（onCreate、onStart、onDestroy）
- 托管正在执行onReceive方法的BroadcastReceiver

### 2.2 可见进程

没有任何的前台组件，但仍会影响用户在屏幕上所见内容的进程，

- 托管不在前台，但仍影响用户可见的Activity（已调起onPause方法）。例如在前台Activity启动了一个覆盖一部分屏幕的对话框。
- 托管绑定到可见或前台Activity的Service。

### 2.3 服务进程

正在运行的服务Service且不属于上述两种更高类别的进程。尽管服务进程与用户所见内容没有直接关联，但是它们通常在执行一些用户关系的操作，如后台播放音乐或下载数据。

### 2.4 后台进程

用户目前不可见的Activity进程（已调起Activity的onStop方法）。

### 2.5 空进程

不包含任何应用组件的进程。保留这种进程的的唯一目的是用作缓存，以缩短下次在其中运行组件所需的启动时间。 为使总体系统资源在进程缓存和底层内核缓存之间保持平衡，系统往往会终止这些进程。

## 3. 线程

应用启动时，系统会为应用创建一个主线程的执行线程，也称UI线程，负责将事件分发给相应的用户界面控件。UI线程需要处理所以的任务，如果执行耗时很长的操作会阻塞整个UI，一旦线程被阻塞，将无法分发任何事件，当主线程阻塞时间超过5秒钟，就好造成ANR。所以需要在单独的线程中执行耗时的操作。

## 4. 序列化机制

跨进程通信就是为了进行数据交换，但并不是所有的数据类型都能被传递，除了基本数据类型外，传递的数据类型必须实现序列化和反序列化，即实现了Serializable接口或者Parcelable接口的数据类型。

### 4.1 Serializable

Serializable接口是Java所提供的一个序列化接口，是一个空接口。类只需要实现此接口，则自动实现默认的序列化过程。此外，为了辅助系统完成对象的序列号和反序列化过程，还可以声明一个long类型数据serivalVersionUID。序列化使系统会把对象的信息以及serivalVersionUID一起保存到某种介质中，当反序列化时会把介质中的serivalVersionUID与类中声明的serivalVersionUID进行对比，如果相同则说明序列化的类与当前类的版本是相同的，则可以序列化成功。如果不想等则说明当前类发生变化，则会导致序列化失败。

如果没有声明serivalVersionUID，编译工具会根据当前类的结构自动生成serivalVersionUID，这样在反序列化时只有类的结构保持一致才能反序列化成功。为了当类的结构发生变化时仍然能够反序列化成功，一般手动的为serivalVersionUID指定一个值。这样即使类结构发生变化，也能最大程度地恢复数据，但是类的结构变化太大依然会导致序列化失败。

此外，类的静态成员变量属于类不属于对象，所以不会参与序列化的过程，用transient关键字标记的成员变量也不会参与序列化过程。

### 4.2 Parcelable

Parcelable是由Android系统提供的序列化接口，官方推荐使用Parcelable进行序列化操作，Bundle、Intent和Bitmap等都实现了Parcelable接口。Parcelable接口相比Serializbale更高效，但实现相对麻烦。可以使用AndroidStudio的工具Android  Parcelable code genertor来自动完成。

