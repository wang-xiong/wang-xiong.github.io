---
layout: post
title: "Apk打包流程"
subtitle: 'Android应用 APP学习'
date:       2018-07-01
author: "Wangxiong"
header-style: text
tags:
  - Android
  - APK
---

## 一、APK打包流程

![apk打包-1.png](https://upload-images.jianshu.io/upload_images/10547376-87b55c0f00eea9ed.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

1. 通过aapt打包res资源文件，生成R.java、resources.arsc和res文件。
2. AIDL工具处理.aidl文件，生成对应的java接口文件
3. 通过Java Compiler编译R.java、Java接口文件、Java源文件生成.class文件。
4. 通过dex命令，将.class文件和第三方库中的.class文件处理生成classes.dex文件。
5. 通过apkbuilder工具，将aapt生成的resources.arsc和res文件、assets文件和classes.dex一起打包生成apk。
6. 通过Jarsigner工具，对上面的apk进行debug或者release签名。
7. 如果是对APK正式签名，还需要使用zipalign工具对APK进行对齐操作，这样应用运行时会减少内存的开销。

## 二、多渠道打包及签名

- 多渠道打包，gradle配置

  配置productFlavors

- 签名：数字证书

## 三、混淆

### 3.1 proguard分为四个步骤

#### 3.1.1 压缩（shrink）

移除未使用的类、方法、字段等。

#### 3.1.2 优化（optimize）

优化字节码、简化代码操作；

#### 3.1.3 混淆（obfuscate）

使用简短的、无意义的名称重命名类名、方法名、字段等。

#### 3.1.4 预校验（preverify）

为class添加预校验信息

### 3.2 四个步骤中的常量配置

#### 3.2.1压缩（shrink）

-dontshrink
声明不进行压缩操作，默认情况下，除了-keep配置（下详）的类及其直接或间接引用到的类，都会被移除。

#### 3.2.2优化（optimize）

-dontoptimize
不对class进行优化，默认开启优化。
注意：由于优化会进行类合并、内联等多种优化，-applymapping可能无法完全应用，需使用热修复的应用，建议使用此配置关闭优化。
-optimizationpasses n  执行优化的次数，默认1次，多次能达到更好的优化效果。
-optimizations optimization_filter
优化配置，可进行字段优化、内联、类合并、代码简化、算法指令精简等操作。只进行 移除未使用的局部变量、算法指令精简
-optimizations code/removal/variable,code/simplification/arithmetic
进行除 算法指令精简、字段、类合并外的所有优化
-optimizations !code/simplification/arithmetic,!field/*,!class/merging/*

#### 3.4.3 混淆（obfuscate）

-dontobfuscate
不进行混淆，默认开启混淆。除-keep指定的类及成员外，都会被替换成简短、随机的名称，以达到混淆的目的。
-applymapping filename
根据指定的mapping映射文件进行混淆。
-obfuscationdictionary filename
指定字段、方法名的混淆字典，默认情况下使用abc等字母组合，比如根据自己的喜好指定中文、特殊字符进行混淆命名。
-classobfuscationdictionary filename
指定类名混淆字典。
-packageobfuscationdictionary filename
指定包名混淆字典。
-useuniqueclassmembernames
指定相同的混淆名称对应不同类的相同成员，不同的混淆名称对应不同的类成员。在没有指定这个选项时，不同类的不同方法都可能映射到a,b,c。
有一种情况，比如两个不同的接口，拥有相同的方法签名，在没有指定这个选项时，这两个接口的方法可能混淆成不同的名称。但如果新增一个类同时实现这两个接口，并且利用-applymapping指定之前的mapping映射文件时，这两个接口的方法必须混淆成相同的名称，这时就和之前的mapping冲突了。
在某此热修复场景下需要指定此选项。
-dontusemixedcaseclassnames
指定不使用大小写混用的类名，默认情况下混淆后的类名可能同时包含大写小字母。这在某些对大小写不敏感的系统（如windowns）上解压时，可能存在文件被相互覆盖的情况。
-keeppackagenames [package_filter]
指定不混淆指定的包名，多个包名可以用逗号分隔，可以使用? * **通配符，并且可以使用否定符（!）。
-keepattributes [attribute_filter]
指定保留属性，多个属性可以用多个-keepattributes配置，也可以用逗号分隔，可以使用? * **
通配符，并且可以使用否定符（!）。
比如，在混淆ibrary库时，应该至少keep Exceptions, InnerClasses, Signature；如果在追踪代码，还需要keep符号表；使用到注解时也需要keep。
-keepattributes Exceptions,InnerClasses,Signature
-keepattributes SourceFile,LineNumberTable
-keepattributes *Annotation*
-keepparameternames
指定keep已经被keep的方法的参数类型和参数名称，在混淆library库时非常有用，可供IDE帮助用户进行信息提示和代码自动填充。

#### 3.4.4 预校验（preverify）

-dontpreverify
指定不对class进行预校验，默认情况下，在编译版本为micro或者1.6或更高版本时是开启的。但编译成Android版本时，预校验是不必须的，配置这个选项可以节省一点编译时间。
（Android会把class编译成dex，并对dex文件进行校验，对class进行预校验是多余的。）

### 3.3 Keep配置

## 五、APK加固

为了防止应用程序被恶意破解，植入病毒，修改代码。所以需要对源APK进行加密，即加固。