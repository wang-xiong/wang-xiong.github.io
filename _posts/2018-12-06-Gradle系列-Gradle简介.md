---
layout: post
title: "Gradle系列"
subtitle: 'Android'
date:       2018-12-05
author: "Wangxiong"
header-style: text
tags:
  - Android
  - Grandle
---
## 1. Gradle简介

Gradle是一款构建工具，是Android官方推荐的Android构建工具，Gradle使用的语言是Grovvy语言，Gradle目前已经应用到Android各个技术体系中，比如系统构建、插件化、组件化、热修复等。

## 2. 配置Gradle环境

安装Gradle前要确保系统已经配置好JDK的环境，要求JDK的版本在1.7或更高。两种安装方式：

1. 保管理方式安装，Window平台的[Chocolatey](https://chocolatey.org/)、[Scoop](https://scoop.sh/)，Mac平台的[MacPortsl](https://www.macports.org/)、Homebrew等等。可参考[官方文档](https://gradle.org/install/)。
2. 手动安装：在<https://gradle.org/releases/> 中下载你想要Gradle版本的binary-only。

安装完成后需要配置环境变量，针对Mac平台安装步骤如下

```xml
mac配置环境变量方法
1、cd~:cd到home目录
2、touch .bash_profile
3、open -e .bash_profile 打开此文件
4、添加路径
5、source .bash_profile 保存生效
注意：
1.空格加转义字符\
2.ls -l查看权限
3.chmod +x gradle.bat 和 chmod +x gradle 添加x权限sudo chmod +x gradlew 
```

添加后的文件类型如下

```x&#39;m
export ANDROID_HOME=/Users/baidu/Library/Android/sdk
export PATH=${PATH}:${ANDROID_HOME}/tools
export PATH=${PATH}:${ANDROID_HOME}/platform-tools
export GRADLE_HOME=/Applications/Android\ Studio.app/Contents/gradle/gradle-4.6
export PATH=${PATH}:${GRADLE_HOME}/bin
export JAVA_HOME="/Library/Internet Plug-Ins/JavaAppletPlugin.plugin/Contents/Home"
export PATH=${JAVA_HOME}/bin:$PATH
export PATH=$PATH:/Users/baidu/flutter/bin
```

## 3. Gradle的任务task

Gradle默认构建脚本文件是build.gradle。配置完环境后，执行gradle命令就好在当前目录下寻找build.gradle文件。项目构建比较复杂，为了使用各种开发语言的开发者都能够快速的构建项目，专家们开发出了Gradle这个基于Groovy的DSL，DSL(Domain Specifc Language)意为领域特定语言，只用于某个特定的领域。我们只要按照Groovy的DSL语法来写，就可以轻松构建项目。task（任务）、action（动作）是Grandle中两个重要的元素

### 3.1 task简介

一个Task代表一个构建工作的原子操作，每个Project都包含了一系列的Task。比如Java源码编译Task、资源编译Task，Jni编译Task，lint坚持Task，打包生成Apk的Task，签名Task。Gradle执行Task时分为两个阶段，首先是配置阶段，然后是实际执行阶段。配置阶段会扫描整个build.gradle文件，配置Project和Task。[Task的Api](<https://docs.gradle.org/current/dsl/org.gradle.api.Task.html>)。执行Task的方法：

- 在Gradle图形化界面执行。

- 在Terminal窗口执行：./gradlew helloTask

  （1）./gradlew tasks 查看当前Project所有的task

  （2）./gradlew properties 查看当前Project的properties

```java
Task {
    String TASK_NAME = "name";
    String TASK_DESCRIPTION = "description";
    String TASK_GROUP = "group";
    String TASK_TYPE = "type";
    String TASK_DEPENDS_ON = "dependsOn";
    String TASK_OVERWRITE = "overwrite";
    String TASK_ACTION = "action";
}
//常用的可配置参数:name,group,description,type,dependsOn
name：task的名称
group：task所在的分组，group相同会被分在一组，默认是other
type: 新建的Task对象从那个基类派生出来的
dependsOn：task依赖与哪一个task
```

### 3.2 创建Task

```groovy
(1)
task helloTask() {
    println("hello my Task")
}

(2)
this.tasks.create(name: 'helloTask1')
helloTask1.group = 'wx'

(3)
task helloTask(group: "wx", description: 'task test') {
    println("hello my Task")
}

(4)
task helloTask() {
    setGroup('wx')
    setDescription('task test')
    println("hello my Task")
}

(5)
task copyFileTask(type: Copy) {
    setGroup('wx')
    from "src/main/AndroidManifest.xml"
    into 'src/main/test'
    rename { String fileName ->
        fileName = "AndroidManifestTest.xml"
    }
    println("copyFileTask final")
}
```

### 3.3 task中的action

doFirst，doLast是task中的常见action，使用方式如下

```groovy
helloTask.doFirst {
    println("hello my Task doFirst")
}
helloTask.doLast {
    println("hello my Task doLast")
}
helloTask << {
    println("hello my Task <<")
}
```

### 5.4 多个task的执行顺序

```grovvy
task taskX(){
    group 'wx'
    doLast{
        println("taskX")
    }
}

task taskY(){
    group 'wx'
    doLast{
        println("taskY")
    }
}

(1)
task taskZ(dependsOn:[taskX,taskY]){
    group 'wx'
    doLast{
        println("taskZ")
    }
}

(2)
task taskZ(){
    group 'wx'
    doLast{
        println("taskZ")
    }
}
taskZ.dependsOn(taskX,taskY)   //这中taskX,taskY的执行顺序是随机的

(3)afterEvaluate使用
afterEvaluate在配置阶段执行完,所有的task都会被创建成功了.
例：统计task执行阶段的时长
def startBuildTime, endBuildTime
this.afterEvaluate {
    //保证要找的task都已经配置完毕
    Project project ->
        //找到最开始执行的task
        def preBuildTask = project.tasks.getByPath('preBuild')
        preBuildTask.doFirst {
            startBuildTime = System.currentTimeMillis()
        }
        //找到最后执行的task
        def buildTask = project.tasks.getByPath('build')
        buildTask.doLast {
            endBuildTime = System.currentTimeMillis()
            println("build的时间差:::${endBuildTime - startBuildTime}")
        }
}
```

### 5.5 Task增量构建（UP-TO-DATE）

Gradle引入了增量式构建的概念：为每个Task定义输入（inputs）和输入（outputs），比如使用java插件编译源码时，输入为java源文件，输出为class文件，如果在执行一个Task时，如果它的输入和输出与前一次执行时没有发生变化，那么Gradle便会认为该Task是最新的（UP-TO-DATE）。

## 4. Gradle的Project

### 4.1 Project简介

在Gradle中，每个待编译的工程都叫一个Project， Gradle为每个build.gradle创建了一个Project领域对象。Gradle支持多个Project，在根Project的settings.gradle配置文件进行配置，Gradle默认已经为Project定义了很多的Property，例如：

```java
project：Project本身
name：Project的名字
path：Project的绝对路径
description：Project的描述信息
buildDir：Project构建结果存放目录
version：Project的版本号
```

### 4.2 自定义Property

```groovy
ext.prperty1 = 'zhangsan'
ext {
   property2 = 'zhangsan
    
}
```

## 5. Gradle的依赖

我们在项目中总会存在一个项目依赖另一个项目或者第三方库等，怎么获取这些依赖。

- 需要配置Gradle的repositories，就会去这里下载
- 依赖的形式在项目的classpatch，即dependencies下的classpatch

### 5.1 依赖组概念（Configuration）

Gradle会对依赖进行分组，比如编译时使用这组的依赖，测试时使用另一组的依赖。每一组的依赖称为一个Configuration    

- 常用的Configuration：implementation testCompile （java plugin）

### 5.2 自定义Configuration

```groovy
configurations {
	myDependency
}
//通过dependencies()方法向myDependency中加入实际的依赖项：
dependencies {
    myDependency 'com.android.support:appcompat-v7:28.0.0'
}
task testMyDependecy() {
    group 'wx'
    println configurations.myDependency.asPath
}

```

## 6. Gradle的日志级别

Gradle和Android一样也定义了日志级别，日志级别如下；

- ERROR	错误消息
- QUIET	重要的信息消息
- WARNING	警告消息
- LIFECYCLE	进度信息消息
- INFO	信息性消息
- DEBUG	调试消息

gradle可以通过命令控制日志级别，如gradle -q + task名称。-q是- -quiet的缩写。

- 无日志选项默认：输出LIFECYCLE及更高日志选项
- -q或- -quiet：输出QUIET及更高级别日志
- -i或-- info：输出INFO及更高级别日志
- -d或- -debug：输出DEBUG及更高级别日志

## 7. Gradle常用命令

gradle tasks获取所以任务信息

gradle taska -x taskb 运行taska任务且不运行taskb任务

gradle help - -task taskName 显示任务的帮助信息

gradle taskA taskB 依次执行taskA、taskB等多个任务

gradle taskWx 可以对驼峰命名的任务进行缩写为gradle tW。但如果任务不是驼峰命名这样运行就会报错。

./gradlew app:dependencies app的库依赖结构