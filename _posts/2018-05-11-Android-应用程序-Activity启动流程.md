---
layout: post
title: "Android应用程序之Activity启动流程"
subtitle: 'Android应用程序学习'
date:       2018-05-11
author: "Wangxiong"
header-style: text
tags:
  - Android
  - Android应用程序
---
启动一个应用程序的表现是，点击Launcher的上的应用图标，然后系统启动应用的第一个Activity，可以称为根Activity。

## 1. 根Activity的启动流程

核心：Launcher请求AMS流程、AMS创建应用进程、AMS与应用进程建立通信，应用进程启动创建启动应用流程。

### 1.1 Launcher请求AMS，AMS到ApplicationThread的调用过程

点击Launcher中的代码最终会调用到startActivityForResult方法启动Activity。

```java
public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
        @Nullable Bundle options) {
    if (mParent == null) {
        options = transferSpringboardActivityOptions(options);
        Instrumentation.ActivityResult ar =
            mInstrumentation.execStartActivity(
                this, mMainThread.getApplicationThread(), mToken, this,
                intent, requestCode, options);
        if (ar != null) {
            mMainThread.sendActivityResult(
                mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                ar.getResultData());
        }
        if (requestCode >= 0) {
            // If this start is requesting a result, we can avoid making
            // the activity visible until the result is received.  Setting
            // this code during onCreate(Bundle savedInstanceState) or onResume() will keep the
            // activity hidden during this time, to avoid flickering.
            // This can only be done when a result is requested because
            // that guarantees we will get information back when the
            // activity is finished, no matter what happens to it.
            mStartedActivity = true;
        }

        cancelInputsAndStartExitTransition(options);
        // TODO Consider clearing/flushing other event sources and events for child windows.
    } else {
        if (options != null) {
            mParent.startActivityFromChild(this, intent, requestCode, options);
        } else {
            // Note we want to go through this method for compatibility with
            // existing applications that may have overridden it.
            mParent.startActivityFromChild(this, intent, requestCode);
        }
    }
}
```

在startActivityForResult方法中，mParent是Activity的类型，表示当前的Activity的父类，此时根Activity未创建所以mParent是null，接着调用Instrumentation的execStartActivity方法。Instrumentation主要用来监控应用程序和系统的交互。

```java
public ActivityResult execStartActivity(
    Context who, IBinder contextThread, IBinder token, String target,
    Intent intent, int requestCode, Bundle options) {
    ...
    try {
        intent.migrateExtraStreamToClipData();
        intent.prepareToLeaveProcess(who);
        int result = ActivityManager.getService()
            .startActivity(whoThread, who.getBasePackageName(), intent,
                    intent.resolveTypeIfNeeded(who.getContentResolver()),
                    token, target, requestCode, 0, null, options);
        checkStartActivityResult(result, intent);
    } catch (RemoteException e) {
        throw new RuntimeException("Failure from system", e);
    }
    return null;
}
```

在Instrumentation的execStartActivity方法中最终调用了ActivityManager.getService()的startActivity方法，ActivityManager.getService()获取的是ActivityManagerService的代理对象。

```java
@Override
public final int startActivity(IApplicationThread caller, String callingPackage,
        Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
    return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
            resultWho, requestCode, startFlags, profilerInfo, bOptions,
            UserHandle.getCallingUserId());
}
```

ActivityManagerService的startActivity方法又调用了startActivityAsUser方法。

```java
public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
        Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId,
        boolean validateIncomingUser) {
    //判断调用者进程是否被隔离
    enforceNotIsolatedCaller("startActivity");

    //判断调用者是否有权限启动
    userId = mActivityStartController.checkTargetUser(userId, validateIncomingUser,
            Binder.getCallingPid(), Binder.getCallingUid(), "startActivityAsUser");

    // TODO: Switch to user app stacks here.
    return mActivityStartController.obtainStarter(intent, "startActivityAsUser")
            .setCaller(caller)
            .setCallingPackage(callingPackage)
            .setResolvedType(resolvedType)
            .setResultTo(resultTo)
            .setResultWho(resultWho)
            .setRequestCode(requestCode)
            .setStartFlags(startFlags)
            .setProfilerInfo(profilerInfo)
            .setActivityOptions(bOptions)
            .setMayWait(userId)
            .execute();

}
```

调用mActivityStartController的excute执行，其实就是调用了ActivityStarter的进行启动。

```java
int execute() {
    try {
        // TODO(b/64750076): Look into passing request directly to these methods to allow
        // for transactional diffs and preprocessing.
        if (mRequest.mayWait) {
            return startActivityMayWait(mRequest.caller, mRequest.callingUid,
                    mRequest.callingPackage, mRequest.intent, mRequest.resolvedType,
                    mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                    mRequest.resultWho, mRequest.requestCode, mRequest.startFlags,
                    mRequest.profilerInfo, mRequest.waitResult, mRequest.globalConfig,
                    mRequest.activityOptions, mRequest.ignoreTargetSecurity, mRequest.userId,
                    mRequest.inTask, mRequest.reason,
                    mRequest.allowPendingRemoteAnimationRegistryLookup);
        } else {
            return startActivity(mRequest.caller, mRequest.intent, mRequest.ephemeralIntent,
                    mRequest.resolvedType, mRequest.activityInfo, mRequest.resolveInfo,
                    mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                    mRequest.resultWho, mRequest.requestCode, mRequest.callingPid,
                    mRequest.callingUid, mRequest.callingPackage, mRequest.realCallingPid,
                    mRequest.realCallingUid, mRequest.startFlags, mRequest.activityOptions,
                    mRequest.ignoreTargetSecurity, mRequest.componentSpecified,
                    mRequest.outActivity, mRequest.inTask, mRequest.reason,
                    mRequest.allowPendingRemoteAnimationRegistryLookup);
        }
    } finally {
        onExecutionComplete();
    }
}
```

在startActivityAsUser方法中设置了mRequest.mayWait为true，所以这里执行的是ActivityStarter的startActivityMayWait方法。ActivityStarter主要是加载Activity的控制类，负责收集所有的逻辑来决定如何将Intent和Flags转换为Activity，并将Activity和Task以及Stack相关联，接着调用了ActivityStarter的startActivityUnchecked方法，最后调用了ActivityStackSupervisor的resumeFocusedStackTopActivityLocked方法。

```java
boolean resumeFocusedStackTopActivityLocked(
        ActivityStack targetStack, ActivityRecord target, ActivityOptions targetOptions) {

    if (!readyToResume()) {
        return false;
    }

    if (targetStack != null && isFocusedStack(targetStack)) {
        return targetStack.resumeTopActivityUncheckedLocked(target, targetOptions);
    }

    final ActivityRecord r = mFocusedStack.topRunningActivityLocked();
    if (r == null || !r.isState(RESUMED)) {
        mFocusedStack.resumeTopActivityUncheckedLocked(null, null);
    } else if (r.isState(RESUMED)) {
        // Kick off any lingering app transitions form the MoveTaskToFront operation.
        mFocusedStack.executeAppTransition(targetOptions);
    }

    return false;
}
```

接着调用了ActivityStack的resumeTopActivityUncheckedLocked方法，最后又调回到ActivityStackSupervisor的startSpecificActivityLocked方法，接着调用了realStartActivityLocked的方法。

```java
realStartActivityLocked {
    ...
    final ClientTransaction clientTransaction = ClientTransaction.obtain(app.thread,
                        r.appToken);
                clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
                        System.identityHashCode(r), r.info,
                        // TODO: Have this take the merged configuration instead of separate global
                        // and override configs.
                        mergedConfiguration.getGlobalConfiguration(),
                        mergedConfiguration.getOverrideConfiguration(), r.compat,
                        r.launchedFromPackage, task.voiceInteractor, app.repProcState, r.icicle,
                        r.persistentState, results, newIntents, mService.isNextTransitionForward(),
                        profilerInfo));
    ...
}
```

在realStartActivityLocked方法中，通过ActivityManagerService调用了ClientTransaction的schedule方法。

```java
public void schedule() throws RemoteException {
    mClient.scheduleTransaction(this);
}
```

其中mClient是IApplicationThread，IApplicationThread的实现类是ActivityThread的内部类ApplicationThread。通过ApplicationThread实现了ActivityManagerService与应用程序的进程的Binder通信，ActivityManagerService运行在SystemServer进程，应用进程的ActivityThread的内部类实现了IApplicationThread.Stub。所以ApplicationThread是AMS所在进程和应用程序进程通信的桥梁。

![activity启动1.png](https://upload-images.jianshu.io/upload_images/10547376-2fc4e16d1ad25cc2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 1.2 ActivityThread启动Activity的过程

ActivityThread运行在应用程序的进程中，在ApplicationThread的scheduleTransaction方法中，调用了ActivityThread的scheduleTransaction方法。

```java
@Override
public void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
    ActivityThread.this.scheduleTransaction(transaction);
}
```

实际调用的是ActivityThread抽象父类ClientTransactionHandler中scheduleTransaction方法，向ActivityThread的H Hander发送了一条EXECUTE_TRANSACTION的Message。

```java
void scheduleTransaction(ClientTransaction transaction) {
    transaction.preExecute(this);
    sendMessage(ActivityThread.H.EXECUTE_TRANSACTION, transaction);
}
```

在ActivityThreade的H的handleMessage方法中

```java
case EXECUTE_TRANSACTION:
    final ClientTransaction transaction = (ClientTransaction) msg.obj;
    mTransactionExecutor.execute(transaction);
    if (isSystem()) {
        // Client transactions inside system process are recycled on the client side
        // instead of ClientLifecycleManager to avoid being cleared before this
        // message is handled.
        transaction.recycle();
    }
    // TODO(lifecycler): Recycle locally scheduled transactions.
    break;
```

调用了TransactionExecutor的execute方法，在execute方法中调用了executeLifecycleState方法，然后调用了cycleToPath方法，然后调用了performLifecycleSequence方法

```java
private void performLifecycleSequence(ActivityClientRecord r, IntArray path) {
    final int size = path.size();
    for (int i = 0, state; i < size; i++) {
        state = path.get(i);
        log("Transitioning to state: " + state);
        switch (state) {
            case ON_CREATE:
                mTransactionHandler.handleLaunchActivity(r, mPendingActions,
                        null /* customIntent */);
                break;
            case ON_START:
                mTransactionHandler.handleStartActivity(r, mPendingActions);
                break;
            case ON_RESUME:
                mTransactionHandler.handleResumeActivity(r.token, false /* finalStateRequest */,
                        r.isForward, "LIFECYCLER_RESUME_ACTIVITY");
                break;
            case ON_PAUSE:
                mTransactionHandler.handlePauseActivity(r.token, false /* finished */,
                        false /* userLeaving */, 0 /* configChanges */, mPendingActions,
                        "LIFECYCLER_PAUSE_ACTIVITY");
                break;
            case ON_STOP:
                mTransactionHandler.handleStopActivity(r.token, false /* show */,
                        0 /* configChanges */, mPendingActions, false /* finalStateRequest */,
                        "LIFECYCLER_STOP_ACTIVITY");
                break;
            case ON_DESTROY:
                mTransactionHandler.handleDestroyActivity(r.token, false /* finishing */,
                        0 /* configChanges */, false /* getNonConfigInstance */,
                        "performLifecycleSequence. cycling to:" + path.get(size - 1));
                break;
            case ON_RESTART:
                mTransactionHandler.performRestartActivity(r.token, false /* start */);
                break;
            default:
                throw new IllegalArgumentException("Unexpected lifecycle state: " + state);
        }
    }
}
```

在这里边满足条件ON_CREATE调用了ClientTransactionHandler的handleLaunchActivity方法，ClientTransactionHandler的实现类就是ActivityThread，即实际调用的就是ActivityThread的handleLaunchActivity方法。

```java
public Activity handleLaunchActivity(ActivityClientRecord r,
        PendingTransactionActions pendingActions, Intent customIntent) {
    ...
    final Activity a = performLaunchActivity(r, customIntent);
    if (a != null) {
        //更新了Activity的状态，置为onResume
            r.createdConfig = new Configuration(mConfiguration);
            reportSizeConfigurations(r);
            if (!r.activity.mFinished && pendingActions != null) {
                pendingActions.setOldState(r.state);
                pendingActions.setRestoreInstanceState(true);
                pendingActions.setCallOnPostCreate(true);
            }
        } else {
            // If there was an error, for any reason, tell the activity manager to stop us.
            try {
                ActivityManager.getService()
                        .finishActivity(r.token, Activity.RESULT_CANCELED, null,
                                Activity.DONT_FINISH_TASK_WITH_ACTIVITY);
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
        }

        return a;
}
```

performLaunchActivity方法

```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    //1.获取ActivityInfo类
    ActivityInfo aInfo = r.activityInfo;
    if (r.packageInfo == null) {
        r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                Context.CONTEXT_INCLUDE_CODE);
    }

    ComponentName component = r.intent.getComponent();
    ...
    //2.创建要启动Activity的上下文环境
    ContextImpl appContext = createBaseContextForActivity(r);
    Activity activity = null;
    try {
        java.lang.ClassLoader cl = appContext.getClassLoader();
        //3.用类加载器创建Activity的实例
        activity = mInstrumentation.newActivity(
                cl, component.getClassName(), r.intent);
        StrictMode.incrementExpectedActivityCount(activity.getClass());
        r.intent.setExtrasClassLoader(cl);
        r.intent.prepareToEnterProcess();
        if (r.state != null) {
            r.state.setClassLoader(cl);
        }
    } catch (Exception e) {
        if (!mInstrumentation.onException(activity, e)) {
            throw new RuntimeException(
                "Unable to instantiate activity " + component
                + ": " + e.toString(), e);
        }
    }

    try {
       	//4.创建Application，makeApplication中会调用创建Application的onCreate方法。
        Application app = r.packageInfo.makeApplication(false, mInstrumentation);
        ...
        if (activity != null) {
            ...
            //5.初始化Activity
            activity.attach(appContext, this, getInstrumentation(), r.token,
                    r.ident, app, r.intent, r.activityInfo, title, r.parent,
                    r.embeddedID, r.lastNonConfigurationInstances, config,
                    r.referrer, r.voiceInteractor, window, r.configCallback);

            //6.调用Activity的onOncreate方法
            if (r.isPersistable()) {
                mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
            } else {
                mInstrumentation.callActivityOnCreate(activity, r.state);
            }
            ...
    return activity;
}
```

启动根Activity的流程涉及到四个进程，Launcher进程、AMS所在进程、Zygote进程、应用自身进程，各进程关系如下：

![activity启动2.png](https://upload-images.jianshu.io/upload_images/10547376-fed4a7718b0292af.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 2. 普通Activity的启动流程

普通Activity启动流程与根Activity启动流程类似，但是一般只涉及到两个进程，即应用程序进程和AMS所在进程(SystemServer进程)。

