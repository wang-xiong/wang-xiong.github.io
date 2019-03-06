# 一、注解

元注解：元注解指注解的注解，包括@Retention @Target @Document @Inherited四种

1. @Retention：定义注解的保留策略

   ```java
   @Retention(RetentionPolicy.SOURCE) //注解仅存在源码中，在class字节码文件中不包含。
   @Retention(RetentionPolicy.CLASS) //默认的保留策略，注解会在class字节码文件中存在，但运行时无法获得
   @Retention(RetentionPolicy.RUNTIME) //注解会在class字节码文件中存在，在运行时可以通过反射获取到。
   ```

2. @Target：定义注解的作用目标

   ```java
   @Target(ElementType.TYPE)   //接口、类、枚举、注解
   @Target(ElementType.FIELD) //字段、枚举的常量
   @Target(ElementType.METHOD) //方法
   @Target(ElementType.PARAMETER) //方法参数
   @Target(ElementType.CONSTRUCTOR)  //构造函数
   @Target(ElementType.LOCAL_VARIABLE)//局部变量
   @Target(ElementType.ANNOTATION_TYPE)//注解
   @Target(ElementType.PACKAGE) ///包 
   ```

3. @Document：说明该注解将被包含在javadoc中

4. @Inherited：说明子类可以继承父类中的该注解

# 一、APT简介

## 1.注解类型

注解分为运行时注解和编译时注解，编译时注解的核心是APT。

## 2.APT是什么

APT（Annotation Processing Tool）即注解处理器，是一种注解处理工具，主要功能就是在编译时期扫描和处理注解，通过注解生成java文件。常见的注解框架有：butterKnife、Dagger2、EvenBus等。

## 3.APT的原理

Apt原理是在某些代码元素上（如类型，函数，字段）添加注解，在编译时编译器会检查AbstractProcessor子类，并调用该类型的process函数，然后将添加注解的所有元素都传递到process函数中，Java APi已经提供了扫描注解的框架，我们可以继承AbstractProcessor类来实现自己的注解逻辑。

### APT核心:

（AbstractProcess）+代码处理（javaPoet）+处理器注册（AutoService）+apt

-  JavaPoet是square开源的Java代码生成框架
- auto-service是Google开源的注解处理器

## 4.android-apt（过时）

android-apt是由一位开发者自己开发的apt框架，源代码托管在这里，随着Android Gradle 插件 2.2 版本的发布，Android Gradle 插件提供了名为 annotationProcessor 的功能来完全代替 android-apt ，自此android-apt 作者在官网发表声明最新的Android Gradle插件现在已经支持annotationProcessor

添加classpath：classpath 'com.neenbedankt.gradle.plugins:android-apt:1.8'

应用插件：apply plugin: 'com.neenbedankt.android-apt'

使用apt申明注解用到的库文件，例如：apt 'com.squareup.dagger:dagger-compiler:1.1.0'

## 5.annotationProcessor

annotationProcessor是APT工具的一种，是google开发的内置框架，不需要引入，直接在build.gradle文件中使用，如下

```groovy
dependencies {
    annotationProcessor project(':xx')
    annotationProcessor 'com.jakewharton:butterknife-compiler:8.4.0'
}
```

annotationProcessor的作用

只在编译的时候执行依赖的库，最终不会打包到apk中，目的是生成想要的类文件，供我们使用。

### 拓展：Provided

Provided的作用也是编译时执行，最终不会打包到apk中，但是区别很大，例如：A 、B、C都是Library，A依赖了C，B也依赖了C ，App需要同时使用A和B ，那么其中A（或者B）可以修改与C的依赖关系为Provided。A这个Library实际上还是要用到C的，只不过它知道B那里也有一个C，自己再带一个就显得多余了，等app开始运行的时候，A就可以通过B得到C，也就是两人公用这个C。所以自己就在和B汇合之前，假设自己有C。如果运行的时候没有C，肯定就要崩溃了。

总结一下，Provided是间接的得到了依赖的Library，运行的时候必须要保证这个Library的存在，否则就会崩溃，起到了避免依赖重复资源的作用。

# 二、自定义APT

## 1.兼容性配置

由于Android目前不是完全支持Java 8的语言特性，会导致编译出错。这里将项目的源和目标兼容性值保留为 Java 7。 打开app模块下的build.gradle，写入如下

```groovy
compileOptions {
sourceCompatibility JavaVersion.VERSION_1_7
targetCompatibility JavaVersion.VERSION_1_7
}
```

## 2.自定义注解

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

## 3.自定义注解处理器AbstractProcessor

### （1）添加依赖：

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

### （2）核心方法

* process:Annotation Processor扫描出的结果会存储进roundEnv中，处理先前 round 产生的类型元素上的注解类型集，并返回这些注解是否由此 Processor 处理。如果返回 true，则这些注解已处理后续 Processor 无需再处理它们；如果返回 false，则这些注解未处理并且可能要求后续Processor 处理它们，可以在这里写扫描、评估和处理注解的代码，生成Java文件，

- init(ProcessingEnvironment env): 每一个注解处理器类都必须有一个空的构造函数。然而，这里有一个特殊的init()方法，它会被注解处理工具调用，并输入ProcessingEnviroment参数。ProcessingEnviroment提供很多有用的工具类Elements,Types和Filer。
- public boolean process(Set<? extends TypeElement> annoations, RoundEnvironment env)这相当于每个处理器的主函数main()。在这里写扫描、评估和处理注解的代码，以及生成Java文件。输入参数RoundEnviroment，可以让查询出包含特定注解的被注解元素。
- getSupportedAnnotationTypes();这里必须指定，这个注解处理器是注册给哪个注解的。注意，它的返回值是一个字符串的集合，包含本处理器想要处理的注解类型的合法全称。换句话说，在这里定义你的注解处理器注册到哪些注解上。 
- getSupportedSourceVersion();用来指定你使用的Java版本。

### （3）通过注解生成Java文件的方式

一种是直接用StringBuilder直接拼接写代码还有一种是使用JavaPoet，调用对应api生成。

### （4）核心类

ProcessingEnvironment是一个注解处理工具的集合。主要包含：Filer、Messager、Elements

* Filer：可以用来编写新文件
* Messager：可以用来打印错误信息
* Elements：可以处理Element的工具类

Element是一个接口，表示一个程序元素，它可以是包、类、方法或者一个变量。Element已知的子接口有：PackageElement，TypeElement，VariableElement

- PackageElement 表示一个包程序元素。提供对有关包及其成员的信息的访问。
- ExecutableElement 表示某个类或接口的方法、构造方法或初始化程序（静态或实例），包括注释类型元素。 
- TypeElement 表示一个类或接口程序元素。提供对有关类型及其成员的信息的访问。注意，枚举类型是一种类，而注解类型是一种接口。 
- VariableElement 表示一个字段、enum 常量、方法或构造方法参数、局部变量或异常参数

### （5）生成的类

​	build过程中创建的，所以build成功后可以查看它生成的类，路径是

app/build/generated/source/apt/debug/<package>/xxx.java

## 4.使用

**直接在使用的模块中添加**

annotationProcessor project(':apt:apt_processor')

## 5.调试

依赖注入的手段

构造方法注入

setter注入

接口注入

# 二、Dagger2

## 1.添加依赖

```groovy
implementation 'com.jakewharton:butterknife:8.0.0'
annotationProcessor 'com.jakewharton:butterknife-compiler:8.0.0'
```

## 2.注解说明

### 1.@Inject

@Inject注解，运用的地方有两处，

1.@Inject给一个类的属性做标记时，表明它是一个依赖需求方，需要一些依赖。

2.@Inject给一个类的构造方法进行注解时，表示它提供依赖的能力。

### 2.@Component

@Component相当于关系纽带，将@inject标记的需求和依赖绑定起来，建立联系。

### 3.@Provides和@Module

Dagger2 中规定，用 @Provides 注解的依赖必须存在一个用 @Module 注解的类中。

如果一个Component没有实现任何构造方法，那么Component中Dagger2会自动创建，如果Module实现了有参的构造方法，那么它需要在Component构建的时候手动传递进去。

Component可以通过create方法创建，但是前提是，Component中的Module中被@Provides注解的方法都必须是静态方法，也就是它们必须被static修饰。

### 4.@Inject 和 @Provides 的优先级

一个类被 @Inject 注解了构造方法，又在某个 Module 中的 @Provides 注解的方法中提供了依赖。那么会使用@Provides提供的依赖。Dagger2依赖查找的顺序是先查找Module内所有的@Provides提供的依赖，如果查找不到再去查找@Inject提供的依赖。

### 5.@Singleton

用@Singleton标记在目标单例上或者Provides提供依赖的方法上，然后用@Singleton标记Component对象上。目标类就实现了此作用域下（Compoent的范围内）的单例效果。

### 6.@Scope

@Scope是作用域的意思，@Scope是一个元注解，@Singleton只是@Scope的一个默认实现而已。

所以可以自定义一个域@Scope的注解

```java
@Target(ANNOTATION_TYPE)
@Retention(RUNTIME)
@Documented
public @interface Scope {}
```

```java
@Scope
@Documented
@Retention(RUNTIME)
public @interface Singleton {}
```

### 7.@Qualifiers 和 @Name

如果存在两个返回值一样类型的Provides，怎么确定依赖关系呢，就是使用@Name注解进行区分，配合@Inject和@Provides一起使用。

@Qualufie是一个元注解，Qualifiers是修饰符的意思，而@Name是被@Qualufies注解的一个注解。所以我们也可以自己定义@Qualufies代替@Name，效果相同。

### 8.Dagger2的延迟加载

Dagger2提供了延迟加载的能力，即只有使用到需要的依赖时才会实例依赖对象。

方法是使用Lazy，Lazy是泛型类，接受任何类型的参数。

```java
@Inject
@Named("TestLazy")
Lazy<String> name;
```

### 9.Provider 强制重新加载

Provider 强制重新加载

```
@Inject
Provider<Integer> randomValue;
```

应用 @Singleton 的时候，我们希望每次都是获取同一个对象，但有的时候，我们希望每次都创建一个新的实例，这种情况显然与 @Singleton 完全相反。Dagger2 通过 Provider 就可以实现。它的使用方法和 Lazy 很类似，但是，需要注意的是 Provider 所表达的重新加载是说每次重新执行 Module 相应的 @Provides 方法，如果这个方法本身每次返回同一个对象，那么每次调用 get() 的时候，对象也会是同一个。

10.Dagger2 中 Component 之间的依赖

通过dependencies添加依赖的Component

```groovy
@Component(modules = CommonModule.class, dependencies = SecondComponent.class)
```

11.Dagger2 中的 SubComponent

在 Java 软件开发中，我们经常面临的就是“组合”和“继承”的概念。它们都是为了扩展某个类的功能。 前面的 Component 的依赖采用 @Component(dependecies=othercomponent.class) 就相当于组合。 

使用 Subcomponent 时，还是要先构造 ParentComponent 对象，然后通过它提供的 SubComponent 再去进行依赖注入。

可以细细观察下它与 depedency 方法的不同之处。但是，SubComponent 同时具备了 ParentComponent 和自身的 @Scope 作用域。所以，这经常会造成混乱的地方。大家需要注意。如果你要我比较，SubComponent 和 dependency 形式哪种更好时，我承认各有优点，但我自己倾向于 dependency，因为它更灵活。不是说 组合优于继承嘛。

# JavaPoet

[JavaPoet](https://github.com/square/javapoet)是一种快速生成代码的工具。

javaPoet的占位符

$L：文本值，

$S：字符串

$T：对象，会自动生成导包语句

$N：方法名

参考：https://blog.csdn.net/qq_18242391/article/details/77018155

# issues

1.多个注解处理器的顺序怎么确定？

Annotation Processor可能会被多次调用.

Annotation Processor被调用一次后,后续若还有注解处理,该Annotation Processor仍然会继续被调用.

自定义Annotation Processor必须带有一个无参构造函数,让javac进行实例化.

如果Annotation Processor抛出一个未捕获异常,javac可能会停止其他的Annotation Processor.只有在无法抛出错误或报告的情况下,才允许抛出异常.

Annotation Processor运行在一个独立的jvm中,所以可以将它看成是一个java应用程序