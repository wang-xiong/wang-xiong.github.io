---
layout: post
title: "Jetpack系列」 三、ViewModel"
subtitle: 'Android'
date:       2019-03-03
author: "Wangxiong"
header-style: text
tags:
  - Android
  - Jetpack
---
## 1. ViewModel简介

ViewModel主要用来存储和管理与UI相关的数据，它能够让数据在屏幕旋转等配置信息发生改变导致UI重建的情况下不被销毁。[官方文档](https://developer.android.google.cn/topic/libraries/architecture/viewmodel)

## 2. ViewModel的生命周期

ViewModel的对象存活在系统中不被回收的时间由创建ViewmMoel传递给ViewModelProvider的Lifecycle决定的。ViewModel将一直留在内存中，直到限定其存在时间范围的Lifecycle生命周期结束。对于Activity，是在Activity destroy时；而对于Fragment，是在Fragment detach时。图示如下

![viewmodel-lifecycle.png](https://upload-images.jianshu.io/upload_images/10547376-aa1b5cc4fb340be5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 3. ViewModel源码

### 3.1 ViewModelProvider实例创建

```java
ViewModelProviders.of(this).get(XxxViewModel.class)；
//1.ViewModelProviders.of(this)创建ViewModelProvider对象
//2.viewModelProvider.get(XxxViewModel.class)生成相对的ViewModel对象。
```

```java
public static ViewModelProvider of(@NonNull FragmentActivity activity) {
    return of(activity, null);
}
    
public static ViewModelProvider of(@NonNull Fragment fragment, @Nullable Factory 
factory) {
        Application application = checkApplication(checkActivity(fragment));
        //1.如果Factory为null，生成默认的AndroidViewModelFactory
        if (factory == null) {
            factory = 
            ViewModelProvider.AndroidViewModelFactory.getInstance(application);
        }
        //2.实例一个ViewModelProvider对象返回
        return new ViewModelProvider(fragment.getViewModelStore(), factory);
    }
```

Factory类是一个接口，create方法用于实例ViewModel对象，系统默认提供了AndroidViewModelFactory类 

```java
public interface Factory {
    @NonNull
    <T extends ViewModel> T create(@NonNull Class<T> modelClass);
}
```

```java
public static class AndroidViewModelFactory extends ViewModelProvider.NewInstanceFactory {
    private static AndroidViewModelFactory sInstance;

    @NonNull
    public static AndroidViewModelFactory getInstance(@NonNull Application application) {
        if (sInstance == null) {
            sInstance = new AndroidViewModelFactory(application);
        }
        return sInstance;
    }

    private Application mApplication;

    public AndroidViewModelFactory(@NonNull Application application) {
        mApplication = application;
    }

    @NonNull
    @Override
    public <T extends ViewModel> T create(@NonNull Class<T> modelClass) {
        //1.判断modelClass是否继承自AndroidViewModel
        if (AndroidViewModel.class.isAssignableFrom(modelClass)) {
            //noinspection TryWithIdenticalCatches
            try {
                //2.调用modleClass类中带有Application参数的构造方法创建一个ViewModel返回。
                return modelClass.getConstructor(Application.class).newInstance(mApplication);
            } catch (NoSuchMethodException e) {
                throw new RuntimeException("Cannot create an instance of " + modelClass, e);
            } catch (IllegalAccessException e) {
                throw new RuntimeException("Cannot create an instance of " + modelClass, e);
            } catch (InstantiationException e) {
                throw new RuntimeException("Cannot create an instance of " + modelClass, e);
            } catch (InvocationTargetException e) {
                throw new RuntimeException("Cannot create an instance of " + modelClass, e);
            }
        }
        //3.如果modelClass不是继承自AndroidViewModel，则调用父类NewInstanceFactory的create方法实例ViweModel对象。
        return super.create(modelClass);
    }
}
```

```java
public static class NewInstanceFactory implements Factory {

    @SuppressWarnings("ClassNewInstance")
    @NonNull
    @Override
    public <T extends ViewModel> T create(@NonNull Class<T> modelClass) {
        //noinspection TryWithIdenticalCatches
        try {
            //1.直接通过反射调用modelClass的无参构造函数实例化一个ViewModel对象返回。
            return modelClass.newInstance();
        } catch (InstantiationException e) {
            throw new RuntimeException("Cannot create an instance of " + modelClass, e);
        } catch (IllegalAccessException e) {
            throw new RuntimeException("Cannot create an instance of " + modelClass, e);
        }
    }
}
```

由上源码分析可知，自定义的ViewModel继承ViewmModel需要一个默认无参的构造方法；如果自定义的ViewModel继承自AndroidViewModel。必须要有一个以Application为唯一参数的构造函数 。

### 3.2 ViewModelStore实例创建

创建ViewModelProvider对象时，需要一个ViewModelStore对象，ViewModelStore对象是Fragment或者Activity的getViewModelStore方法返回的。

```java
public ViewModelStore getViewModelStore() {
    if (getApplication() == null) {
        throw new IllegalStateException("Your activity is not yet attached to the "
                + "Application instance. You can't request ViewModel before onCreate call.");
    }
    if (mViewModelStore == null) {
        mViewModelStore = new ViewModelStore();
    }
    return mViewModelStore;
}
```

```java
public class ViewModelStore {
    //1.用HashMap存储ViewModel对象
    private final HashMap<String, ViewModel> mMap = new HashMap<>();

    //2.新增ViewModel,如果对应的key存在，直接替换，并调用原ViewModel的onCleared方法。
    final void put(String key, ViewModel viewModel) {
        ViewModel oldViewModel = mMap.put(key, viewModel);
        if (oldViewModel != null) {
            oldViewModel.onCleared();
        }
    }

    //3.获取指定key的ViewModel对象
    final ViewModel get(String key) {
        return mMap.get(key);
    }

    //4.调用所以ViewModle的onCleared方法，并清空HashMap。
    public final void clear() {
        for (ViewModel vm : mMap.values()) {
            vm.onCleared();
        }
        mMap.clear();
    }
}
```

### 3.3 ViewModel的实例创建

创建了ViewModleProvider对象，并且实例了ViewModelStore对象，然后 调用ViewModleProvider的get方法实例出ViewModel对象。

```java
public <T extends ViewModel> T get(@NonNull String key, @NonNull Class<T> modelClass) {
    //1.从mViewModelStore取ViewModel对象，取到直接返回。
    ViewModel viewModel = mViewModelStore.get(key);

    if (modelClass.isInstance(viewModel)) {
        //noinspection unchecked
        return (T) viewModel;
    } else {
        //noinspection StatementWithEmptyBody
        if (viewModel != null) {
            // TODO: log a warning.
        }
    }
    //2.用调用Factory的create方法创建ViewModel对象
    viewModel = mFactory.create(modelClass);
    //3.将ViewModel对象存储在mViewModelStore中。
    mViewModelStore.put(key, viewModel);
    //noinspection unchecked
    return (T) viewModel;
}
```

### 3.4 ViewModel实例唯一的原理

根据上传源码可知ViewModel对象被保存造ViewModelStore对象中，ViewModelStore又是Fragment和Activity中的局部变量，所以保证ViewModelStore实例唯一即保证ViewModel唯一，ViewModelStore时如何保证唯一的，由下源码可知。

```java
 protected void onCreate(@Nullable Bundle savedInstanceState) {
        mFragments.attachHost(null /*parent*/);

        super.onCreate(savedInstanceState);

        NonConfigurationInstances nc =
                (NonConfigurationInstances) getLastNonConfigurationInstance();
        if (nc != null) {
            //1.onCreate方法从NonConfigurationInstances取出ViewModel
            mViewModelStore = nc.viewModelStore;
        }
     	....
 }
public final Object onRetainNonConfigurationInstance() {
    Object custom = onRetainCustomNonConfigurationInstance();

    FragmentManagerNonConfig fragments = mFragments.retainNestedNonConfig();

    if (fragments == null && mViewModelStore == null && custom == null) {
        return null;
    }

    NonConfigurationInstances nci = new NonConfigurationInstances();
    nci.custom = custom;
    nci.viewModelStore = mViewModelStore;
    nci.fragments = fragments;
    return nci;
}
```

## 4. Fragment共享数据

ViewModelProviders.of(this)方法需要传递一个Activity或者Fragment，因为ViewModel生命周期与其绑定，所以同一个Activity下的Fragment的ViewModel是同一个对象。所以可以ViewModel进行同一个Activity下多个Fragment直接的数据通信。

