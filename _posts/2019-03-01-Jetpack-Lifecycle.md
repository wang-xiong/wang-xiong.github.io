[Android Jetpack](https://developer.android.google.cn/jetpack)组件是Google推出的一套应用软件组件，分为四个方面基础、架构、行为、界面。

## 1. Lifecycle简介

Lifecycle组件是指android.arch.lifecycle包下边提供的各种类和接口，可以让构建能够感知其他组件生命周期的类，如Activity、Fragment。

## 2. Lifecycle在MVP中的使用

MVP中的Presenter需要感知Activity或者Fragment的生命周期，一般情况下回在义的基类IPresent接口添加对应的Activity什么周期，然后在Activity各个生命周期中回掉对应的方法。如果使用Lifecycle组件，只需要将IPresent接口继承LifecycleObserver接口，然后在对应的方法添加@OnLifecycleEvent注解指明各个生命周期，然后在Activity的onCreate方法中调用getLifecycle().addObserver方法添加监听对象即可。

```java
public interface IPresenter extends LifecycleObserver {

    @OnLifecycleEvent(Lifecycle.Event.ON_CREATE)
    void onCreate(LifecycleObserver owner);

    @OnLifecycleEvent(Lifecycle.Event.ON_CREATE)
    void onDestroy(LifecycleObserver owner);
}
```

## 3. 源码分析

### 3.1 注册过程

分析Activity的源码可知，getLifecycle()方法是LifecycleOwner接口中的方法，Activity的父类SupportActivity实现了LifecycleOwner接口，返回了LifecycleRegistry对象，LifecycleRegistry类继承自Lifecycle抽象类。即addObserver是在LifecycleRegistry中实现的。

```java
public void addObserver(@NonNull LifecycleObserver observer) {
    State initialState = mState == DESTROYED ? DESTROYED : INITIALIZED;
    //1.构建ObserverWithState对象，传入观察者observer。
    ObserverWithState statefulObserver = new ObserverWithState(observer, initialState);
    ObserverWithState previous = mObserverMap.putIfAbsent(observer, statefulObserver);

    if (previous != null) {
        return;
    }
    LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
    if (lifecycleOwner == null) {
        // it is null we should be destroyed. Fallback quickly
        return;
    }

    boolean isReentrance = mAddingObserverCounter != 0 || mHandlingEvent;
    State targetState = calculateTargetState(observer);
    mAddingObserverCounter++;
    while ((statefulObserver.mState.compareTo(targetState) < 0
            && mObserverMap.contains(observer))) {
        pushParentState(statefulObserver.mState);
        //2.调用ObserverWithState的dispatchEvent方法，通知状态变化
        statefulObserver.dispatchEvent(lifecycleOwner, upEvent(statefulObserver.mState));
        popParentState();
        // mState / subling may have been changed recalculate
        targetState = calculateTargetState(observer);
    }

    if (!isReentrance) {
        // we do sync only on the top level.
        sync();
    }
    mAddingObserverCounter--;
}
```

```java
static class ObserverWithState {
    State mState;
    GenericLifecycleObserver mLifecycleObserver;

    ObserverWithState(LifecycleObserver observer, State initialState) {
        mLifecycleObserver = Lifecycling.getCallback(observer);
        mState = initialState;
    }

    void dispatchEvent(LifecycleOwner owner, Event event) {
        State newState = getStateAfter(event);
        mState = min(mState, newState);
        mLifecycleObserver.onStateChanged(owner, event);
        mState = newState;
    }
}
```

ObserverWithState类中会调用Lifecycling.getCallback(observer)方法，生成真正的观察者包装对象，然后在通知变化的时候调用onStateChanged方法。

```java
static GenericLifecycleObserver getCallback(Object object) {
    //1.第一种方式，继承DefaultLifecycleObserver接口，因为DefaultLifecycleObserver接口继承自FullLifecycleObserver接口，
    if (object instanceof FullLifecycleObserver) {
        return new FullLifecycleObserverAdapter((FullLifecycleObserver) object);
    }
    //2.第二种方式，继承自GenericLifecycleObserver接口。
    if (object instanceof GenericLifecycleObserver) {
        return (GenericLifecycleObserver) object;
    }

    final Class<?> klass = object.getClass();
    int type = getObserverConstructorType(klass);
    //3.第三种方式，使用注解的方式
    if (type == GENERATED_CALLBACK) {
        List<Constructor<? extends GeneratedAdapter>> constructors =
                sClassToAdapters.get(klass);
        if (constructors.size() == 1) {
            GeneratedAdapter generatedAdapter = createGeneratedAdapter(
                    constructors.get(0), object);
            return new SingleGeneratedAdapterObserver(generatedAdapter);
        }
        GeneratedAdapter[] adapters = new GeneratedAdapter[constructors.size()];
        for (int i = 0; i < constructors.size(); i++) {
            adapters[i] = createGeneratedAdapter(constructors.get(i), object);
        }
        return new CompositeGeneratedAdaptersObserver(adapters);
    }
    return new ReflectiveGenericLifecycleObserver(object);
}
```

Lifecycling.getCallback方法会返回GenericLifecycleObserver对象。Lifecycle使用有三种方式

- 第一种方式

  实现DefaultLifecycleObserver接口，并实现DefaultLifecycleObserver接口的对应方法。然后在FullLifecycleObserverAdapter对象的onStateChanged进行分发调用对应方法。

- 第二种方式

  直接实现GenericLifecycleObserver接口，并在onStateChanged方法进行Lifecycle.Event判断调用。

- 第三种方式

  使用@onLifecycleEvent注解方式，注解处理器会将注解动态生成为GeneratedAdapter子类的代码。这个子类会生成callMethods方法，在方法中根据对应Lifecycle.Event调用不同的方法Observer的方法，终通过GenericLifecycleObserver的onStateChanged方法调用生成的GeneratedAdapter的callMechods方法进行事件分发。

### 3.2 分发流程

```java
@Override
@SuppressWarnings("RestrictedApi")
protected void onCreate(@Nullable Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    ReportFragment.injectIfNeededIn(this);
}
```

ReportFragment.injectIfNeededIn(this);目的是向Activity中添加一个Fragment，Fragment能够感知Activity的生命周期，然后将事件分发给实现LifecycleObserver接口的观察者。然后在ReportFragment的各个生命周期将Lifecycle.Event.xxx事件调用dispatch方法分发。

```java
private void dispatch(Lifecycle.Event event) {
    Activity activity = getActivity();
    if (activity instanceof LifecycleRegistryOwner) {
        ((LifecycleRegistryOwner) activity).getLifecycle().handleLifecycleEvent(event);
        return;
    }

    if (activity instanceof LifecycleOwner) {
        Lifecycle lifecycle = ((LifecycleOwner) activity).getLifecycle();
        if (lifecycle instanceof LifecycleRegistry) {
            ((LifecycleRegistry) lifecycle).handleLifecycleEvent(event);
        }
    }
}
```

调用LifecycleRegistry的handleLifecycleEvent方法通知事件更新

```java
public void handleLifecycleEvent(@NonNull Lifecycle.Event event) {
    State next = getStateAfter(event);
    moveToState(next);
}
```

## 4. 总结

- LifecycleOwner：生命周期的事件发起者，在Activity/Fragment他们的生命周期发生变化时发出响应的Event给LifecycleRegistry。
- LifecycleObserver：生命周期的观察者，通过注解将处理函数与希望监听的Event绑定，当相应的Event发生时，LifecycleRegistry会通知相应的 函数进行处理。
- LifecycleRegistry：控制中心，它负责控制state的转换，接受分发 event事件。