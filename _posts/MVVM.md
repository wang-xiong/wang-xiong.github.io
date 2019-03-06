目前主流的三种架构模式：MVC MVP MVVM

目的是为了解决应用程序复杂的逻辑问题，通过应用相关架构模式，职责分理，不同测层次做不同的事情。

MVC(Model-View-Controller)

Model：数据模型，View：视图模型，Controller：控制器

Controller和View都依赖于Modle，VIew和Controller互相依赖

MVP(Model-VIew-Presenter)

Presenter：负责完成View和Model直接的交互，Model：业务逻辑和实体模型，View：负责VIew的控制以及与用户的交互

MVVM(Model-View-ViewModel)

MVP的一种变革，核心思想：数据模型数据双向绑定。

ViewModel会有一个叫Binder，或者是Data-binging engine的东西。MVP中Presenter负责VIew和Model之间数据同步操作全部交由Binder处理，只需要在View中声明View显示的内容是和哪一块数据绑定的，当ViewModel更新Model是，Binder会自动绑数据更新到View上，当用户操作View时，也会自动把数据更新到Model上。这种方式称为双向绑定。

Android官方推出的DataBinding便是一个双向绑定的库。

## Lifecycle

Lifecycle 组件指的是 android.arch.lifecycle 包下提供的各种类与接口，可以让开发者构建能感知其他组件（主要指Activity 、Fragment）生命周期（lifecycle-aware）的类。

## ViewModel

ViewModel用来管理和UI交互的数据，通常情况下会在Activity的onCreate()方法获取ViewModel，此后无论onCreate()调用多少次，获取到的ViewModel都是同一个实例。ViewModel是一个抽象类，中间只有一个可选择实现的方法onCleared()，该方法会在Activity或者Fragment的onDestory()被调用，用于回收资源。

## LiVeData

参考文档：https://developer.android.google.cn/topic/libraries/architecture/livedata

优势：

1.能够确保数据和UI统一
LiveData采用了观察者模式，当数据发生变化时，主动通知被观察者。

2.没有内存泄露，onDestory的时候自动解绑。

3.当Activity停止时不会引起崩溃

当Activity组件处于inactive非活动状态时，它不会收到LiveData数据变化的通知。

4.不需要手动处理生命周期的变化

5.确保总能获取到最新的数据

6.configuration changes时，不需要额外的处理来保存数据我们知道，当你把数据存储在组件中时，当configuration change（比如语言、屏幕方向变化）时，组件会被recreate，然而系统并不能保证你的数据能够被恢复的。当我们采用LiveData保存数据时，因为数据和组件分离了。当组件被recreate，数据还是存在LiveData中，并不会被销毁。

7.通过继承LiveData类，然后将该类定义成单例模式，在该类封装监听一些系统属性变化，然后通知LiveData的观察者。

基于观察者模式构建的

订阅：调用observe，observeForever方法注册

发布：调用setValue，postValue方法发布数据

```java
@MainThread
public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<T> observer) {
    //如果LifecycleOwner的生命周期已经DESTROYED，没有必要观察，直接返回。
    if (owner.getLifecycle().getCurrentState() == DESTROYED) {
        // ignore
        return;
    }
        // 创建继承了GenericLifecycleObserver的LifecycleBoundObserver，并且将这个LifecycleBoundObserver 
    // 存进观察者集合mObservers中统一管理
    LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
    ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
    if (existing != null && !existing.isAttachedTo(owner)) {
        throw new IllegalArgumentException("Cannot add the same observer"
                + " with different lifecycles");
    }
    if (existing != null) {
        return;
    }
    // 调用observe()这个方法添加的observer，只有Activity/Fragment等生命周期组件可见时
    // 才会收到数据更新的通知，为了知道什么时候Activity/Fragment是可见的，这里需要注册到
    // Lifecycle中感知生命周期
    // 也是因为这个，observe()比observeForever()多了一个参数lifecycleOwner
    owner.getLifecycle().addObserver(wrapper);
}
```

```java
private abstract class ObserverWrapper {
    final Observer<T> mObserver;
    boolean mActive;
    int mLastVersion = START_VERSION;

    ObserverWrapper(Observer<T> observer) {
        mObserver = observer;
    }

    abstract boolean shouldBeActive();

    boolean isAttachedTo(LifecycleOwner owner) {
        return false;
    }

    void detachObserver() {
    }

    void activeStateChanged(boolean newActive) {
        if (newActive == mActive) {
            return;
        }
        // immediately set active state, so we'd never dispatch anything to inactive
        // owner
        mActive = newActive;
        boolean wasInactive = LiveData.this.mActiveCount == 0;
        LiveData.this.mActiveCount += mActive ? 1 : -1;
       // LiveData.this.mActiveCount表示处于active状态的observer的数量
        // 当mActiveCount大于0时，LiveData处于active状态
        // 注意区分observer的active状态和 LiveData 的active状态
        if (wasInactive && mActive) {
              // inactive -> active
            onActive();
        }
        if (LiveData.this.mActiveCount == 0 && !mActive) {
            // mActiveCount在我们修改前等于1，也就是说，LiveData从active
            // 状态变到了inactive
            onInactive();
        }
        
        //如果是active状态，通知观察者数据变化（dispatchingValue方法在下一节发布中分析）
        if (mActive) {
            dispatchingValue(this);
        }
    }
}
```

```java
@MainThread
protected void setValue(T value) {
    //判断当前是否是主线程。如果不是直接抛异常
    assertMainThread("setValue");
    //每次更新value，都会使mVersion+1
    mVersion++;
    //将这次数据保存在LiveData的mData变量中。mData的值永远是最新的值
    mData = value;
    //发布
    dispatchingValue(null);
}
```

```java
class LifecycleBoundObserver extends ObserverWrapper implements GenericLifecycleObserver {
    @NonNull final LifecycleOwner mOwner;

    LifecycleBoundObserver(@NonNull LifecycleOwner owner, Observer<T> observer) {
        super(observer);
        mOwner = owner;
    }

    @Override
    boolean shouldBeActive() {
        return mOwner.getLifecycle().getCurrentState().isAtLeast(STARTED);
    }

    // 这个方法是由Lifecycle结构中的mLifecycleRegistry所调用，一旦LifecycleOwner的生命周期
    // 发生变化，都会调用到onStateChanged这个方法进行生命周期转换的通知
    @Override
    public void onStateChanged(LifecycleOwner source, Lifecycle.Event event) {
        if (mOwner.getLifecycle().getCurrentState() == DESTROYED) {
            removeObserver(mObserver);
            return;
        }
        
        // 一开始创建LifecycleBoundObserver实例的时候，mActive默认为false，
        // 当注册到Lifecycle后，Lifecycle会同步生命周期状态给我们（也就是回调本方法）
        activeStateChanged(shouldBeActive());
    }

    @Override
    boolean isAttachedTo(LifecycleOwner owner) {
        return mOwner == owner;
    }

    @Override
    void detachObserver() {
        mOwner.getLifecycle().removeObserver(this);
    }
}
```

```java
//initiator为null，表示将新数据发布到所以得observer
//如果initiator不为null，表示只通知给传入的观察者initiator
private void dispatchingValue(@Nullable ObserverWrapper initiator) {
    if (mDispatchingValue) {
        mDispatchInvalidated = true;
        return;
    }
    // 在observer的回调里面又触发了数据的修改
    // 设置mDispatchInvalidated为true后，可以让下面的循环知道
    // 数据被修改了，从而开始一轮新的迭代。
    //
    // 比方说，dispatchingValue -> observer.onChanged -> setValue -> dispatchingValue
    // 这里return的是后面那个dispatchingValue，然后在第一个
    // dispatchingValue会重新遍历所有的observer，并调用他们的onChanged。
    //
    // 如果想避免这种情况，可以在回调里面使用 postValue 来更新数据
    mDispatchingValue = true;
    do {
        mDispatchInvalidated = false;
        if (initiator != null) {
            considerNotify(initiator);
            initiator = null;
        } else {
            for (Iterator<Map.Entry<Observer<T>, ObserverWrapper>> iterator =
                    mObservers.iteratorWithAdditions(); iterator.hasNext(); ) {
                considerNotify(iterator.next().getValue());
                if (mDispatchInvalidated) {
                    break;
                }
            }
        }
    } while (mDispatchInvalidated);
    mDispatchingValue = false;
}
```

为了防止循环调用，我们在调用客户代码前先置位一个标志（mDispatchingValue），结束后再设为 false。如果在回调里面又触发了这个方法，可以通过 mDispatchingValue 来检测。

检测到循环调用后，再设置第二个标志（mDispatchInvalidated），然后返回。返回又会回到之前的调用，前一个调用通过检查 mDispatchInvalidated，知道数据被修改，于是开始一轮新的迭代。

```java
private void considerNotify(ObserverWrapper observer) {
    // 如果observer的状态不是active，那么不向该observer通知，直接返回
    if (!observer.mActive) {
        return;
    }
    // Check latest state b4 dispatch. Maybe it changed state but we didn't get the event yet.
    //
    // we still first check observer.active to keep it as the entrance for events. So even if
    // the observer moved to an active state, if we've not received that event, we better not
    // notify for a more predictable notification order.
    if (!observer.shouldBeActive()) {
        observer.activeStateChanged(false);
        return;
    }
    // 上面的源码分析，我们知道每一次setValue或者postValue的调用都会是mVersion自增1，
    // mLastVersion的作用是为了与mVersion作比较，这个比较作用主要有两点：
    // 1.如果说mLastVersion >= mVersion，证明这个观察者已经接受过本次发布事件通知，不需要重复通知了，直接返回
    // 2.实现粘性事件。比如有一个数据（LiveData）在A页面setValue()之后，则该数据（LiveData）中的
    // 全局mVersion+1,也就标志着数据版本改变，然后再从A页面打开B页面，在B页面中开始订阅该LiveData，
    // 由于刚订阅的时候内部的数据版本都是从-1开始，此时内部的数据版本就和该LiveData全局的数据
    // 版本mVersion不一致，根据上面的原理图，B页面打开的时候生命周期方法一执行，则会进行notify，
    // 此时又同时满足页面是从不可见变为可见、数据版本不一致等条件，所以一进B页面，B页面的订阅就会被响应一次
    if (observer.mLastVersion >= mVersion) {
        return;
    }
    
    // 设置mLastVersion = mVersion，以免重复通知观察者
    observer.mLastVersion = mVersion;
     // 这里就最终调用了我们一开始通过observe()方法传入的observer中的onChange()方法
    // 即新数据被发布给了observer
    //noinspection unchecked
    observer.mObserver.onChanged((T) mData);
}
```

![image-20190124165526595](/Users/wangxiong/Library/Application Support/typora-user-images/image-20190124165526595.png)