---
layout: post
title: "「Gradle系列」Gradle Wrapper"
subtitle: 'Java'
date:       2018-12-08
author: "Wangxiong"
header-style: text
tags:
  - Java
  - Gradle
---
## 1. Gradle Wrapper简介

Gradle Wrapper是Gradle的包装器，Gradle有多个版本，不同版本Gradle对构建出的apk会造成不确定性，为了解决这个问题引出了Gradle Wrapper，Gradle Wrapper是一个脚本，可以让IDE在没有安装Gradle的情况下运行Gradle构建。Gradle Wrapper可以指定Gradle的版本保证了项目所构建的Gradle版本一致。当使用Gradle Wrapper启动Gradle时，如果指定的Gradle版本没有被下载关联，就会从Gradle官方仓库下载该版本Gradle到用户本地，进行解包并执行批处理文件。后续构建运行都会重用这个下载好的Gradle。

## 2. 构建Gradle Wrapper

安装并配置好Gradle环境后，就可以执行Gradle内置的wrapper Task，执行wrapper任务就可以在当前目录下生成Gradle Wrapper的目录文件。执行命令：gradle wrapper。当前目录下生成的文件如下：

```xml
 ├─ gradle 
 │   └── wrapper 
 │       └── gradle-wrapper.jar 
 │       └── gradle-wrapper.properties 
 ├── gradlew 
 └── gradlew.bat
```

- gradle-wrapper.jar：包含Gradle运行时的逻辑代码。
- gradle-wrapper.properties：负责配置包装器运行时的属性文件，包括Gradle的版本信息属性。
- gradlew：Linux平台下，用来执行Gradle命令的包装器脚本。
- gradlew.bat：Window平台下，用来执行命令的包装器脚本。

生成Gradle Wrapper包装器可以执行gradle wrapper，gradle wrapper命令后也可以加参数，如gradle wrapper –gradle-version 4.2.1 –distribution-type all。

- –gradle-version：用于下载和执行指定的gradle版本。
- –distribution-type：指定下载Gradle发行版的类型，可用选项有bin和all，默认值是bin代表Gradle发行版，只包含运行时，但不包含源码和文档。
- –gradle-distribution-url： 指定下载Gradle发行版的完整URL地址。
- –gradle-distribution-sha256-sum：使用的SHA 256散列和验证下载的Gradle发行版。

## 3. Gradle Wrapper配置

gradle-wrapper.properties是Gradle Wrapper的属性文件，用来配置Gradle Wrapper，常见如下

```groovy
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
distributionUrl=https\://services.gradle.org/distributions/gradle-4.6-all.zip
```

- distributionBase：Gradle解包后存储的主目录。
- distributionPath：distributionBase指定目录的子目录。distributionBase+distributionPath就是Gradle解包后的存放位置。
- distributionUrl：Gradle发行版压缩包的下载地址，可以用国内镜像地址替代，也可以设置为本地项目目录。
- zipStoreBase：Gradle压缩包存储主目录。
- zipStorePath：zipStoreBase指定目录的子目录。zipStoreBase+zipStorePath就是Gradle压缩包的存放位置。

## 4. Gradle Wrapper使用

使用Gradle Wrapper需要使用gradlew和gradlew.bat脚本。比如Window平台使用gradlew.bat task命令执行任务，Linux平台使用gradlew test执行命令。

## 5. 升级Gradle Wrapper

升级Gradle Wrapper有两种方式，一种是直接修改Gradle Wrapper的属性问题的distributionUrl属性，另一种是通用运行wrapper任务 ，如执行`gradlew wrapper –gradle-version 5.1.1`升级到5.1.1版本。使用`gradlew -v`检测当前的Gradle Wrapper版本。

## 6. 自定义Gradle Wrapper

Gradle已经内置了Wrapper Task，因此构建Gradle Wrapper会生成Gradle Wrapper的属性文件，这个属性文件可以通过自定义继承Wrapper Task来设置。如下所示：

```groovy
task wrapper(type: Wrapper) {
    gradleVersion = '5.1.1'
    distributionUrl = '../../gradle-4.2.1-bin.zip' 
    distributionPath=wrapper/dists
}
```

