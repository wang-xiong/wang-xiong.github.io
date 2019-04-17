---
layout: post
title: "Android应用程序之Activiy界面加载流程"
subtitle: 'Android应用程序学习'
date:       2018-05-12
author: "Wangxiong"
header-style: text
tags:
  - Android
  - Android应用程序
---
在Activity中调用setContentView方法就可以将布局加载显示，通过阅读Activity的setContentView方法源码，学习界面加载显示原理。

## 1. setContentView流程

在Activity的onCreate方法调用setContentView方法，进行布局添加。

```java
public void setContentView(@LayoutRes int layoutResID) {
    getWindow().setContentView(layoutResID);
    initWindowDecorActionBar();
}
```

setContentView方法中调用了getWindow()的方法返回的是PhoneWindow的实例，查看PhoneWindow的setContentView方法。

### 1.1 PhoneWindow的setContentView方法

```java
@Override
public void setContentView(int layoutResID) {
    if (mContentParent == null) {
        installDecor();
    } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        mContentParent.removeAllViews();
    }

    if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID, getContext());
        transitionTo(newScene);
    } else {
        mLayoutInflater.inflate(layoutResID, mContentParent);
    }
    mContentParent.requestApplyInsets();
    final Callback cb = getCallback();
    if (cb != null && !isDestroyed()) {
        cb.onContentChanged();
    }
    mContentParentExplicitlySet = true;
}
```

PhoneWindow的setContentView方法中，判断mContentParent为空，调用installDecor()；最后调用mLayoutInflater.inflate(layoutResID, mContentParent)将布局加载到mContentParent中

### 1.2 PhoneWindow的installDecor方法

```java
private void installDecor() {
    mForceDecorInstall = false;
    if (mDecor == null) {
        mDecor = generateDecor(-1);
        mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
        mDecor.setIsRootNamespace(true);
        if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
            mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
        }
    } else {
        mDecor.setWindow(this);
    }
    if (mContentParent == null) {
        mContentParent = generateLayout(mDecor);
    }
}
```

### 1.3 PhoneWindow的generateDecor方法

在PhoneWindow的installDecor方法中，新启动的ActivitymDecor、和mContentParent都为空；继续查看generateDecor方法和generateLayout方法。

```java
protected DecorView generateDecor(int featureId) {
    //.... 
    return new DecorView(context, featureId, this, getAttributes());
}
```

PhoneWindow的generateDecor方法实例了DecorView对象并返回，DecorView继承自FrameLayout。

### 1.4 PhoneWindow的generateLayout方法

```java
protected ViewGroup generateLayout(DecorView decor) {
    // Apply data from current theme.
    //1.获取Window的Style属性
    TypedArray a = getWindowStyle();
    //...
    requestFeature(FEATURE_NO_TITLE);
    //...
    requestFeature(FEATURE_ACTION_BAR_OVERLAY);
    //...
    setFlags(FLAG_LAYOUT_IN_OVERSCAN, FLAG_LAYOUT_IN_OVERSCAN&(~getForcedWindowFlags()));
    //...
    //2.获取Window所设置的feature
    int features = getLocalFeatures();
    //...根据features确定要加载的布局layoutResource
    //4.实例layoutResource布局，并add到mDecor中
    mDecor.onResourcesLoaded(mLayoutInflater, layoutResource);
    //5.找到contentParent布局
    ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
}
```

generateLayout方法中，首先会获取Window的Style属性，然后调用一大堆的requestFeature方法和setFlags方法，对Window的状态属性进行设置；调用getLocalFeatures获取Window所设置的features，根据features加载不同的布局到mDecor中，实例contentParent View并返回。

## 2.handleResumeActivity流程

### 2.1 handleResumeActivity方法

在Activity的onResume后，调用了ActivityThread的handleResumeActivity。

```java
@Override
public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
        String reason) {
    //...
    if (r.window == null && !a.mFinished && willBeVisible) {
        r.window = r.activity.getWindow();
        View decor = r.window.getDecorView();
        decor.setVisibility(View.INVISIBLE);
        ViewManager wm = a.getWindowManager();
        WindowManager.LayoutParams l = r.window.getAttributes();
        a.mDecor = decor;
        l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
        l.softInputMode |= forwardBit;
        //...
        if (a.mVisibleFromClient) {
            if (!a.mWindowAdded) {
                a.mWindowAdded = true;
                wm.addView(decor, l);
```

在handleResumeActivity方法中，会获取ViewManager对象，ViewManager类负责对View进行管理，然后调用wm.addView(decor, l)方法加载decor。WindowManager接口继承自ViewManger接口。WindowManager接口的具体实现是WindowManagerImpl，WindowManagerImpl内部通过调用代理类WindowManagerGlobal类实现相应功能。WindowManagerGlobal是一个单例，也就是一个进程只有一个WindowManagerGlobal对象服务于所以页面的View。

### 2.2 WindowManagerGlobal的addView方法

```java
public void addView(View view, ViewGroup.LayoutParams params,
        Display display, Window parentWindow) {
    //...
    ViewRootImpl root;
    //...
    //1.创建ViewRootImpl对象
    root = new ViewRootImpl(view.getContext(), display);        
    view.setLayoutParams(wparams);
    //2.将Window所对应的View、ViewRootImpl、LayoutParams顺序添加在WindowManager中
    mViews.add(view); //所有Window对象中的View   
    mRoots.add(root); //所有Window对象中的View所对应的ViewRootImpl 
    mParams.add(wparams);//所有Window对象中的View所对应的布局参数
    //3.最后把Window对应的view设置给新创建的ViewRootImpl对象中，通过ViewRootImpl完成Window的添加过程
    root.setView(view, wparams, panelParentView);
}
```

WindowManagerGlobal的addView方法保存了Window中的View，VieRootImpl以及View所对应的参数，并且创建了ViewRootImpl，然后ViewRootImpl绑定Window所对应的View，并对该View进行测量、布局、绘制等。

### 2.3 ViewRootImpl的setView方法

```java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
    //...
    //1.Window在添加完成之前先进行一次布局，确保后续能再接受到系统其它事件之后重新布局，对View进行异步刷新，执行View的绘制方法。
    // Schedule the first layout -before- adding to the window            
    // manager, to make sure we do the relayout before receiving            
    // any other events from the system.            
    requestLayout();
    //...
    //将该Window添加到屏幕。mWindowSession实现了IWindowSession接口，它是Session的客户端Binder对象。addToDisplay是一次AIDL的跨进程通信，通知WindowManagerService添加IWindow
    res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes, getHostVisibility(), mDisplay.getDisplayId(), mWinFrame, mAttachInfo.mContentInsets, mAttachInfo.mStableInsets, mAttachInfo.mOutsets, mAttachInfo.mDisplayCutout, mInputChannel);
    //...
}
```

在ViewRootImpl的setView方法方法中，调用了requestLayout进行布局绘制 。

### 2.4 ViewRootImpl的requestLayout方法

```java
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        checkThread();
        mLayoutRequested = true;
        scheduleTraversals();
    }
}
```

ViewRootImpl的requestLayout方法调用了scheduleTraversals方法。

### 2.5 ViewRootImpl的scheduleTraversals方法

```java
void scheduleTraversals() {
    //1.一个刷新周期只执行一次即可，屏蔽其他的刷新请求
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        //2.设置同步障碍Message
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
         //3.屏幕刷新信号VSYNC 监听回调把mTraversalRunnable（执行doTraversal()；push到主线程了且是个异步Message会优先得到执行 ，具体看下Choreographer的实现
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        if (!mUnbufferedInputDispatch) {
            scheduleConsumeBatchedInput();
        }
        notifyRendererOfFramePending();
        pokeDrawLockIfNeeded();
    }
}
```

在ViewRootImpl的方法中发送了同步障碍消息 ，屏幕刷新信号VSYNC 监听回调把mTraversalRunnable。

### 2.6 ViewRootImpl的mTraversalRunnable

```java
final class TraversalRunnable implements Runnable {
    @Override
    public void run() {
        doTraversal();
    }
}

final TraversalRunnable mTraversalRunnable = new TraversalRunnable();
```

同步障碍消息内执行了doTraversal方法。

### 2.7 ViewRootImpl的doTraversal方法 

```java
void doTraversal() {
    if (mTraversalScheduled) {
        mTraversalScheduled = false;
        //1.移除同步障碍Message
mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);

        if (mProfile) {
            Debug.startMethodTracing("ViewAncestor");
        }
        //2.真正执行decorView的绘制
        performTraversals();

        if (mProfile) {
            Debug.stopMethodTracing();
            mProfile = false;
        }
    }
}
```

在ViewRootImpl的doTraversal方法中 ，首先移除了之前设置的同步障碍Message，然后调用performTraversals方法

### 2.8 ViewRootImplperformTraversals方法

```java
private void performTraversals() {
    // cache mView since it is used so much below...
    final View host = mView;
    //... 
    if (mFirst) {
        //...
        //performTraversals第一次调用时，调用decorView的dispatchAttachedToWindow方法，设置mAttachInfo变量。
        host.dispatchAttachedToWindow(mAttachInfo, 0);
    }
   
    //...
    // Ask host how big it wants to be                
    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
    //...
    performLayout(lp, mWidth, mHeight);
    //...
    mFirst = false;
    //...
    performDraw();
    //...
    
```

在ViewRootImplperformTraversals方法中，依次调用了performLayout、performMeasure、performDraw方法。

- ViewRootImpl调用performMeasure执行Window对应的View的测量，调用View的measure方法。
- ViewRootImpl调用performLayout执行Window对应的View的布局，调用View的layout方法。
- ViewRootImpl调用performDraw执行Window对应的View的布局，调用View的draw方法。