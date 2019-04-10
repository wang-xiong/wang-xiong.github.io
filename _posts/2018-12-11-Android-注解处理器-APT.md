## 1. APT简介

注解分为运行时注解和编译时注解，编译时注解的核心是APT。APT（Annotation Processing Tool）即注解处理器，是一种注解处理工具，主要功能就是在编译时期扫描和处理注解，通过注解生成java文件。常见的注解框架有：butterKnife、Dagger2、EvenBus等。

## 2. APT的原理

Apt原理是在某些代码元素上（如类型，函数，字段）添加注解，在编译时编译器会检查AbstractProcessor子类，并调用该类型的process函数，然后将添加注解的所有元素都传递到process函数中，Java APi已经提供了扫描注解的框架，我们可以继承AbstractProcessor类来实现自己的注解逻辑。

**APT核心:**（AbstractProcess）+代码处理（javaPoet）+ 处理器注册（AutoService）

- JavaPoet是square开源的Java代码生成框架
- auto-service是Google开源的注解处理器

## 3. android-apt（过时）

android-apt是由一位开发者自己开发的apt框架，源代码托管在这里，随着Android Gradle 插件 2.2 版本的发布，Android Gradle 插件提供了名为 annotationProcessor 的功能来完全代替 android-apt ，自此android-apt 作者在官网发表声明最新的Android Gradle插件现在已经支持annotationProcessor

添加classpath：classpath 'com.neenbedankt.gradle.plugins:android-apt:1.8'

应用插件：apply plugin: 'com.neenbedankt.android-apt'

使用apt申明注解用到的库文件，例如：apt 'com.squareup.dagger:dagger-compiler:1.1.0'

## 4. annotationProcessor使用

annotationProcessor是APT工具的一种，是google开发的内置框架，不需要引入，直接在build.gradle文件中使用，如下

```groovy
dependencies {
    annotationProcessor project(':xx')
    annotationProcessor 'com.jakewharton:butterknife-compiler:8.4.0'
}
//annotationProcessor的作用：只在编译的时候执行依赖的库，最终不会打包到apk中，目的是生成想要的类文件，供我们使用。
```

**拓展：Provided**

Provided的作用也是编译时执行，最终不会打包到apk中，但是区别很大，例如：A 、B、C都是Library，A依赖了C，B也依赖了C ，App需要同时使用A和B ，那么其中A（或者B）可以修改与C的依赖关系为Provided。A这个Library实际上还是要用到C的，只不过它知道B那里也有一个C，自己再带一个就显得多余了，等app开始运行的时候，A就可以通过B得到C，也就是两人公用这个C。所以自己就在和B汇合之前，假设自己有C。如果运行的时候没有C，肯定就要崩溃了。

总结一下，Provided是间接的得到了依赖的Library，运行的时候必须要保证这个Library的存在，否则就会崩溃，起到了避免依赖重复资源的作用。

## 5. 自定义APT

### 5.1 兼容性配置

由于Android目前不是完全支持Java 8的语言特性，会导致编译出错。这里将项目的源和目标兼容性值保留为 Java 7。 打开app模块下的build.gradle，写入如下

```groovy
compileOptions {
sourceCompatibility JavaVersion.VERSION_1_7
targetCompatibility JavaVersion.VERSION_1_7
}
```

### 5.2 自定义注解

```java
1.元注解：用来定义定义注解的注解，共用四种原注解：@Retention, @Target, @Inherited, @Documented
2.@Retention：保留的范围：默认CLASS,
SOURCE：只在源码中可用
CLASS：在源码和字节码中可用
RUNTIME：在源码，字节码，运行时都可用
3.@Retention：用来修饰哪些程序元素
TYPE, METHOD, CONSTRUCTOR, FIELD, PARAMETER等，未标注则表示可修饰所有
4.@Inherited 是否可以被继承，默认为false
5.@Documented 是否会保存到 Javadoc 文档中
注解不会继承
```

### 5.3 自定义注解处理器AbstractProcessor

#### 5.3.1 添加依赖

```groovy
implementation 'com.google.auto.service:auto-service:1.0-rc2'
implementation 'com.squareup:javapoet:1.10.0'
```

- 正常在使用注解处理器需要先声明,步骤：

  a、需要在 processors 库的 main 目录下新建 resources 资源文件夹；

  b、在 resources文件夹下建立 META-INF/services 目录文件夹；

  c、在 META-INF/services 目录文件夹下创建javax.annotation.processing.Processor 文件；

  d、在 javax.annotation.processing.Processor 文件写入注解处理器的全称，包括包路径。如：com.wx.annotationprocessor.test.MyProcessor

- 使用依赖库auto-service替代注解声明

通过auto-service中的@AutoService可以自动生成AutoService注解处理器是Google开发的，用来生成 META-INF/services/javax.annotation.processing.Processor 文件，使用方式加上如下注解@AutoService(Processor.class)。

```java
@SupportedAnnotationTypes("com.example.apt_annotation.BindView")//待处理的注解全称
@SupportedSourceVersion(SourceVersion.RELEASE_7)//表示处理的JAVA版本。
@AutoService(Processor.class)
public Class MyProcessor extends AbstractProcessor {
    //核心方法
    public boolean process()
}
```

#### 5.3.2 核心方法

- process:Annotation Processor扫描出的结果会存储进roundEnv中，处理先前 round 产生的类型元素上的注解类型集，并返回这些注解是否由此 Processor 处理。如果返回 true，则这些注解已处理后续 Processor 无需再处理它们；如果返回 false，则这些注解未处理并且可能要求后续Processor 处理它们，可以在这里写扫描、评估和处理注解的代码，生成Java文件，

- init(ProcessingEnvironment env): 每一个注解处理器类都必须有一个空的构造函数。然而，这里有一个特殊的init()方法，它会被注解处理工具调用，并输入ProcessingEnviroment参数。ProcessingEnviroment提供很多有用的工具类Elements,Types和Filer。
- public boolean process(Set<? extends TypeElement> annoations, RoundEnvironment env)这相当于每个处理器的主函数main()。在这里写扫描、评估和处理注解的代码，以及生成Java文件。输入参数RoundEnviroment，可以让查询出包含特定注解的被注解元素。
- getSupportedAnnotationTypes();这里必须指定，这个注解处理器是注册给哪个注解的。注意，它的返回值是一个字符串的集合，包含本处理器想要处理的注解类型的合法全称。换句话说，在这里定义你的注解处理器注册到哪些注解上。 
- getSupportedSourceVersion();用来指定你使用的Java版本。

#### 5.3.3 通过注解生成Java文件的方式

一种是直接用StringBuilder直接拼接写代码还有一种是使用`JavaPoet`，调用对应api生成。

#### 5.3.4 核心类

ProcessingEnvironment是一个注解处理工具的集合。主要包含：Filer、Messager、Elements

- Filer：可以用来编写新文件
- Messager：可以用来打印错误信息
- Elements：可以处理Element的工具类

Element是一个接口，表示一个程序元素，它可以是包、类、方法或者一个变量。Element已知的子接口有：PackageElement，TypeElement，VariableElement

- PackageElement 表示一个包程序元素。提供对有关包及其成员的信息的访问。
- ExecutableElement 表示某个类或接口的方法、构造方法或初始化程序（静态或实例），包括注释类型元素。 
- TypeElement 表示一个类或接口程序元素。提供对有关类型及其成员的信息的访问。注意，枚举类型是一种类，而注解类型是一种接口。 
- VariableElement 表示一个字段、enum 常量、方法或构造方法参数、局部变量或异常参数

#### 5.3.5 生成的类

build过程中创建的，所以build成功后可以查看它生成的类，路径是app/build/generated/source/apt/debug/<package>/xxx.java

## 6. 调试

依赖注入的手段

构造方法注入

setter注入

接口注入

## 7. 重点拓展

1. 多个注解处理器的顺序怎么确定？

   Annotation Processor可能会被多次调用。Annotation Processor被调用一次后,后续若还有注解处理,该Annotation Processor仍然会继续被调用.

2. 自定义Annotation Processor必须带有一个无参构造函数，让javac进行实例化.

3. 如果Annotation Processor抛出一个未捕获异常,javac可能会停止其他的Annotation Processor。只有在无法抛出错误或报告的情况下，才允许抛出异常.

4. Annotation Processor运行在一个独立的jvm中,所以可以将它看成是一个java应用。

5. JavaPoet

   [JavaPoet](https://github.com/square/javapoet)是一种快速生成代码的工具。javaPoet的占位符

   $L：文本值，

   $S：字符串

   $T：对象，会自动生成导包语句

   $N：方法名

   [学习参考](https://blog.csdn.net/qq_18242391/article/details/77018155)