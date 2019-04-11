## 1. Context简介

Context是Android较为常用的类，意为上下文或者场景，是一个应用程序环境信息的接口。一般Context的使用场景大致为两类，一个是直接调用Context的方法，另一个是调用方法时传入Context。Activity、Service、Application都是间接的继承Context的，因此一个应用程序中Context的数量等于Activity和Service加Application。Context是一个抽象类，它的内部定义了很多方法和静态常量，具体的实现类为ContextImpl。和Context相关的类还有ContextWrapper、ContextThemeWrapper和Activity等等，关系如下：

ContextImpI和ContextWrapper都是继承Context，ContextWrapper内部包含了一个Context类型的mBase引用，mBase具体指向的是ContextImpl对象。对ContextWrapper方法的调用都是在内部调用ContextImpl的方法。此设计是使用了装饰模式，ContextWrapper是装饰类，对ContexImple进行了包装，ContextWrapper主要起传递作用。让ContextThemeWrapper、Service、Application都继承自ContextWrapper，继承了ContextWrapper的方法并进行了拓展，都通过mBase引用了ContextImpl。ContextThemeWrapper中包含了和主题相关的方法，所以需要主题的Activity继承了ContextThemeWrapper类。

## 2. Application Context的创建过程

在Activity的启动流程源码分析中可在，在ActivityThread类的performLaunchActivity方法中调用了**LoadedApk**的makeApplication方法创建Application。

```java
public Application makeApplication(boolean forceDefaultAppClass,
        Instrumentation instrumentation) {
   ...
   //1.创建ContextImpl对象。
   ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
   //2.创建Application对象，并传入ContextImple对象。
   app = mActivityThread.mInstrumentation.newApplication(
                cl, appClass, appContext);
   //3.将创建Application对象赋值给ContextImpl的成员变量mOunterContext. 
    appContext.setOuterContext(app);
    ...
    //4.将Application对象赋值给LoadedApk的成员变量。
    mActivityThread.mAllApplications.add(app);
    mApplication = app;
    ...    
    instrumentation.callApplicationOnCreate(app);
    ...
    return app;
}
```

```java
public Application newApplication(ClassLoader cl, String className, Context context)
        throws InstantiationException, IllegalAccessException, 
        ClassNotFoundException {
     //1.通过反射创建Application对象
    Application app = getFactory(context.getPackageName())
            .instantiateApplication(cl, className);
    //2.调用attach方法传入ContextImpl对象。
    app.attach(context);
    return app;
}
```

```java
protected void attachBaseContext(Context base) {
    if (mBase != null) {
        throw new IllegalStateException("Base context already set");
    }
    mBase = base;
}
```

attachBaseContext方法在父类ContextWrapper中，即负责给ContextWrapper的mBase成员变量，证实了mBase指向的的对象就是ContextImpl对象。

## 3. Application Context的获取过程

通过调用getApplicationContext方法来获取Application的Context，getApplicationContext方法是在父类ContextWrapper中实现。

```java
@Override
public Context getApplicationContext() {
    return mBase.getApplicationContext();
}
```

mBase是ContextImpl的对象。ContextImpl的getApplicationContext方法如下。

```java
@Override
public Context getApplicationContext() {
    return (mPackageInfo != null) ?
            mPackageInfo.getApplication() : mMainThread.getApplication();
}
```

mPackageInfo就是LoadedApk对象，LoadedApk此时不为null，所以调用的是LoadedApk的getApplication方法，返回的是的mApplication成员变量，LoadedApk的mApplication是在LoadedApk的makeApplication方法中赋值的。

## 4. Activity的Context创建过程

在ActivityThread的performLaunchActivity方法中

```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    //1.创建Activity的ContextImpl对象
    ContextImpl appContext = createBaseContextForActivity(r);
    ...
    //2.创建Activity的实例
    activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
    //3.将Activity的实例赋值给ContextImpl的成员变量mOuterContext，使得ContextImple可以访问Activity的方法。
    appContext.setOuterContext(activity);
    //4.调用Activity的attach方法，将ContextImpl对象appContext传入到Activity。
    activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback);
    //5.调用Instrumentation的callActivityOnCreate方法，从而调用Activity的onCreate方法。
    if (r.isPersistable()) {          
        mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);            
    } else {       
        mInstrumentation.callActivityOnCreate(activity, r.state);            
    }
}
```

```java
final void attach(Context context, ActivityThread aThread,
        Instrumentation instr, IBinder token, int ident,
        Application application, Intent intent, ActivityInfo info,
        CharSequence title, Activity parent, String id,
        NonConfigurationInstances lastNonConfigurationInstances,
        Configuration config, String referrer, IVoiceInteractor voiceInteractor,
        Window window, ActivityConfigCallback activityConfigCallback) {
    //1.将传入的ContextImpl对象赋值
    attachBaseContext(context);

    mFragments.attachHost(null /*parent*/);
    //2.创建PhoneWindow对象
    mWindow = new PhoneWindow(this, window, activityConfigCallback);
    mWindow.setWindowControllerCallback(this);
    //3.设置PhoneWindow的CallBack，Activity实现了
    mWindow.setCallback(this);
    mWindow.setOnWindowDismissedCallback(this);
    mWindow.getLayoutInflater().setPrivateFactory(this);
	...
    //4.给PhoneWindow设置WindowManager对象
    mWindow.setWindowManager(
            (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
            mToken, mComponent.flattenToString(),
            (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
    if (mParent != null) {
        mWindow.setContainer(mParent.getWindow());
    }
    //5.将PhoneWindow的WindowManager对象负责给Activity的成员变量mWindowManager
    mWindowManager = mWindow.getWindowManager();
    mCurrentConfig = config;

    mWindow.setColorMode(info.colorMode);

    setAutofillCompatibilityEnabled(application.isAutofillCompatibilityEnabled());
    enableAutofillCompatibilityIfNeeded();
}
```

PhoneWindow代表了应用程序的窗口，在程序运行期间，PhoneWindow会触发很多事件，比如点击事件，菜单弹出，屏幕焦点变化，这些事件都是转发给PhoneWindow关联的Activity，转发操作都是通过CallBack接口回调，Activity实现了CallBack接口，通过注释4和5可知Activity可以直接通过getWindowManager的到WindowManager的原因。

其中attachBaseContext方会调用父类的ContextWrapper对象的attachBaseContext方法，将ContextImpl赋值给成员变量mBase。

## 5. Service的Context创建过程

在Service的启动流程中，调用了ActivityThread的handleCreateService方法。

```java
private void handleCreateService(CreateServiceData data) {
    ...
    //1.创建ContextImpl的对象
    ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
    //2.将Service对象赋值给ContextImpl的成员变量
    context.setOuterContext(service);
    //3.创建Application对象
    Application app = packageInfo.makeApplication(false, mInstrumentation);
    //4.调用Service的attach方法将ContextImpl对象传入，最终赋值给父类ContextWrapper中的mBase变量
    service.attach(context, this, data.info.name, data.token, app,
                    ActivityManager.getService());        
    service.onCreate();
}
```