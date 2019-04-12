## 1. Gradle插件简介

插件的好处就是可以动态的插拔 ，来实现或卸载某些功能。自定义Gradle插件可以让我们更好的管理构建Android apk，从而在编译构建的过程中添加某些功能。应用Gradle插件需要调用Project的apply方法。Gradle插件一般分为两种，一种是脚本插件，脚本插件是额外的构建插件可以理解为普通的xxx.gradle。另一种是对象插件又叫二进制插件，是通过实现了Plugin接口的类。[学习Demo](https://github.com/wang-xiong/WxApp/tree/master/gradle_plugins)

### 1.1 脚本插件

脚本插件直接定义在项目中，如my.gradle。应用时直接在项目其他插件中通过领域对象Project的apply方法引入：

```groovy
apply from ：'my.gradle'
```

### 1.2 对象插件

对象插件实现了`org.gradle.api.plugins<Project>`接口，对象插件可以分为内部插件和第三方插件。

#### 1.2.1内部插件

Gradle 的发行包中有大量的插件，这些插件有很多类型，比如语言插件、集成插件、软件开发插件等等，例如如果我们想在项目中应用Java插件，如下 

```groovy
`apply plugin: org.gradle.api.plugins.JavaPlugin `
```

#### 1.2.2 第三方插件

第三方插件对象通常是jar文件，引入第三方插件，需要现在根目录的build.gradle文件的buildscript下指定插件下载仓库地址：

```groovy
buildscript { //定义全局的相关属性
    repositories {
        maven {
            url uri('gradle_plugins/repo') //本地仓库
        }
    }
     
    dependencies {
        //自定义插件
        classpath 'com.wx.plugin.example.plugin:plugin_test:1.1.0'
    }
    
}

apply plugin: 'test-plugin' //应用插件
```

## 2. 插件DSL

Gradle的特性有四种状态，分别是Internal、Incubating、Public、Deprecated，插件DSL属于Incubating状态（孵化状态）。这也导致插件DSL的特性在将来的Gradle版本中可能会发生变化，直到它不再孵化为止。使用Java插件可以这么写：

```groovy
//1.使用内部插件
plugins {
	id 'java'
}
//2.使用第三方插件
plugins {
    id "test-plugin" version "1.1.0"
}
```

## 3. 自定义插件

自定义gradle第三方插件有多种方式，可以直接在IDE上新建Groovy工程，前提是IDE需要安装对应插件。

### 3.1 直接在build.gradle中定义

```groovy
//Project中应用插件
apply plugin: MyPlugin
//Project中指定插件定义Extension的属性
dateAndTime {   		
    timeFormat = 'HH:mm:ss.SSS' 
    dateFormat = 'MM/dd/yyyy'
}

class MyPlugin implements Plugin<Project> {
    @Override
    void apply(Project target) {
        //1.为Project领域对象自定义extension为dateAndTime 
        target.extensions.create("dateAndTime", DateAddTimePluginExtension)
        //2.为Project定义task 
        target.task('showTime') << {
            println("current time is:" + new 
                    Date().format(target.dateAndTime.timeFormat)) 
        }
        //3.设置task的group
        target.showTime.group = 'wx'

        target.tasks.create('showData') << {
            println("current date is:" + new 
                    Date().format(target.dateAndTime.dateFormat))
        }
        target.showData.group = 'wx'
    }
}

//为Project定义拓展属性
class DateAddTimePluginExtension {
    String timeFormat = "MM/dd/yyyyHH:mm:ss.SSS"
    String dateFormat = "yyyy-MM-dd"
}
```



### 3.2 新建单独的插件工程

- 新建工程（plugin_test）
- 工程目录新建src/main目录和build.gradle文件
- main目录下新建groovy目录和resources目录
- groovy目录下新建包名目录（com.wx.plugin.test）
- resources下新建META-INF/gradle-plugins目录
- groovy/包名目录新建.groovy文件，如MyPlugin.groovy
- resources/META-INF/gradle-plugins目录新建.properties文件，文件名就是最终的插件名，如：test-plugin.properties

### 3.2 直接新建buildSrc

直接在项目根目录新增buildSrc目录，将插件的源码放在rootProjectDir/buildSrc/src/main/groovy目录下，Gradle回自动识别来完成编译。

## 4. Transform

[Gradle Transform](http://tools.android.com/tech-docs/new-build-system/transform-api)是Android官方提供给开发者在项目构建阶段即由class到dex转换期间修改class文件的一套api。目前比较经典的应用是字节码插桩、代码注入技术。同时结合ASM框架生成相对应的代码，ASM框架主要是写入或者插入class的字节码。

1. TransformInput：Transform就是对输入的class文件转变成目标字节码文件，TransformInput就是这些输入文件的抽象。目前它包括两部分：DirectoryInput集合与JarInput集合。

   - DirectoryInput：它代表着以源码方式参与项目编译的所有目录结构及其目录下的源码文件，可以借助于它来修改输出文件的目录结构、以及目标字节码文件。
   - JarInput：它代表着以jar包方式参与项目编译的所有本地jar包或远程jar包，可以借助于它来动态添加jar包。

2. TransformOutputProvider：它代表的是Transform的输出。

3. 自定义Transform主要实现如下方法： 

   ```groovy
   class MyTransform extends Transform {
       @Override
       void transform(Context context
                      , Collection<TransformInput> inputs
                      , Collection<TransformInput> referencedInputs
                      , TransformOutputProvider outputProvider
                      , boolean isIncremental) throws IOException, TransformException
       , InterruptedException {
           
       }
   
       @Override
       String getName() {
           return "MyTransform"
       }
       
       @Override
       Set<QualifiedContent.ContentType> getInputTypes() {
           return TransformManager.CONTENT_CLASS
       }
   
       @Override
       Set<? super QualifiedContent.Scope> getScopes() { 
           return TransformManager.SCOPE_FULL_PROJECT
       }
       @Override
       boolean isIncremental() {
           return false
       }
   }
   ```

- getName:代表Transform的Task名称
- getInputTypes：用于指明Transform的输入类型，可以作为输入过滤的手段

```groovy
public static final Set<ContentType> CONTENT_CLASS = ImmutableSet.of(CLASSES);
public static final Set<ContentType> CONTENT_JARS = ImmutableSet.of(CLASSES, RESOURCES);
public static final Set<ContentType> CONTENT_RESOURCES = ImmutableSet.of(RESOURCES);
public static final Set<ContentType> CONTENT_NATIVE_LIBS =
        ImmutableSet.of(NATIVE_LIBS);
public static final Set<ContentType> CONTENT_DEX = ImmutableSet.of(ExtendedContentType.DEX);
public static final Set<ContentType> DATA_BINDING_ARTIFACT =
        ImmutableSet.of(ExtendedContentType.DATA_BINDING);
public static final Set<ContentType> DATA_BINDING_BASE_CLASS_LOG_ARTIFACT =
        ImmutableSet.of(ExtendedContentType.DATA_BINDING_BASE_CLASS_LOG);
```

- getScopes：用于指明Transform的作用域，在TransformManager定义了如下的三种种类型：

  ```groovy
  public static final Set<Scope> SCOPE_FULL_PROJECT =
          Sets.immutableEnumSet(
                  Scope.PROJECT,
                  Scope.SUB_PROJECTS,
                  Scope.EXTERNAL_LIBRARIES);
  public static final Set<ScopeType> SCOPE_FULL_WITH_IR_FOR_DEXING =
          new ImmutableSet.Builder<ScopeType>()
                  .addAll(SCOPE_FULL_PROJECT)
                  .add(InternalScope.MAIN_SPLIT)
                  .build();
  public static final Set<ScopeType> SCOPE_FULL_LIBRARY_WITH_LOCAL_JARS =
          ImmutableSet.of(Scope.PROJECT, InternalScope.LOCAL_DEPS);
  ```

- isIncremental：用于指明是否是增量构建。