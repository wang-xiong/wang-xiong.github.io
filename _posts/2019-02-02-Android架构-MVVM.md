---
layout: post
title: "Android架构之MVVM"
subtitle: 'Android'
date:       2019-03-06
author: "Wangxiong"
header-style: text
tags:
  - Android
  - 架构
---
目前主流的三种架构模式：MVC MVP MVVM

目的是为了解决应用程序复杂的逻辑问题，通过应用相关架构模式，职责分理，不同测层次做不同的事情。

## 1. MVC(Model-View-Controller)

Model：数据模型，View：视图模型，Controller：控制器

Controller和View都依赖于Modle，VIew和Controller互相依赖

## 2. MVP(Model-VIew-Presenter)

Presenter：负责完成View和Model直接的交互，Model：业务逻辑和实体模型，View：负责VIew的控制以及与用户的交互

## 3. MVVM(Model-View-ViewModel)

MVP的一种变革，核心思想：数据模型数据双向绑定。

ViewModel会有一个叫Binder，或者是Data-binging engine的东西。MVP中Presenter负责VIew和Model之间数据同步操作全部交由Binder处理，只需要在View中声明View显示的内容是和哪一块数据绑定的，当ViewModel更新Model是，Binder会自动绑数据更新到View上，当用户操作View时，也会自动把数据更新到Model上。这种方式称为双向绑定。

Android官方推出的DataBinding便是一个双向绑定的库。

## Lifecycle

Lifecycle 组件指的是 android.arch.lifecycle 包下提供的各种类与接口，可以让开发者构建能感知其他组件（主要指Activity 、Fragment）生命周期（lifecycle-aware）的类。

## ViewModel

ViewModel用来管理和UI交互的数据，通常情况下会在Activity的onCreate()方法获取ViewModel，此后无论onCreate()调用多少次，获取到的ViewModel都是同一个实例。ViewModel是一个抽象类，中间只有一个可选择实现的方法onCleared()，该方法会在Activity或者Fragment的onDestory()被调用，用于回收资源。

![image-20190124165526595](/Users/wangxiong/Library/Application Support/typora-user-images/image-20190124165526595.png)