---
layout: post
title: "Android框架之Hybrid开发"
subtitle: 'Android框架学习，Hybrid混合开发学习
date:       2019-01-02
author: "Wangxiong"
header-style: text
tags:
  - Android
  - Hybrid
  - 应用程序框架
---
Hybrid Android混合开发，java+h5

参考来源：https://www.cnblogs.com/dailc/p/5931324.html

issues：

1、WebView如何加载H5的页面

2、Android如何调用H5的方法

3、H5如何调用Android的方法

1.Webview加载H5

* 加载在线url

  ```java
  webView.loadUrl("http://www.baidu.com");
  ```

* 加载离线h5，如加载assets文件夹下的test.html

  ```java
  webView.loadUrl("file:///android_asset/webview_test.html");
  ```

2.Android调用js的方法

```java
//1、调用无参数无返回值的方法
private void invokeJsMethod1() {
    webView.loadUrl("JavaScript:show()");
}

//2、调用带参数的方法
private void invokeJsMethod2() {
    //固定字符串之间用单引号
    //webView.loadUrl("javascript:alertMessage('hello word')");
    //变量名用转译分隔符隔开
    String content = "Android调用了js方法";
    webView.loadUrl("javascript:alertMessage( \" " + content + " \" )");
}

//3.evaluateJavascript调用
private void invokeJsMethod4() {
    String content = "Android调用了js方法";
    webView.evaluateJavascript("javascript:alertMessage(\"" + content + " \" )", new ValueCallback<String>() {
        @Override
        public void onReceiveValue(String value) {
            Toast.makeText(WebActivity.this, "调用js结果value:" + value, Toast.LENGTH_SHORT).show();
        }
    });
}

//4、调用有参数有返回值的方法
private void invokeJsMethod3() {
    webView.evaluateJavascript("sum(1,2)", new ValueCallback<String>() {
        @Override
        public void onReceiveValue(String value) {
            //value是js的返回结果
            Toast.makeText(WebActivity.this, "调用js结果value:" + value, Toast.LENGTH_SHORT).show();
        }
    });
}
```

总结：

- **4.4之前Native通过loadUrl来调用JS方法,只能让某个JS方法执行,但是无法获取该方法的返回值，方便简洁 效率低、返回值麻烦 使用场景：不需要返回值对性能要求低时**
- **4.4之后,通过evaluateJavascript异步调用JS方法,并且能在onReceiveValue中拿到返回值**
- **不适合传输大量数据(大量数据建议用接口方式获取)**
- **mWebView.loadUrl("javascript: 方法名('参数,需要转为字符串')");函数需在UI线程运行，因为mWebView为UI控件(但是有一个坏处是会阻塞UI线程)**

```java
private void invokeJsMethod() {
    String content = "Android调用了js方法";
    if (Build.VERSION.SDK_INT < 18) {
        webView.loadUrl("javascript:alertMessage( \" " + content + " \" )");
    } else {
        webView.evaluateJavascript("javascript:alertMessage(\"" + content + " \" )", new ValueCallback<String>() {
            @Override
            public void onReceiveValue(String value) {
                Toast.makeText(WebActivity.this, "调用js结果value:" + value, Toast.LENGTH_SHORT).show();
            }
        });
    }
}
```

3.JS调用Native
对于JS调用Android代码的方法有3种：

1.通过WebView的addJavascriptInterface（）进行对象映射

2.通过 WebViewClient 的shouldOverrideUrlLoading ()方法回调拦截 url

3.通过 WebChromeClient 的onJsAlert()、onJsConfirm()、onJsPrompt（）方法回调拦截JS对话框alert()、confirm()、prompt（） 消息

首先必须要设置Android容器允许JS脚本调用

```java
WebSettings webSettings = mWebView.getSettings();
webSettings.setJavaScriptEnabled(true);
```

方法一

```java
//Android容器设置侨连对象
private void invokeAndroidMethod() {
    //1、定义类，方法
    //2、打开js接口，参数1类名， 参数2，别名
    //3、h5调用方法为window.类名.别名.方法名
    //问题存在内存泄露
    webView.addJavascriptInterface(new AndroidToJs(), "android");

}

public class AndroidToJs {
    //1.定义JS需要调用的方法
    @JavascriptInterface
    public String hello(String msg) {
        return "JS调用了Android的方法";
    }
}
```

原生和H5的另一种通信方式：JSBridge

什么是JSBridge

JSBridge是广为流行的Hybrid开发中JS和Native一种通信方式,各大公司的应用中都有用到这种方法

简单的说,JSBridge就是定义Native和JS的通信,Native只通过一个固定的桥对象调用JS,JS也只通过固定的桥对象调用Native,基本原理是:

H5->通过某种方式触发一个url->Native捕获到url,进行分析->原生做处理->Native调用H5的JSBridge对象传递回调。如下图

![image-20190115144051749](/Users/wangxiong/Library/Application Support/typora-user-images/image-20190115144051749.png)

使用JSBridge的原因

- **Android4.2以下,addJavascriptInterface方式有安全漏掉**
- **iOS7以下,JS无法调用Native**
- **url scheme交互方式是一套现有的成熟方案,可以完美兼容各种版本,不存在上述问题**

