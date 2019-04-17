---
layout: post
title: "Apk打包编译配置"
subtitle: 'Android应用 App性能 Apk'
date:       2018-07-02
author: "Wangxiong"
header-style: text
tags:
  - Android
  - APK
---

## 1. compileSdkVersion，miniSdkVersion，targetSdkVersion

minSdkVersion<=targetSdkVersion<=compileSdkVersion

### 1.1 compileSdkVersion:

编译app时使用的sdk版本，只是编译时选择的版本，不涉及运行时的行为。
好处：预编译时可以提示警告，避免使用废弃的api

### 1.2 miniSdkVersion：

程序运行的最低sdk，系统低于这个sdk时无法安装
好处：使用的方法是大于miniSdkVersion才有的，编译时会提示，做兼容处理。

### 1.3 targetSdkVersion:

targetSdk提供向前兼容的，表现形式按照targetSdk。好处：1、提供向下兼容，2、确认app的表现形式、3、允许适应新的变化行为可以使用新的api（编译已经使用compileSdkVersion）。

- 系统版本大于targetSdkVersion时：
  按照targetSdkVersion具有的表现能力处理，即targetSdkVersion是确定表现形式的。并且也做到了向下兼容。
- 系统版本等于targetSdkVersion时：
  不需要做兼容性检查判断了
- 系统版本小于targetSdkVersion时：
  代码处理，类似minSdkVersion

## 2. buildToolsVersion

buildToolsVersion是表明的你构建工具的版本号是多少，规则是可以用高的构建工具来构建低版本Sdk的工程。

## 3. CPU架构与ABI

ABI(Application Binary Interface)：应用程序二进制接口，描述可应用程序和操作系统之间，一个应用和它的库之间，或者应用组成部分之间的低接口。每一种架构都关联着一个相应的ABI，Android系统目前支持以下七种不同的CPU架构。

| CPU架构 | Zygote进程         | 时间     |
| ------- | ------------------ | -------- |
| ARMv5   | 32                 | 2010年起 |
| ARMv7   | 32(目前大多数机型) | 2010     |
| x86     | 32                 | 2011     |
| MIPS    | 32                 | 2012     |
| ARMv8   | 64(高端机型)       | 2014     |
| MIPS64  | 64                 | 2014     |
| x86_64  | 64                 | 2014     |

| ABI（横向）和cpu（纵向） | armeabi | armeabi-v7a | arm64-v8a | mips | mips64 | x86  | x86_64 |
| ------------------------ | ------- | ----------- | --------- | ---- | ------ | ---- | ------ |
| ARMv5                    | 支持    |             |           |      |        |      |        |
| ARMv7                    | 支持    | 支持        |           |      |        |      |        |
| ARMv8                    | 支持    | 支持        | 支持      |      |        |      |        |
| MIPS                     |         |             |           | 支持 |        |      |        |
| MIPS64                   |         |             |           | 支持 | 支持   |      |        |
| x86                      | 支持    | 支持        |           |      |        | 支持 |        |
| x86_64                   | 支持    |             |           |      |        | 支持 | 支持   |

4.指定Gradle使用的Jdk

可以在gradle.properties添加，指定编译使用的jdk

```java
org.gradle.java.home=/Applications/Android Studio.app/Contents/jre/jdk/Contents/Home
```