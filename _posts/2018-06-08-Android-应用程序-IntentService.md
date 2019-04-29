---
title: "Android应用程序之IntentenService"
subtitle: 'Android应用程序学习'
date:       2018-06-08
author: "Wangxiong"
header-style: text
tags:
  - Android
  - Android应用程序
---
## 1. IntentenService源码解读

```java
public abstract class IntentService extends Service {
    private volatile Looper mServiceLooper;
    private volatile ServiceHandler mServiceHandler;
    private String mName;
    private boolean mRedelivery;

    private final class ServiceHandler extends Handler {
        public ServiceHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            //1.执行任务
            onHandleIntent((Intent)msg.obj);
            //2.停止Service
            stopSelf(msg.arg1);
        }
    }
    
    public IntentService(String name) {
        super();
        mName = name;
    }
    
    public void setIntentRedelivery(boolean enabled) {
        mRedelivery = enabled;
    }

    @Override
    public void onCreate() {
        super.onCreate();
        HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
        thread.start();

        mServiceLooper = thread.getLooper();
        mServiceHandler = new ServiceHandler(mServiceLooper);
    }

    @Override
    public void onStart(@Nullable Intent intent, int startId) {
        Message msg = mServiceHandler.obtainMessage();
        msg.arg1 = startId;
        msg.obj = intent;
        mServiceHandler.sendMessage(msg);
    }

    @Override
    public int onStartCommand(@Nullable Intent intent, int flags, int startId) {
        onStart(intent, startId);
        return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
    }

    @Override
    public void onDestroy() {
        mServiceLooper.quit();
    }

    @Override
    @Nullable
    public IBinder onBind(Intent intent) {
        return null;
    }
    
    @WorkerThread
    protected abstract void onHandleIntent(@Nullable Intent intent);
}
```

## 2. IntentService原理

- Service默认是应用一个进程中，Service运行在主线程中，不能在Service中做耗时任务。
- IntentService继承了Service并且是一个抽象类，包含Service全部特性和生命周期，可以做耗时任务，并且任务结束会自动停止。
- 在onCreate方法中，创建了一个子线程HanlerThread，并且创建了一个子线程的ServiceHandler
- 所有onStartCommand传递的Intent都发送给ServiceHandler处理，ServiceHandler的handleMessage方法中调用onHandleIntent方法处理任务，并且调用stopSelf(msg.arg1)停止服务。
- 默认onBind方法返回null。

