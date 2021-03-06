---
layout: post
title: "Apk安全机制"
subtitle: 'Android应用 App性能 Apk'
date:       2018-07-03
author: "Wangxiong"
header-style: text
tags:
  - Android
  - APK
---
Android中的安全机制，一般有五道防线

## 1. 第一道防线

代码混淆proguard，应用通过混淆代码，放在被其他恶意反编译识别。

## 2. 第二防线

应用权限控制，在AndroidMainifest文件的权限声明，Android应用默认情况未关联任何权限，如果要使用系统其他功能，需要声明权限，申明权限的方式是在AndroidManifest文件中，使用*<uses-permission>*进行声明。如果申明的是普通权限，则系统自动授予这些权限，如果是危险权限，并且应用运行的Android版本在6.0及以上，并且设置了targetSdkVersion是23或者更高的版本，则需要应用再运行时动态申请这些权限。

## 3. 第三道防线

应用签名机制-数字证书

## 4. 第四道防线

Linux内核层安全机制-Uid、访问权限控制，Android系统是一种多用户的Linux系统，其中每个应用都是一个不同的用户。默认情况下系统会为每个应用分配一个唯一的Linux用户ID，系统为应用中的所以文件设置权限，使得只有分配给该应用的用户ID才能访问这些文件。

## 5. 第五道防线

Android虚拟机沙箱机制-沙箱隔离。Android的App运行在虚拟机中，每个app运行在单独的app进程中。每个应用的进程都有自己的虚拟机，因此应用代码是在与其他应用隔离的环境中运行。

