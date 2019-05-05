---
layout: post
title: "Android应用程序之Fragment"
subtitle: 'Android应用程序学习'
date:       2018-06-11
author: "Wangxiong"
header-style: text
tags:
  - Android
  - Android应用程序
---
## 1. Fragment简介

Fragment(碎片)是一种可以嵌入在活动当中的UI片段，能够让程序更加合理和充分地利用大屏幕空间，和活动类似同样包含布局、同样有自己的生命周期。

## 2. Fragment的生命周期

onAttach、onCreate、onCreateView、onActivityCreated、onStart、onResume、onPause onStop、onDestroyView、onDestroy、onDetach。

## 3. Fragment的数据传递

- 通过setArgument方法传递数据。
- 在Fragment中直接调用Activity的方法
- 在Fragment申明接口，Activity实现接口（Fragment的onAttach方法中得到的Activity就是实现接口的实例）
- 使用ViewModel共享统一个Activity下Fragment数据。

## 4. FragmentManager

Activity中有个FragmentManager，其内部维护fragment队列，以及fragment事务的回退栈。在Fragment被创建、并由FragmentManager管理时，FragmentManager就把它放入自己维护的fragment队列中。

## 5. FragmentTransaction

知道了FragmentManger可以管理和维护Fragment，那么FragmentManager是直接去绑定Fragment然后把它set进自己的队列中吗？不是的，而是用FragmentTransaction（Fragment事务），FragmentManager调用beginTransaction()方法返回一个新建的事务，用于记录对于Fragment的add、replace等操作，最终将事务commit回FragmentManager，才开始启动执行事务的内容，实现真正的Fragment显示。

## 6. setUserVisibleHint实现Fragment懒加载

[参考](https://www.cnblogs.com/leevey/p/5678037.html)