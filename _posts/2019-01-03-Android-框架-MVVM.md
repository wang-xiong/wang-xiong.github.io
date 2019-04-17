---
layout: post
title: "Android框架之MVVM"
subtitle: 'Android'
date:       2019-01-03
author: "Wangxiong"
header-style: text
tags:
  - Android
  - MVVM
  - 应用程序框架
---
Android目前主流的三种APP架构模式：MVC MVP MVVM，目的是为了解决应用程序复杂的逻辑问题，通过应用相关架构模式，职责分理，不同测层次做不同的事情。

## 1. MVC(Model-View-Controller)

- Model：数据模型，
- View：视图模型，对应于xml布局
- Controller：控制器，对应于Activity业务逻辑，数据处理和UI处理

Controller和View都依赖于Modle，VIew和Controller互相依赖

## 2. MVP(Model-VIew-Presenter)

- Model: 依然是实体模型
- View: 对应于Activity和xml，负责View的绘制以及与用户交互
- Presenter：负责完成View和Model直接的交互，Model：业务逻辑和实体模型，View：负责VIew的控制以及与用户的交互


## 3. MVVM(Model-View-ViewModel) 

- Model: 实体模型
- View: 对应于Activity和xml，负责View的绘制以及与用户交互
- ViewModel: 负责完成View于Model间的交互,负责业务逻辑

MVP的一种变革，核心思想：数据模型数据双向绑定，ViewModel会有一个叫Binder，或者是Data-binging engine的东西。MVP中Presenter负责VIew和Model之间数据同步操作全部交由Binder处理，只需要在View中声明View显示的内容是和哪一块数据绑定的，当ViewModel更新Model是，Binder会自动绑数据更新到View上，当用户操作View时，也会自动把数据更新到Model上。这种方式称为双向绑定。Android官方推出的DataBinding便是一个双向绑定的库。

## 4. DataBinding

DataBinding是一种框架，mvvm是一种模式，DataBinding是一个数据和ui绑定的框架，是mvvm模式的一个工具，ViewModel和View可以通过DataBinding来实现单向绑定和双向绑定，这套UI和数据之间的动态监听和动态更新的框架Google已经帮我们做好了。
在MVVM模式中ViewModel和View是用绑定关系来实现的，所以有了DataBinding使我们构建Android MVVM应用程序成为可能。使用DataBinding优缺点：

### 4.1 优点

1. 双向绑定技术，当Model变化时，View-Model会自动更新，View也会自动变化。很好做到数据的一致性，不用担心，在模块的这一块数据是这个值，在另一块就是另一个值了。所以 MVVM模式有些时候又被称作：model-view-binder模式。
2. View的功能进一步的强化，具有控制的部分功能，若想无限增强它的功能，甚至控制器的全部功几乎都可以迁移到各个View上（不过这样不可取，那样View干了不属于它职责范围的事情）。View可以像控制器一样具有自己的View-Model.
3. 由于控制器的功能大都移动到View上处理，大大的对控制器进行了瘦身。不用再为看到庞大的控制器逻辑而发愁了。
4. 可以对View或ViewController的数据处理部分抽象出来一个函数处理model。这样它们专职页面布局和页面跳转，它们必然更一步的简化。

### 4.2 缺点

1. 故障难定位:数据绑定使得bug很难被调试，界面异常了，有可能是你的view代码有bug，业有可能是model代码有问题。数据绑定使得一个位置的bug被快速传递到别的位置，要定位原始问题就比较难了
2. 内存占用大：一个大的模块中，model也会很大，虽然使用方便保证了数据的一致性，当长期持有，不释放内存，就造成了花费更多的内存
3. 数据的绑定不利于代码的重用，数据双向的绑定技术，让你的一个view都绑定了一个model，不同模块的model都不同，就不能简单的重用view