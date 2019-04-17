---
layout: post
title: "Android消息总线之EventBus3.0"
subtitle: 'Android消息总线学习'
date:       2019-04-01
author: "Wangxiong"
header-style: text
tags:
  - Android
  - 消息总线
  - EventBus
---
## 1. EventBus简介

EventBus是一个Android事件发布/订阅框架，简化了应用程序内各组件间、组件与后台线程的通信。EventBus可以替代Android传统的Intent、Handler、Broadcast或者接口回调，在Activity、Fragment、Service线程之间传递数据，执行方法。优点是开销小，代码更优雅，发送者和接受者解耦，EventBus不支持跨进程通信。[Github地址](<https://github.com/greenrobot/EventBus>b)

### 1.1 EventBus三要素

- Event：事件，可以是任意类型的对象
- Subscriber：事件订阅者，在EventBus3.0之前消息处理的方法只限定于onEvent、onEventMainThread、onEventBackgroudThread、onEventAsync，分别代表了四种线程模型。在EventBus3.0之后，事件处理的方法可以随意命名，但需要添加一个注解@Subscribe，并且指定线程模型，默认是POSTING。
- Publishe：事件发布者，可以在任意线程任意位置发布事件，直接调用EventBus的post方法，可以自己实例EventBus对象，但一般使用EventBus.getDefault方式获取EventBus对象。

### 1.2 EventBus的四种线程模型

- POSTING(默认)：如果事件处理函数指定线程模型为POSTING，那么该事件在哪个线程发布出来的 ，事件处理函数就好在这个线程运行，也就是发布事件和接受事件在同一个线程。在线程模型为POSTING的事件处理函数中尽量避免执行耗时操作，因为它会阻塞事件的传递，甚至引起ANR.
- MAIN：事件的处理会在UI线程中执行，事件处理的时间不能太长，否则会造成ANR.
- BACKGROUD：如果事件是在UI线程中发布的，那么事件处理函数就会在新的线程中执行，如果事件本身就是子线程发布的，那么事件处理函数直接在发布的线程中执行。在此事件处理函数中禁止进行UI操作。
- AYSNC：无论线程在哪个线程中发布，该事件处理函数都会在新建的子线程中执行。在此事件处理函数中禁止进行UI操作。

## 2. EventBus的使用

使用EventBus需要添加依赖，添加对应的混淆规则。

### 2.1 定义事件类型

```java
public class MessageEvent {

    private String message;

    public MessageEvent(String message) {
        this.message = message;
    }

    public String getMessage() {
        return message;
    }

    public void setMessage(String message) {
        this.message = message;
    }
}
```

### 2.2 订阅事件，处理事件，取消订阅事件

```java
EventBus.getDefault().register(this);

@Subscribe(threadMode = ThreadMode.MAIN)    
public void xxx(MessageEvent messageEvent) {    
    messageEvent.getMessage();    
}

if (EventBus.getDefault().isRegistered(this)) {        
    EventBus.getDefault().unregister(this);        
}
```

### 2.3 发送事件

```java
EventBus.getDefault().post(new MessageEvent("Test eventBus"));
```

## 3. EventBus黏性事件

处理普通事件外，EventBus还支持发送黏性事件，即发送事件后再订阅改事件也能收到该事件，和黏性广播类型。订阅黏性时间和普通事件类型，但需要在事件处理函数@Subscribe注解中添加属性sticky=true。

```java
@Subscribe(threadMode = ThreadMode.MAIN, sticky = true)
public void xxx(MessageEvent messageEvent) {
    messageEvent.getMessage();
}
```

## 4. EventBus源码解析

### 4.1 EventBus实例对象

获取EventBus对象一般通过EventBus.getDefault方法获取。

```java
public static EventBus getDefault() {
    if (defaultInstance == null) {
        synchronized (EventBus.class) {
            if (defaultInstance == null) {
                defaultInstance = new EventBus();
            }
        }
    }
    return defaultInstance;
}
```

EventBus.getDefault方法采用单例模式的双重检查模式设计，调用了EventBus构成方法实例对象。

```java
public EventBus() {
    this(DEFAULT_BUILDER);
}

EventBus(EventBusBuilder builder) {
        subscriptionsByEventType = new HashMap<>();
        typesBySubscriber = new HashMap<>();
        stickyEvents = new ConcurrentHashMap<>();
        mainThreadPoster = new HandlerPoster(this, Looper.getMainLooper(), 10);
        backgroundPoster = new BackgroundPoster(this);
        asyncPoster = new AsyncPoster(this);
        indexCount = builder.subscriberInfoIndexes != null ? builder.subscriberInfoIndexes.size() : 0;
        subscriberMethodFinder = new SubscriberMethodFinder(builder.subscriberInfoIndexes,
                builder.strictMethodVerification, builder.ignoreGeneratedIndex);
        logSubscriberExceptions = builder.logSubscriberExceptions;
        logNoSubscriberMessages = builder.logNoSubscriberMessages;
        sendSubscriberExceptionEvent = builder.sendSubscriberExceptionEvent;
        sendNoSubscriberEvent = builder.sendNoSubscriberEvent;
        throwSubscriberException = builder.throwSubscriberException;
        eventInheritance = builder.eventInheritance;
        executorService = builder.executorService;
    }
```

DEFAULT_BUILDER是默认的EventBusBuilder，用来构建EventBus。

### 4.2 订阅者注册

#### 4.2.1 register方法

```java
public void register(Object subscriber) {
    Class<?> subscriberClass = subscriber.getClass();
    List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
    synchronized (this) {
        for (SubscriberMethod subscriberMethod : subscriberMethods) {
            subscribe(subscriber, subscriberMethod);
        }
    }
}
```

调用subscriberMethodFinder的findSubscriberMethods找到订阅者subscriberClass的所有订阅的方法。然后变量订阅者的订阅方法完成订阅，SubscriberMethod保存了订阅方法的信息，主要有定义方法Method对象、线程模型、事件类型、优先级、是否是黏性事件等属性。

#### 4.2.2 SubscriberMethodFinder的findSubscriberMethods方法

```java
List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
    //1.从缓存中获取SubscriberMethod集合
    List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
    if (subscriberMethods != null) {
        return subscriberMethods;
    }
    //2.ignoreGeneratedIndex表示是否忽略注解器生成的索引
    if (ignoreGeneratedIndex) {
        //3.通过反射获取SubscriberMethod集合
        subscriberMethods = findUsingReflection(subscriberClass);
    } else {
        //4.根据编译时生成的索引查找获取SubscriberMethod集合
        subscriberMethods = findUsingInfo(subscriberClass);
    }
    //5.再获取SubscriberMethod集合后，如果订阅者不存在@Subscribe注解并且为Public的订阅处理方法，则抛出异常。
    if (subscriberMethods.isEmpty()) {
        throw new EventBusException("Subscriber " + subscriberClass
                + " and its super classes have no public methods with the @Subscribe annotation");
    } else {
        //6.存储到缓存中。
        METHOD_CACHE.put(subscriberClass, subscriberMethods);
        return subscriberMethods;
    }
}
```

ignoreGeneratedIndex默认为false，所以会调用findUsingInfo方法。

#### 4.2.3 SubscriberMethodFinder的findUsingInfo方法

```java
private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
    //寻找方法的临时变量封装到了FindState类
    //1.到对象池中获取上下文，避免频繁创造对象，
    FindState findState = prepareFindState();
    //2.初始化寻找对象的上下文
    findState.initForSubscriber(subscriberClass);
    while (findState.clazz != null) { 
        //3.获取定义者了的相关信息，如果没有设置注解处理器，即没有索引，返回的就是null
        findState.subscriberInfo = getSubscriberInfo(findState);
        if (findState.subscriberInfo != null) {
            SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();
            for (SubscriberMethod subscriberMethod : array) {
                if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
                    //目的是为了避免在父类中找到的方法是被子类重写的，保证回调时执行子类的方法。
                    findState.subscriberMethods.add(subscriberMethod);
                }
            }
        } else {
            //4.索引找不到，降级成运行时通过注解和反射去查找。
            findUsingReflectionInSingleClass(findState);
        }
        //5.上下文切换，子类找不到到父类中查找
        findState.moveToSuperclass();
    }
    //6.查找完之后，释放findState到对象池中，并返回查找都得方法。
    return getMethodsAndRelease(findState);
}
```

#### 4.2.4 SubscriberMethodFinder的findUsingReflectionInSingleClass方法

```java
private void findUsingReflectionInSingleClass(FindState findState) {
    Method[] methods;
    try {
        // This is faster than getMethods, especially when subscribers are fat classes like Activities
        methods = findState.clazz.getDeclaredMethods();
    } catch (Throwable th) {
        // Workaround for java.lang.NoClassDefFoundError, see https://github.com/greenrobot/EventBus/issues/149
        methods = findState.clazz.getMethods();
        findState.skipSuperClasses = true;
    }
    for (Method method : methods) {
        int modifiers = method.getModifiers();
        if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
            Class<?>[] parameterTypes = method.getParameterTypes();
            if (parameterTypes.length == 1) {
                Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                if (subscribeAnnotation != null) {
                    Class<?> eventType = parameterTypes[0];
                    if (findState.checkAdd(method, eventType)) {
                        ThreadMode threadMode = subscribeAnnotation.threadMode();
                        findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
                                subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
                    }
                }
            } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                throw new EventBusException("@Subscribe method " + methodName +
                        "must have exactly 1 parameter but has " + parameterTypes.length);
            }
        } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
            String methodName = method.getDeclaringClass().getName() + "." + method.getName();
            throw new EventBusException(methodName +
                    " is a illegal @Subscribe method: must be public, non-static, and non-abstract");
        }
    }
}
```

findUsingReflectionInSingleClass方法主要使用了Java的反射和对注解的解析。

#### 4.2.5 EventBus的subscribe方法

在查找完所有的订阅方法以后便开始遍历方法，调用subscribe方法进行注册。

```java
private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
    Class<?> eventType = subscriberMethod.eventType;
    //1.根据订阅者和订阅方法构建一个Subscription对象。
    Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
    //2.获取当前订阅者和订阅方法的Subscription的list集合
    CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
    if (subscriptions == null) {
        //3.Subscription集合为null则创建集合对象，并保存当前Subscription对象
        subscriptions = new CopyOnWriteArrayList<>();
        subscriptionsByEventType.put(eventType, subscriptions);
    } else {
        //4.Subscription集合已经存在当前Subscription，则抛出异常
        if (subscriptions.contains(newSubscription)) {
            throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                    + eventType);
        }
    }

    int size = subscriptions.size();
    for (int i = 0; i <= size; i++) {
        if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
            //5.遍历订阅事件，找到比当前订阅事件priority小的事件插入，可知Subscription集合从大到下排列。
            subscriptions.add(i, newSubscription);
            break;
        }
    }
    //6.通过订阅者或者该订阅者所以订阅事件方法的集合
    List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
    if (subscribedEvents == null) {
        subscribedEvents = new ArrayList<>();
        typesBySubscriber.put(subscriber, subscribedEvents);
    }
    //7.将当前订阅事件添加到订阅事件subscribedEvents集合
    subscribedEvents.add(eventType);

    //8.黏性事件处理
    if (subscriberMethod.sticky) {
        if (eventInheritance) {
            // Existing sticky events of all subclasses of eventType have to be considered.
            // Note: Iterating over all events may be inefficient with lots of sticky events,
            // thus data structure should be changed to allow a more efficient lookup
            // (e.g. an additional map storing sub classes of super classes: Class -> List<Class>).
            Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
            for (Map.Entry<Class<?>, Object> entry : entries) {
                Class<?> candidateEventType = entry.getKey();
                if (eventType.isAssignableFrom(candidateEventType)) {
                    Object stickyEvent = entry.getValue();
                    checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                }
            }
        } else {
            Object stickyEvent = stickyEvents.get(eventType);
            checkPostStickyEventToSubscription(newSubscription, stickyEvent);
        }
    }
}
```

订阅的代码主要就做了两件事，第一件事是将订阅方法和订阅者封装到subscriptionsByEventType和typesBySubscriber中，subscriptionsByEventType是我们投递订阅事件的时候，就是根据我们的EventType找到我们的订阅事件，从而去分发事件，处理事件的；typesBySubscriber在调用unregister(this)的时候，根据订阅者找到EventType，又根据EventType找到订阅事件，从而对订阅者进行解绑。第二件事，如果是粘性事件的话，就立马投递、执行。

```java
//Key是订阅的事件eventType，Value是Subscription，Subscription是订阅方法和订阅者的包装对象。
private final Map<Class<?>, CopyOnWriteArrayList<Subscription>> subscriptionsByEventType;
//Key是订阅者，Value是订阅者对应的订阅事件eventType的集合
private final Map<Object, List<Class<?>>> typesBySubscriber;
```

#### 4.2.6 流程图

![EventBus-1.png](https://upload-images.jianshu.io/upload_images/10547376-63a14b12688d4de2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 4.3 发送事件

#### 4.3.1 EventBus的post方法

发送事件是调用的EventBus的post方法

```java
public void post(Object event) {
    //1.PostingThreadState保存事件队列和线程状态信息
    PostingThreadState postingState = currentPostingThreadState.get();
    //2.获取事件队列，将当前事件添加到队列中
    List<Object> eventQueue = postingState.eventQueue;
    eventQueue.add(event);

    if (!postingState.isPosting) {
        postingState.isMainThread = Looper.getMainLooper() == Looper.myLooper();
        postingState.isPosting = true;
        if (postingState.canceled) {
            throw new EventBusException("Internal error. Abort state was not reset");
        }
        try {
            //3.处理队列中的所有事件。
            while (!eventQueue.isEmpty()) {
                postSingleEvent(eventQueue.remove(0), postingState);
            }
        } finally {
            postingState.isPosting = false;
            postingState.isMainThread = false;
        }
    }
}
```

Event的post方法首先从PostingThreadState对象中取出事件队列，然后再将当前的事件插入到事件队列当中。最后将队列中的事件依次交由postSingleEvent方法进行处理，并移除该事件。

#### 4.3.2 EventBus的postSingleEvent方法

```java
private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
    Class<?> eventClass = event.getClass();
    boolean subscriptionFound = false;
    if (eventInheritance) {
        List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
        int countTypes = eventTypes.size();
        for (int h = 0; h < countTypes; h++) {
            Class<?> clazz = eventTypes.get(h);
            subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);
        }
    } else {
        subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);
    }
    if (!subscriptionFound) {
        if (logNoSubscriberMessages) {
            Log.d(TAG, "No subscribers registered for event " + eventClass);
        }
        if (sendNoSubscriberEvent && eventClass != NoSubscriberEvent.class &&
                eventClass != SubscriberExceptionEvent.class) {
            post(new NoSubscriberEvent(this, event));
        }
    }
}
```

eventInheritance表示是否向上查找父类事件，默认为true，最后调用postSingleEventForEventType发送事件

#### 4.3.3 EventBus的postSingleEventForEventType方法

```java
private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
    CopyOnWriteArrayList<Subscription> subscriptions;
    synchronized (this) {
        //1.取出改事件对应的subscriptions集合
        subscriptions = subscriptionsByEventType.get(eventClass);
    }
    if (subscriptions != null && !subscriptions.isEmpty()) {
        //2.变量subscriptions集合将Subscription中的订阅者和订阅方法传递给PostingThreadState。
        for (Subscription subscription : subscriptions) {
            postingState.event = event;
            postingState.subscription = subscription;
            boolean aborted = false;
            try {
                //3.调用postToSubscription对事件进行处理
                postToSubscription(subscription, event, postingState.isMainThread);
                aborted = postingState.canceled;
            } finally {
                postingState.event = null;
                postingState.subscription = null;
                postingState.canceled = false;
            }
            if (aborted) {
                break;
            }
        }
        return true;
    }
    return false;
}
```

#### 4.3.4. EventBus的postToSubscription方法

```java
private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
    switch (subscription.subscriberMethod.threadMode) {
        case POSTING:
            invokeSubscriber(subscription, event);
            break;
        case MAIN:
            if (isMainThread) {
                invokeSubscriber(subscription, event);
            } else {
                mainThreadPoster.enqueue(subscription, event);
            }
            break;
        case BACKGROUND:
            if (isMainThread) {
                backgroundPoster.enqueue(subscription, event);
            } else {
                invokeSubscriber(subscription, event);
            }
            break;
        case ASYNC:
            asyncPoster.enqueue(subscription, event);
            break;
        default:
            throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
    }
}
```

在postToSubscription方法中，根据订阅方法的线程模型，提交到相应线程进行发射调用。

#### 4.3.5 流程图

![EventBus-2.png](https://upload-images.jianshu.io/upload_images/10547376-76bd1155bb358c95.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 4.4 订阅者取消注册

#### 4.4.1 EventBus的unregister方法

取消注册订阅是调用的EventBus的unregister方法

```java
public synchronized void unregister(Object subscriber) {
    //1.取出订阅者订阅的对应事件集合。
    List<Class<?>> subscribedTypes = typesBySubscriber.get(subscriber);
    if (subscribedTypes != null) {
        //2.遍历订阅事件集合，调用unsubscribeByEventType解除订阅事件和订阅者关系
        for (Class<?> eventType : subscribedTypes) {
            unsubscribeByEventType(subscriber, eventType);
        }
        //3.移除订阅者
        typesBySubscriber.remove(subscriber);
    } else {
        Log.w(TAG, "Subscriber to unregister was not registered before: " + subscriber.getClass());
    }
}
```

#### 4.4.2 EventBus的unsubscribeByEventType方法

```java
private void unsubscribeByEventType(Object subscriber, Class<?> eventType) {
    //1.根据订阅事件取出订阅者和订阅方法保证对象Subscription
    List<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
    if (subscriptions != null) {
        int size = subscriptions.size();
        for (int i = 0; i < size; i++) {
            Subscription subscription = subscriptions.get(i);
            //2.将订阅者对应的包装对象Subscription移除。
            if (subscription.subscriber == subscriber) {
                subscription.active = false;
                subscriptions.remove(i);
                i--;
                size--;
            }
        }
    }
}
```

#### 4.4.3 流程图

![EventBus-3.png](https://upload-images.jianshu.io/upload_images/10547376-9afdda0e4c610ebb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 5. EventBus总结

### 5.1 核心架构

EvenBus的核心架构就是基于观察者模式实现的 ，EventBus好处比较明显，它能够解耦和，将业务和视图分离，代码实现比较容易。而且3.0后，我们可以通过apt预编译找到订阅者，避免了运行期间的反射处理解析，大大提高了效率。当然EventBus也会带来一些隐患和弊端，如果滥用的话会导致逻辑的分散并造成维护起来的困难。另外大量采用EventBus代码的可读性也会变差。

![EventBus-4.png](https://upload-images.jianshu.io/upload_images/10547376-dfea470831d60c7d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 5.2 核心代码

```java
//1.Key是订阅的事件eventType，Value是Subscription，Subscription是订阅方法和订阅者的包装对象。
private final Map<Class<?>, CopyOnWriteArrayList<Subscription>> subscriptionsByEventType;
//2.Key是订阅者，Value是订阅者对应的订阅事件eventType的集合
private final Map<Object, List<Class<?>>> typesBySubscriber;
//3.黏性事件集合
private final Map<Class<?>, Object> stickyEvents;
```

