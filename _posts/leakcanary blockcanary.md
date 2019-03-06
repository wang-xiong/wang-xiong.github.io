APP性能优化

APM

MAT

leakcanary

blockcanary

BlockCanary

* Android 平台-非侵入式的性能监控组件
* 针对轻微的UI卡顿及不流畅现象的检查工具

UI卡顿

最优策略：60fps——>16ms/帧(一幅图像) ，16ms内是否能完成一次操作

尽量保证每次在16ms内处理完单次的所有的CPU与GPU计算、绘制、渲染等操作，否则会造成丢帧卡顿问题

UI卡顿的原因：

1、UI线程的耗时操作

2、布局Layout过于复杂，无法在16ms内完成渲染

3、View过度绘制，cpu or gpu负载过重。

4、View频繁的触发measure、layout事件，导致在绘制Ui的时间加剧，损耗加剧，造成View频繁渲染。

5、内存频繁触发GC，在同一帧频繁的创建临时变量加剧内存的浪费和渲染的增加。虚拟机在执行GC垃圾回收的时候，会暂停所有的线程包括ui线程。只有当gc垃圾回收器完成工作才能继续开启线程执行工作。



ANR

ANR：Application Not responding 程序未响应，超过预定时间仍然未响应就会造成ANR

检测工具：Activity Manager 和WindowManager 进行监控的

ANR分类：

1、Service Timeout：服务如果在20秒内没有完成执行的话，就会造成ANR

2、BroadcastQueue Timeout：广播如果在10秒内没有完成执行的话，就会造成ANR

3、InputDispatching Timeout：输入事件如果超过了5秒钟就会造成ANR，广播如果在10秒内没有完成执行的话，就会造成ANR

4、ContentProvider Timeout 内容着提供执行超时。

ANR造成的主要原因：

1、主线程做了一些耗时操作，比如网络、数据库获取操作等

2、主线程被其他线程锁住，主线程所需要的资源在被其他线程所使用中，导致主线程无法获取到该资源而造成主线程的阻塞，进而造成ANR现象

3、cpu被其他进程占用，这个进程没有被分配到足够的cpu资源

ANR的解决措施

1、不在主线程读取数据，主线程禁止从网络获取数据，但是可以从数据库获取数据，虽然未被禁止该操作，但是执行这类操作会造成掉帧现象。

2、sharePreference 的commit () /apply()

commit() 是同步方法，apply()是异步方法，所以主线程中尽量不要调用 commit()方法，调用同步方法会阻塞主线程，尽量通过apply()方法执行操作。

3、不要在BroadCastReceiver 的onReceive()方法中执行耗时操作，onReceive（）：也是运行在主线程中的，后台操作，通过IntentService()方法执行相关操作。

4、Activity的生命周期函数中都不应该有太耗时的操作

该生命周期函数大多数执行于主线程中，及时是Service 服务或是 内容提供者ContentProvider也不要在onCreate()中执行耗时操作。

ANR检测工具

在linux 内核下，当Watchdog 启动后，便设定了一个定时器，当出现故障时候，通过会让Android系统重启。由于这种机制的存在，经常会出现一些system_server 进程被watchdog杀掉而发生手机重启的问题。

watchdog初始化

Android 的watchdog 是一个单例线程 ，在System server时候就会初始化watchdog 。在watchdog初始化化时候会构建很多 HandlerChecker ，大致分为两类：

Monitor Checker，用于检查是Monitor对象可能发生的死锁, AMS, PKMS, WMS等核心的系统服务都是Monitor对象。

Looper Checker，用于检查线程的消息队列是否长时间处于工作状态。Watchdog自身的消息队列，Ui, Io, Display这些全局的消息队列都是被检查的对象。此外，一些重要的线程的消息队列，也会加入到Looper Checker中，譬如AMS, PKMS，这些是在对应的对象初始化时加入的。

 两类 HandlerChecker的侧重点不同，Monitor Checker预警我们不能长时间持有核心系统服务的对象锁，否则会阻塞很多函数的运行；Looper Checker 预警我妈不能长时间的霸占消息队列，否则其他消息将得不到处理。这两类都会导致系统卡住 ANR

ANRWatchDog：继承自Thread类，是一个线程。

简单原理：

1、创建一个ANR线程，不断的向Ui线程通过handler post一个runnable任务

2、执行完上面的操作会让线程睡眠固定的时间，给线程执行操作留出时间

3、线程重新开始运行 检测之前的post的任务是在执行了 两个临时变量是否相等，不相等代表Ui线程没有阻塞

4、刚才保存的_tick变量是否等于 刚才开启子线程当中进行run()+1 是否不等于保存的变量。通过对比来查看post runnable 是否已经发送到了主线程，主线程是否已经执行了该消息，执行后_tick ！= lastTick

拓展

new Thread 在Android 开启线程的弊端

​    1、多个耗时任务时就会开多个新线程，开销是非常大的 ，造成很大的性能浪费

​    2、如果在线程中执行循环任务，只能通过一个人为的标识位Flag来控制它的停止

​    3、没有线程切换的接口

​    4、如果从UI线程启动一个子线程，则该线程优先级默认为Default，这样就和Ui线程级别相同了 体现不出开子线程的目的

   通过方法设定线程的优先级,将子线程的优先级将低为后台线程：  Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUD);

线程间的通信

1.Thread和handler方法

2.AsyncTask

内部通过线程池管理线程，通过handler切换Ui 线程和子线程

缺陷：AsyncTask 会持有当前Activity的引用，在使用的时候要把AsyncTask声明为静态static，在AsyncTask内部持有外部Activity的弱引用，预防内存泄漏。

3.0以后默认api设置为串行的原因：当一个进程中开启多个AsyncTask的时候，它会使用同一个线程池执行任务，所以又多个AsyncTask一起并行执行的话，而且要在DIBG中访问相同的资源，这时候就有可能出现数据不安全的情况。设计成串行就不会有多线程数据不安全的问题。

4.使用HandlerThread

继承自Thread 类，适用于单线程或是异步队列场景，耗时不多不会产生较大阻塞的情况比如io流读写操作，并不适合于进行网络数据的获取！！！

优点：

- 有自己内部的Looper对象，
- 通过Looper().prepare()可以初始化looper对象
- 通过Looper().loop（）开启looper循环
- HandlerThread的looper对象传递给Handler对象，然后在handleMessage()方法中执行异步任务

5.使用IntentService

- IntentService 是Service类的子类，拥有service所有的生命周期的方法
- 会单独开启一个线程HandlerThread 来处理所有的Intent请求所对应的任务
- 当IntentService处理完所有的任务后，它会在适当的时候自动结束服务

多进程的优点 与缺陷

1.多进程的优点：

​        1、解决OOM问题——将耗时的工作放在辅助进程中避免主进程出现OOM

​        2、合理利用内存，在适当的时候生成新的进程，在不需要的时候杀掉这个进程

​        3、单一进程崩溃不会影响整个app的使用

​        4、项目解耦、模块化开发有好处

2、多进程的缺陷：

​     1、每次新进程的创建都会创建一个Application，造成多次创建Application

​        解析：根据进程名区分不同的进程，然后进行不同的初始化。不要在Application中进行过度静态变量初始化

​     2、文件读写潜在的问题

​        解析：需要并发访问的文件、本地文件、数据库文件等；多利用接口而避免直接访问文件，文件锁是基于进程和虚拟机的级别，如果从不同的进程访问一个文件锁，这个锁是失效的！（sharePreference)

 3、静态变量和单例模式完全失效

​        解析：在进程中间，我们的内存空间是相互独立的。虚拟方法区内的静态变量也是相互独立的，由于静态变量是基于进程的，所以单例模式会失效。在不同进程间访问同一个相同类的静态变量，他的值也不一定相同

​        尽量避免在多进程中频繁的使用静态变量

​    4、线程同步机制完全失效

​        解析：Java的同步机制也是由虚拟机进行调度的。两个进程会有两个不同的虚拟机。同步关键字都将没用意义

synchronized 和 voliate 的三大区别 ？

   1、阻塞线程与否：

voliate关键字本质上是告诉JVM虚拟机当前的变量在寄存器中的值是不确定的，需要从主存中去获取不会造成线程的阻塞，当单例对象被修饰成voliate后，每一次instance内存中的读取都会从主内存中获取，而不会从缓存中获取，这样就解决了双重效验锁单例模式的缺陷

synchronized关键字指明的代码块只有当前线程可以访问它的临界区的资源，其他的线程就会被阻塞住

 2、使用范围

voliate关键字 只是修饰变量的

synchronized关键字不仅可以修饰变量还可以修饰方法

3、原子性-操作不会再分（不会因为多线程造成操作顺序的改变）

voliate关键字不具备原子性

synchronized关键字可以保证变量的原子性

 

 4.9  voliate关键字和单例写法

​    单例模式：饿汉、懒汉、线程安全的 分析其中的问题 

bugly

./gradlew app:dependencies



内存泄露(Memory Leak)

内存泄漏是指程序中己动态分配的堆内存由于某种原因程序未释放或无法释放，造成系统内存的浪费，导致程序运行速度减慢甚至系统崩溃等严重后果。



内存泄露与内存溢出

内存溢出是内存分配达到了最大值，而内存泄露是无用的内存占用内存堆；因此内存泄露也是内存溢出的一个原因。



JVM如何判断一个对象是垃圾对象

JVM采用可达遍历算法来判断一个对象是否是垃圾对象，如果对象是可达的，则认为该对象是被引用的，GC不会回收，如果一个对象或者多个对象组成的对象块是不可达的，那么GC会回收该垃圾对象。



leakcanary

1.在`Activity/Fragment`的`onDestroy`方法添加检测监听

2.通过`ReferenceQueue`+`WeakReference`+`手动调用 GC判断是否回收，获取未回收的对象。

3.确定未被回收的对象是否被其他对象引用，即是否存在内存泄露。通过`VMDebug` + `HAHA` 确定

VM 会有堆内各个对象的引用情况，并能以`hprof`文件导出。HAHA 是一个由 square 开源的 Android 堆分析库，分析 `hprof` 文件生成`Snapshot`对象。`Snapshot`用以查询对象的最短引用链。

4.找到最短引用链，定位问题，排查代码