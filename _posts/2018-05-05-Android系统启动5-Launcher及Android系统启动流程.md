---
layout: post
title: "「Android系统启动」五、Launcher及Android系统启动流程"
subtitle: 'Android系统启动过程学习'
date:       2018-05-05
author: "Wangxiong"
header-style: text
tags:
  - Android
  - Android系统启动
---
## 1. Launcher简介

Android系统启动的最后一步就是启动一个系统已经安装的Launcher应用程序。Launcher启动过程中会会请求PackageManagerService返回系统中已经安装的应用程序信息，将这些信息封装成一个快捷图标显示在屏幕。

## 2. Launcher启动流程

SystemServer进程启动的过程会启动PackageManagerService，PackageManagerService启动后会将系统中的应用程序安装完成。同事SystemServer启动的时候会启动ActivityManagerService，ActivityManagerService会将Launcher启动起来，具体代码在SystemServer.java中调用了mActivityManagerService.systemReady。

```java
private void startOtherServices() {
    ...
    mActivityManagerService.systemReady(new Runnable() {    
        @Override    
        public void run() {   
            Slog.i(TAG, "Making services ready");    
            mSystemServiceManager.startBootPhase(        
                SystemService.PHASE_ACTIVITY_MANAGER_READY);
            ...
        }
        ...
    }                                       
}
```

startOtherServices函数中，会调用ActivityManagerService的systemReady函数：
*frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java*

```java
public void systemReady(final Runnable goingCallback) {
...
synchronized (this) {
           ...
            mStackSupervisor.resumeFocusedStackTopActivityLocked();
            mUserController.sendUserSwitchBroadcastsLocked(-1, currentUserId);
        }
    }
```

systemReady函数中调用了ActivityStackSupervisor的resumeFocusedStackTopActivityLocked函数：*frameworks/base/services/core/java/com/android/server/am/ActivityStackSupervisor.java*

```java
boolean resumeFocusedStackTopActivityLocked(ActivityStack targetStack, ActivityRecord target, ActivityOptions targetOptions) {
        if (targetStack != null && isFocusedStack(targetStack)) {
            return targetStack.resumeTopActivityUncheckedLocked(target, targetOptions);//1
        }
        final ActivityRecord r = mFocusedStack.topRunningActivityLocked();
        if (r == null || r.state != RESUMED) {
            mFocusedStack.resumeTopActivityUncheckedLocked(null, null);
        }
        return false;
    }
```

调用ActivityStack的resumeTopActivityUncheckedLocked函数，ActivityStack对象是用来描述Activity堆栈的，resumeTopActivityUncheckedLocked函数如下所示。
*frameworks/base/services/core/java/com/android/server/am/ActivityStack.java*

```java
boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
       if (mStackSupervisor.inResumeTopActivity) {
           // Don't even start recursing.
           return false;
       }
       boolean result = false;
       try {
           // Protect against recursion.
           mStackSupervisor.inResumeTopActivity = true;
           if (mService.mLockScreenShown == ActivityManagerService.LOCK_SCREEN_LEAVING) {
               mService.mLockScreenShown = ActivityManagerService.LOCK_SCREEN_HIDDEN;
               mService.updateSleepIfNeededLocked();
           }
           result = resumeTopActivityInnerLocked(prev, options);
       } finally {
           mStackSupervisor.inResumeTopActivity = false;
       }
      return result;
   }
```

```java
private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
   ...
   return isOnHomeDisplay() &&
          mStackSupervisor.resumeHomeStackTask(returnTaskType, prev, "prevFinished");
   ...                 
}
```

```java
boolean resumeHomeStackTask(int homeStackTaskType, ActivityRecord prev, String reason) {
    ...
    if (r != null && !r.finishing) {
        mService.setFocusedActivityLocked(r, myReason);
        return resumeFocusedStackTopActivityLocked(mHomeStack, prev, null);
    }
    return mService.startHomeActivityLocked(mCurrentUser, myReason);//1
}
```

```java
boolean startHomeActivityLocked(int userId, String reason) {
     if (mFactoryTest == FactoryTest.FACTORY_TEST_LOW_LEVEL
             && mTopAction == null) {//1
         return false;
     }
     Intent intent = getHomeIntent();//2
     ActivityInfo aInfo = resolveActivityInfo(intent, STOCK_PM_FLAGS, userId);
     if (aInfo != null) {
         intent.setComponent(new ComponentName(aInfo.applicationInfo.packageName, aInfo.name));
         aInfo = new ActivityInfo(aInfo);
         aInfo.applicationInfo = getAppInfoForUser(aInfo.applicationInfo, userId);
         ProcessRecord app = getProcessRecordLocked(aInfo.processName,
                 aInfo.applicationInfo.uid, true);
         if (app == null || app.instrumentationClass == null) {//3
             intent.setFlags(intent.getFlags() | Intent.FLAG_ACTIVITY_NEW_TASK);
             mActivityStarter.startHomeActivityLocked(intent, aInfo, reason);//4
         }
     } else {
         Slog.wtf(TAG, "No home screen found for " + intent, new Throwable());
     }

     return true;
 }
```

注释1处的mFactoryTest代表系统的运行模式，系统的运行模式分为三种，分别是非工厂模式、低级工厂模式和高级工厂模式，mTopAction则用来描述第一个被启动Activity组件的Action，它的值为Intent.ACTION_MAIN。因此注释1的代码意思就是mFactoryTest为FactoryTest.FACTORY_TEST_LOW_LEVEL（低级工厂模式）并且mTopAction=null时，直接返回false。注释2处的getHomeIntent函数如下所示。

```java
Intent getHomeIntent() {
    Intent intent = new Intent(mTopAction, mTopData != null ? Uri.parse(mTopData) : null);
    intent.setComponent(mTopComponent);
    intent.addFlags(Intent.FLAG_DEBUG_TRIAGED_MISSING);
    if (mFactoryTest != FactoryTest.FACTORY_TEST_LOW_LEVEL) {
        intent.addCategory(Intent.CATEGORY_HOME);
    }
    return intent;
}
```

getHomeIntent函数中创建了Intent，并将mTopAction和mTopData传入。mTopAction的值为Intent.ACTION_MAIN，并且如果系统运行模式不是低级工厂模式则将intent的Category设置为Intent.CATEGORY_HOME。我们再回到ActivityManagerService的startHomeActivityLocked函数，假设系统的运行模式不是低级工厂模式，在注释3处判断符合Action为Intent.ACTION_MAIN，Category为Intent.CATEGORY_HOME的应用程序是否已经启动，如果没启动则调用注释4的方法启动该应用程序。

## 3. Android系统启动流程

1. 启动电源以及系统启动

   当按下电源时引导芯片代码从预定义的地方开始执行，加载引导程序Bootloader到RAM，然后执行。

2. 引导程序Bootloader

   引导程序Bootloader是在Android操作系统运行前的一个小程序，主要作用就是把系统OS拉起来。

3. Linux内核启动

   内核启动时，设置缓存，被保护存储器，计划列表，加载驱动，当内核完成系统设置，它首先绘制系统文件中寻找init.rc文件，并启动init进程。

4. init进程启动

   初始化和启动属性服务，并启动Zygote进程。

5. Zygote进程启动

   创建JavaVM并为JavaVM注册JNI，创建服务Socket，启动SystemServer进程。

6. SystemServer进程启动

   启动Binder线程池，创建SystemServiceManager，并启动各种系统服务。

7. Launcher启动

   被SystemServer进程启动的ActivityManagerService会启动Launcher，Launcher启动后会将已经安装应用的快捷图片显示在界面上。