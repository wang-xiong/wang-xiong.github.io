Flutter

**pubspec.yaml**  文件管理 Flutter 应用程序的 assets（资源，如图片、package。

State*less* widgets 是不可变的，这意味着它们的属性不能改变——所有的值都是 final。

State*ful* widgets 持有的状态可能在 widget 生命周期中发生变化，实现一个 stateful widget 至少需要两个类：1）一个 [StatefulWidget](https://docs.flutter.io/flutter/widgets/StatefulWidget-class.html) 类；2）一个 [State](https://docs.flutter.io/flutter/widgets/State-class.html) 类，StatefulWidget 类本身是不变的，但是 State 类在 widget 生命周期中始终存在。

Flutter Application

Flutter Plugin

* Android：插件本地代码的Android端实现
* ios：IOS端的实现
* lib：Dart代码。插件的客户将会使用这里实现的接口
* example：插件的使用示例

Flutter Packagge

Flutter Module



Android引入Flutter模块https://www.cnblogs.com/fuyaozhishang/p/9617234.html

* 1.创建一个Android工程FlutterInAndroid

* 2.同一级目录创建一个FlutterModule，命令：flutter create -t module my_flutter

* Android工程的setting.gradle引入flutter模块

  ```groo
  setBinding(new Binding([gradle: this]))
  evaluate(new File(
          settingsDir.parentFile,
          'my_flutter/.android/include_flutter.groovy'
  ))
  ```

* app的build.gradle添加依赖

  ```groovy
  implementation project(':flutter')
  ```