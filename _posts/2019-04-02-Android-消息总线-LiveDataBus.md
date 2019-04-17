---
layout: post
title: "Android消息总线之LiveDataBus"
subtitle: 'Android消息总线学习'
date:       2019-04-02
author: "Wangxiong"
header-style: text
tags:
  - Android
  - 消息总线
  - LiveDataBus
---
## 1. LiveDataBus简介

LiveDataBus是基于LiveData的Android消息总线框架，LiveData是Android Architecture Compponents的框架，LiveData是一个可以被观察的数据持有类，可以感知Activity等组件的生命周期。使用LiveDataBus的原因：

- LiveData具有生命周期的感知能力；
- 使用者不需要显示调用反注册；
- 相比EventBus和RxBus代码简单，依赖方支持更好。

## 2. 黏性LiveDataBus

LiveData是黏性的，有个局部变量mVersion初始是-1，当调用了setValue或者postValue其version就会+1；对于每个观察者的封装类ObserverWrapper中的mLastVersion初始也是-1，当LiveData设置ObserverWrapper时，如果LiveData的version大于ObserverWrapper的mLastVersion，LiveData就会强制把当前value推送给Observer。代码实现如下：

```java
public class LiveDataBus {

    private final Map<String, MutableLiveData<Object>> bus;

    private LiveDataBus() {
        this.bus = new HashMap<>();
    }

    private static class SingletonHolder {
        private static final LiveDataBus DATA_BUS = new LiveDataBus();
    }

    public static LiveDataBus getInstance() {
        return SingletonHolder.DATA_BUS;
    }

    public <T> void observeEvent(String eventKey, LifecycleOwner owner, Observer<T> observer) {
        MutableLiveData<T> mutableLiveData = getLiveData(eventKey);
        mutableLiveData.observe(owner, observer);
    }

    public <T> void observeForeverEvent(String eventKey, Observer<T> observer) {
        MutableLiveData<T> mutableLiveData = getLiveData(eventKey);
        mutableLiveData.observeForever(observer);
    }

    public <T> void sendEventInMainThread(String eventKey, T t) {
        MutableLiveData<T> mutableLiveData = getLiveData(eventKey);
        mutableLiveData.setValue(t);
    }

    public <T> void sendEventInBackgroundThread(String eventKey, T t) {
        MutableLiveData<T> mutableLiveData = getLiveData(eventKey);
        mutableLiveData.postValue(t);
    }

    private <T> MutableLiveData<T> getLiveData(String eventKey) {
        if (!bus.containsKey(eventKey)) {
            bus.put(eventKey, new MutableLiveData<>());
        }
        return (MutableLiveData<T>) bus.get(eventKey);
    }

}
```

## 3. 非黏性LiveDataBus

### 3.1 拦截注册事件，包装ObserverWrapper方法

```java
public class LiveDataBus3 {

    private static volatile LiveDataBus3 instance;
    private final ConcurrentHashMap<String, MutableLiveDataBus<Object>> mLiveBus;

    private LiveDataBus3() {
        mLiveBus = new ConcurrentHashMap<>();
    }

    public static LiveDataBus3 getInstance() {
        if (instance == null) {
            synchronized (LiveDataBus3.class) {
                if (instance == null) {
                    instance = new LiveDataBus3();
                }
            }
        }
        return instance;
    }

    public <T> void sendEventInMainThread(String eventKey, T value) {
        MutableLiveData<T> mutableLiveData = getLiveData(eventKey);
        mutableLiveData.setValue(value);
    }

    public <T> void sendEventInBackgroundThread(String eventKey, T value) {
        MutableLiveData<T> mutableLiveData = getLiveData(eventKey);
        mutableLiveData.postValue(value);
    }

    public <T> void observeEvent(String eventKey, LifecycleOwner owner, Observer<T> observer) {
        MutableLiveDataBus<T> mutableLiveData = getLiveData(eventKey);
        mutableLiveData.observe(owner, observer);
    }

    public <T> void observeForeverEvent(String eventKey, Observer<T> observer) {
        MutableLiveDataBus<T> mutableLiveData = getLiveData(eventKey);
        mutableLiveData.observeForever(observer);
    }

    private <T> MutableLiveDataBus<T> getLiveData(String eventKey) {
        if (!mLiveBus.containsKey(eventKey)) {
            mLiveBus.put(eventKey, new MutableLiveDataBus<>(true));
        } else {
            MutableLiveDataBus mutableLiveDataBus = mLiveBus.get(eventKey);
            if (mutableLiveDataBus != null) {
                mutableLiveDataBus.isFirstSubscribe = false;
            }
        }

        return (MutableLiveDataBus<T>) mLiveBus.get(eventKey);
    }

    private static class MutableLiveDataBus<T> extends MutableLiveData<T> {
        private boolean isFirstSubscribe;

        public MutableLiveDataBus(boolean isFirstSubscribe) {
            this.isFirstSubscribe = isFirstSubscribe;
        }

        @Override
        public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<T> observer) {
            super.observe(owner, new ObserverWrapper<>(observer, isFirstSubscribe));
        }

        @Override
        public void observeForever(@NonNull Observer<T> observer) {
            super.observeForever(new ObserverWrapper<>(observer, false));
        }
    }

    private static class ObserverWrapper<T> implements Observer<T> {
        private boolean isChanged;
        private Observer<T> observer;

        public ObserverWrapper(Observer<T> observer, boolean isFirstSubscribe) {
            this.observer = observer;
            this.isChanged = isFirstSubscribe;
        }

        @Override
        public void onChanged(@Nullable T t) {
            if (isChanged) {
                observer.onChanged(t);
            } else {
                isChanged = true;
            }
        }
    }
}
```

### 3.2 拦截注册事件，observe修改version，observeForever判断调用堆栈

```java
public class LiveDataBus2 {

    private final Map<String, MutableLiveDataBus<Object>> bus;

    private LiveDataBus2() {
        this.bus = new HashMap<>();
    }

    private static class SingletonHolder {
        private static final LiveDataBus2 DATA_BUS = new LiveDataBus2();
    }

    public static LiveDataBus2 getInstance() {
        return SingletonHolder.DATA_BUS;
    }

    public <T> void observeEvent(String eventKey, LifecycleOwner owner, Observer<T> observer) {
        MutableLiveDataBus<T> mutableLiveData = getLiveData(eventKey);
        mutableLiveData.observe(owner, observer);
    }

    public <T> void sendEventInMainThread(String eventKey, T value) {
        MutableLiveData<T> mutableLiveData = getLiveData(eventKey);
        mutableLiveData.setValue(value);
    }

    public <T> void sendEventInBackgroundThread(String eventKey, T value) {
        MutableLiveDataBus<T> mutableLiveData = getLiveData(eventKey);
        mutableLiveData.postValue(value);
    }

    public <T> void observeForeverEvent(String eventKey, Observer<T> observer) {
        MutableLiveDataBus<T> mutableLiveData = getLiveData(eventKey);
        mutableLiveData.observeForever(observer);
    }

    private <T> MutableLiveDataBus<T> getLiveData(String key) {
        if (!bus.containsKey(key)) {
            bus.put(key, new MutableLiveDataBus<>());
        }
        return (MutableLiveDataBus<T>) bus.get(key);
    }

    private static class BusObserverWrapper<T> implements Observer<T> {
        private Observer<T> observer;

        private BusObserverWrapper(Observer<T> observer) {
            this.observer = observer;
        }

        @Override
        public void onChanged(@Nullable T t) {
            if (observer != null) {
                //判断是observeForever注册的方式，直接返回。
                if (isCallOnObserveForever()) {
                    return;
                }
                observer.onChanged(t);
            }
        }

        private boolean isCallOnObserveForever() {
            StackTraceElement[] stackTrace = Thread.currentThread().getStackTrace();
            if (stackTrace.length > 0) {
                for (StackTraceElement element : stackTrace) {
                    if ("android.arch.lifecycle.LiveData".equals(element.getClassName()) &&
                            "observeForever".equals(element.getMethodName())) {
                        return true;
                    }
                }
            }
            return false;
        }
    }

    private static class MutableLiveDataBus<T> extends MutableLiveData<T> {
        private Map<Observer<T>, BusObserverWrapper<T>> observableMap = new HashMap<>();

        @Override
        public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<T> observer) {
            super.observe(owner, observer);
            try {
                //反射的方式改变ObserverWrapper的version值和LiveData值相同。
                hook(observer);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }

        @Override
        public void observeForever(@NonNull Observer<T> observer) {
            if (!observableMap.containsKey(observer)) {
                observableMap.put(observer, new BusObserverWrapper<>(observer));
            }
            super.observeForever(Objects.requireNonNull(observableMap.get(observer)));
        }

        @Override
        public void removeObserver(@NonNull Observer<T> observer) {
            if (observableMap.containsKey(observer)) {
                super.removeObserver(Objects.requireNonNull(observableMap.remove(observer)));
            } else {
                super.removeObserver(observer);
            }

        }

        private void hook(Observer<T> observer) throws Exception {
            Class<LiveData> liveDataClass = LiveData.class;

            Field fieldObservers = liveDataClass.getDeclaredField("mObservers");
            fieldObservers.setAccessible(true);
            Object objectObservers = fieldObservers.get(this);

            Class<?> classObservers = objectObservers.getClass();
            Method methodGet = classObservers.getDeclaredMethod("get", Object.class);
            methodGet.setAccessible(true);

            Object objectWrapperEntry = methodGet.invoke(objectObservers, observer);
            Object objectWrapper = null;
            if (objectWrapperEntry instanceof Map.Entry) {
                objectWrapper = ((Map.Entry) objectWrapperEntry).getValue();
            }
            if (objectWrapper == null) {
                throw new NullPointerException("Wrapper cant not be null");
            }
            Field fieldVersion = liveDataClass.getDeclaredField("mVersion");
            fieldVersion.setAccessible(true);
            Object objectVersion = fieldVersion.get(this);

            Class<?> classObserverWrapper = objectWrapper.getClass().getSuperclass();
            Field fieldLastVersion = classObserverWrapper.getDeclaredField("mLastVersion");
            fieldLastVersion.setAccessible(true);
            fieldLastVersion.set(objectWrapper, objectVersion);
        }
    }

}
```

## 4. 使用方法

注册事件和发送事件的key对应的Value类型不能改变，否则类型转换异常。

```java
//1.注册事件
LiveDataBus.getInstance().observeEvent("key_test", this, new Observer<MessageEvent>() {
    @Override
    public void onChanged(@Nullable MessageEvent event) {

    }
});
//2.发送事件
LiveDataBus.getInstance().sendEventInMainThread("key_test", new MessageEvent("aaaa"));
```

源码地址：<https://github.com/wang-xiong/WxApp/tree/master/app_eventbus>