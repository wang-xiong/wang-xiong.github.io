---
layout: post
title: "「Android系统启动」四、SystemServer进程启动流程"
subtitle: 'Android系统启动过程学习'
date:       2018-05-04
author: "Wangxiong"
header-style: text
tags:
  - Android
  - Android系统启动
---

## 1.SystemServer进程启动过程

ZygoteInit.java的starSystemServer函数中调用handleSystemServerProcess函数启动了SystemServer进程。

```java
private static void handleSystemServerProcess(
          ZygoteConnection.Arguments parsedArgs)
          throws ZygoteInit.MethodAndArgsCaller {
    //1.SyetemServer进程是复制了Zygote进程的地址空间，因此也会得到Zygote进程创建的Socket，这个Socket对于SyetemServer进程没有用处，因此需要关闭该Socket。  
    closeServerSocket();
    ...
      if (parsedArgs.invokeWith != null) {
         ...
      } else {
          ClassLoader cl = null;
          if (systemServerClasspath != null) {
              cl = createSystemServerClassLoader(systemServerClasspath,
                                                 parsedArgs.targetSdkVersion);
              Thread.currentThread().setContextClassLoader(cl);
          }
          //2.调用RuntimeInit的zygoteInit函数
          RuntimeInit.zygoteInit(parsedArgs.targetSdkVersion, parsedArgs.remainingArgs, cl);
      }
  }
```

RuntimeInit.java代码位置：*frameworks/base/core/java/com/android/internal/os/RuntimeInit.java*。

```java
public static final void zygoteInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
            throws ZygoteInit.MethodAndArgsCaller {
        if (DEBUG) Slog.d(TAG, "RuntimeInit: Starting application from zygote");
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "RuntimeInit");
        redirectLogStreams();
        commonInit();
    //1.nativeZygoteInit函数，启动Binder线程池。
        nativeZygoteInit();
    //2.调用applicationInit函数
        applicationInit(targetSdkVersion, argv, classLoader);
    }
```

### 1.1 调用nativeZygoteInit函数

nativeZygoteInit函数对应的JNI文件，*frameworks/base/core/jni/AndroidRuntime.cpp*

```c
static const JNINativeMethod gMethods[] = {
    { "nativeFinishInit", "()V",
        (void*) com_android_internal_os_RuntimeInit_nativeFinishInit },
    { "nativeZygoteInit", "()V",
        (void*) com_android_internal_os_RuntimeInit_nativeZygoteInit },
    { "nativeSetExitWithoutCleanup", "(Z)V",
        (void*) com_android_internal_os_RuntimeInit_nativeSetExitWithoutCleanup },
};
```

通过JNI的gMethods数组，可以看出nativeZygoteInit函数对应的是JNI文件AndroidRuntime.cpp的com_android_internal_os_RuntimeInit_nativeZygoteInit函数。

```c
...
static AndroidRuntime* gCurRuntime = NULL;
...
static void com_android_internal_os_RuntimeInit_nativeZygoteInit(JNIEnv* env, jobject clazz)
{
    gCurRuntime->onZygoteInit();
}
```

gCurRuntime是AndroidRuntime的指针，AndroidRuntime的子类AppRuntime在app_main.cpp中定义，AppRuntime的onZygoteInit函数，代码位置：*frameworks/base/cmds/app_process/app_main.cpp*

```c
virtual void onZygoteInit()
   {
       sp<ProcessState> proc = ProcessState::self();
       ALOGV("App process: starting thread pool.\n");
       proc->startThreadPool();
   }
```

调用startThreadPool方法启动一个Binder线程池，SystemServer使用Binder与其他进程进行通信。

### 1.2 调用applicationInit函数

```java
private static void applicationInit(int targetSdkVersion, String[] argv, ClassLoader 
classLoader)        
    throws ZygoteInit.MethodAndArgsCaller {
    ...    
    invokeStaticMain(args.startClass, args.startArgs, classLoader);    
}
```

```java
private static void invokeStaticMain(String className, String[] argv, ClassLoader classLoader)
         throws ZygoteInit.MethodAndArgsCaller {
     Class<?> cl;
     try {
         //1.className为com.android.server.SystemServer，反射得到SystemServer类
         cl = Class.forName(className, true, classLoader);//1
     } catch (ClassNotFoundException ex) {
         throw new RuntimeException(
                 "Missing class when invoking static main " + className,
                 ex);
     }
     Method m;
     try {
         //2.得到SystemServer中的main函数。
         m = cl.getMethod("main", new Class[] { String[].class });
     } catch (NoSuchMethodException ex) {
         throw new RuntimeException(
                 "Missing static main on " + className, ex);
     } catch (SecurityException ex) {
         throw new RuntimeException(
                 "Problem getting static main on " + className, ex);
     }
     int modifiers = m.getModifiers();
     if (! (Modifier.isStatic(modifiers) && Modifier.isPublic(modifiers))) {
         throw new RuntimeException(
                 "Main method is not public and static on " + className);
     }
    //3.将找到的main函数传入到MethodAndArgsCaller异常中并抛出该异常。截获MethodAndArgsCaller异常的代码在ZygoteInit.java的main函数中
     throw new ZygoteInit.MethodAndArgsCaller(m, argv);
 }
```

代码位置：*frameworks/base/core/java/com/android/internal/os/ZygoteInit.java*

```java
public static void main(String argv[]) {
     ...
          closeServerSocket();
      } catch (MethodAndArgsCaller caller) {
    	 //1.调用了MethodAndArgsCaller的run方法。
          caller.run();
      } catch (RuntimeException ex) {
          Log.e(TAG, "Zygote died with exception", ex);
          closeServerSocket();
          throw ex;
      }
  }
```

```java
 public void run() {
        try {
            //mMethod就是SystemServer的main函数。
            mMethod.invoke(null, new Object[] { mArgs });
        } catch (IllegalAccessException ex) {
         ...
        }
    }
}
```

## 2.解析SyetemServer进程

SystemServer的main函数代码地址：*frameworks/base/services/java/com/android/server/SystemServer.java*

```java
`public static void main(String[] args) {        new SystemServer().run();    }`
```

```java
       ...
           //1.加载libandroid_servers.so。
           System.loadLibrary("android_servers");
       ...
           //2.创建SystemServiceManager
           mSystemServiceManager = new SystemServiceManager(mSystemContext);
           LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);
       ...    
        try {
           Trace.traceBegin(Trace.TRACE_TAG_SYSTEM_SERVER, "StartServices");
           //3.startBootstrapServices函数调用SystemServiceManager启动了ActivityManagerService、PowerManagerService、PackageManagerService等服务。
           startBootstrapServices();
            //4.启动了BatteryService、UsageStatsService和WebViewUpdateService。
           startCoreServices();
            //5.启动了CameraService、AlarmManagerService、VrManagerService等服务
           startOtherServices();
       } catch (Throwable ex) {
           Slog.e("System", "******************************************");
           Slog.e("System", "************ Failure starting system services", ex);
           throw ex;
       } finally {
           Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
       }
       ...
   }
```

Android官方把系统服务分为了三种类型，分别是引导服务、核心服务和其他服务，其中其他服务为一些非紧要和一些不需要立即启动的服务。系统服务大约有80多个，父类都是SystemServer。

| 引导服务                    | 作用                                                         |
| --------------------------- | ------------------------------------------------------------ |
| Installer                   | 系统安装apk时的一个服务类，启动完成Installer服务之后才能启动其他的系统服务 |
| ActivityManagerService      | 负责四大组件的启动、切换、调度。                             |
| PowerManagerService         | 计算系统中和Power相关的计算，然后决策系统应该如何反应        |
| LightsService               | 管理和显示背光LED                                            |
| DisplayManagerService       | 用来管理所有显示设备                                         |
| UserManagerService          | 多用户模式管理                                               |
| SensorService               | 为系统提供各种感应器服务                                     |
| PackageManagerService       | 用来对apk进行安装、解析、删除、卸载等等操作                  |
| **核心服务**                |                                                              |
| BatteryService              | 管理电池相关的服务                                           |
| UsageStatsService           | 收集用户使用每一个APP的频率、使用时常                        |
| WebViewUpdateService        | WebView更新服务                                              |
| **其他服务**                |                                                              |
| CameraService               | 摄像头相关服务                                               |
| AlarmManagerService         | 全局定时器管理服务                                           |
| InputManagerService         | 管理输入事件                                                 |
| WindowManagerService        | 窗口管理服务                                                 |
| VrManagerService            | VR模式管理服务                                               |
| BluetoothService            | 蓝牙管理服务                                                 |
| NotificationManagerService  | 通知管理服务                                                 |
| DeviceStorageMonitorService | 存储相关管理服务                                             |
| LocationManagerService      | 定位管理服务                                                 |
| AudioService                | 音频相关管理服务                                             |
| …                           | ….                                                           |

### 2.1 startService方法启动服务

如启动PowerManagerService会调用如下代码

```java
mPowerManagerService = mSystemServiceManager.startService(PowerManagerService.class);
```

startService函数代码位置：*frameworks/base/services/core/java/com/android/server/SystemServiceManager.java*

```java
public <T extends SystemService> T startService(Class<T> serviceClass) {
  ...
            final T service;
            try {
                Constructor<T> constructor = serviceClass.getConstructor(Context.class);
                //1.创建SystemService，这里的SystemService是PowerManagerService，
                service = constructor.newInstance(mContext);
            } catch (InstantiationException ex) {
                throw new RuntimeException("Failed to create service " + name
                        + ": service could not be instantiated", ex);
            }
...
            // Register it.
    		//2.将PowerManagerService添加到mServices中，这里mServices是一个存储SystemService类型的ArrayList.
            mServices.add(service);
            // Start it.
            try {
                //3.调用PowerManagerService的onStart函数启动PowerManagerService。
                service.onStart();
            } catch (RuntimeException ex) {
                throw new RuntimeException("Failed to start service " + name
                        + ": onStart threw an exception", ex);
            }
    		//4.返回service
            return service;
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER)
        }
    }
```

### 2.2 其他方法启动服务

除了SystemServiceManager的starService方法启动，也可以用其他方法启动，如以PackageManagerService为例。

```java
mPackageManagerService = PackageManagerService.main(mSystemContext
    ,installer, mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
```

```java
public static PackageManagerService main(Context context, Installer installer,
        boolean factoryTest, boolean onlyCore) {
    // Self-check for initial settings.
    PackageManagerServiceCompilerMapping.checkProperties();
    //1.直接创建PackageManagerService
    PackageManagerService m = new PackageManagerService(context, installer,
            factoryTest, onlyCore);
    m.enableSystemUserPackages();
    // Disable any carrier apps. We do this very early in boot to prevent the apps from being
    // disabled after already being started.
    CarrierAppUtils.disableCarrierAppsUntilPrivileged(context.getOpPackageName(), m,
            UserHandle.USER_SYSTEM);
    //2.将PackageManagerService注册到ServiceManager中
    ServiceManager.addService("package", m);
    return m;
}
```

### 2.3 还有有的服务直接注册在ServiceManager中。

ServiceManager用来管理系统中的各种Service，用于系统C/S架构中的Binder机制通信：Client端要使用某个Service，则需要到ServiceManager查下Service的相关信息，然后根据Service的相关信息与所在的Server进程建立通讯通路，这样Client端酒可以使用service了。

```java
telephonyRegistry = new TelephonyRegistry(context);
ServiceManager.addService("telephony.registry", telephonyRegistry);
```

## 3.总结SystemServer进程

SystemServer进程启动做了如下工作：

1. 启动Binder线程池，这样就可以与其他进程进行通信。
2. 创建SystemServiceManager用于对系统服务进程的创建，启动和生命周期管理。
3. 启动各种系统服务。