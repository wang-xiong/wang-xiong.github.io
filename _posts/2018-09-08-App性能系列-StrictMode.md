## 1. 严格模式分为两种策略

- 线程监控策略（ThreadPolicy）：抓取可能造成长执行时间的存取行为。如磁盘读写，数据库存取，主要在主线程上监控
- VM虚拟机监控策略（VMPolicy）：抓出可能造成内存泄漏的代码。

## 2. 为何主线程上的磁盘读写会产生手机卡顿问题？

- 磁盘的并发处理速度慢。当短时间多个应用同时进行磁盘读写，读写速度将明显加长，甚至达“秒级”。
- 手机长期使用后，不但存量变少且磁盘碎片化问题严重。过低的磁盘存量或碎片化将大幅降低磁盘读写速度。

## 3. 解决掉严格模式问题有什么好处

- 减缓手机越用越慢或是突然卡顿的问题

- 提升应用启动时间及减少ANR发生概率。举例来说，数据库存取时间过长会连带拉长应用启动时间，甚至是产生ANR成因之一。数据库存取时间长有一大部分来自以下三种原因：

  1.数据库资料持续增加，造成查询/新增/删除/修改时间拉长。

  2.数据库存取遭遇其他应用同时存取磁盘，而拉长完成时间。

  3.等待被其他应用以同步锁锁住的数据库。

- 严格模式能查出跨进程通讯而产生的磁盘读写，远比以代码评审来找有效率。

## 4. 分析严格模式问题日志

### 4.1 常见的严格模式日志样式

![strictmode1.png](https://upload-images.jianshu.io/upload_images/10547376-0c2234ef155f974e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- duration: 严格模式造成的时间延迟(蓝字)

  基于不产生blocking code为原则，安卓认定严格模式问题始于磁盘存取发生时间点，终于执行线程的looper进入idle。所以，其时间延迟计算是不准。请专注在磁盘存取的解决，而不是以这个延迟时间作为严重性的依据。

- 可由堆栈第一个项判断严格模式问题种类(红字)

  StrictModeDiskWriteViolation （磁盘写入）

  StrictModeDiskReadViolation （磁盘读取）

- 每一个严格模式问题会对应一个堆栈。(斜体)

### 4.2 跨进程调用产生的严格模式问题的日志样式

![strictmode2.png](https://upload-images.jianshu.io/upload_images/10547376-8eeac691e3e4d70a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- duration: 严格模式造成的时间延迟(蓝字)

  一般主要专注在磁盘存取的解决，而不是以请勿这个延迟时间作为严重性的依据。

- 可由堆栈第一个项判断严格模式问题种类(红字)

  在堆栈中出现“#via Binder call with stack” (绿字)字样代表该严格模式问题来自为跨进程调用。

## 5. 常见的严格模式问题

### 5.1 来自应用自行触发的磁盘存取

解决方法：

- 去掉拿掉不需要的代码或者已经不再维护的功能。
- 以不需要磁盘读取的方式实现。例如系统应用以systemproperties取代数据库存储资料。假如是必要的磁盘存取，将其移到子线程执行。但须注意以下子线程的设置。
- 建议以HanderThread或AsyncTask，让子线程中的磁盘读取以队列方式执行。
- AsyncTask优先级设置默认是Process.THREAD_PRIORITY_BACKGROUND。假如以默认的优先级执行使用AsyncTask读取重要性高的资料时，其将被分配到较少比例的处理器资源，而影响处理速度。建议处理重要资料时，先调整AsyncTask的优先级为Process.THREAD_PRIORITY_DEFAULT。但仍然不能提高为更高的优先级设置。
- 数据库存取可考虑利用AsyncQueryHandler简化子线程存取数据库实作。AsyncQueryHandler的默认优先级设置已经是Process.THREAD_PRIORITY_DEFAULT，不需更动。
- 非必要的话，避免以直接产生新线程类(new Thread())来执行磁盘存取。这方法容易犯造成线程泄漏的错误。

### 5.2 使用SharedPreferences读写资料

SharedPreferences的架构中需要写入对应的XML文件与应用的文件夹以保存其内容，也因此产生磁盘存取于执行的线程中。解决方法是使用SharedPreferences.Editor.apply()取代SharedPreferences.Editor.commit()以避免在主线程上读取磁盘。Android提供SharedPreferences.Editor.apply()与SharedPreferences.Editor.commit()作为写入对应的XML文件的接口。不同的是commit()直接产生磁盘存取于执行的线程中。 apply()则产生子线程来执行磁盘存取。如果使用commit()执行于的主线程便会产生严格模式问题。

### 5.3 系统级应用送非“保护性广播”

系统级应用送非“保护性广播”，触发system server写入Dropbox，Android认为有一些广播只能由系统发送的，并且提供了标记让系统级应用在AndroidManifest.xml明确宣告。在系统运作起来之后，如果某个不具有系统权限的应用试图发送“保护性广播” ，AMS会抛出异常，提示"Permission Denial: not allowed to send broadcast"。系统级应用指的是Persistent应用或userid为SYSTEM_UID/PHONE_UID/SHELL_UID/BLUETOOTH_UID/NFC_UID的应用。反之，系统级应用发出的broadcast必须宣告成”保护性广播”，否则会触发systemserver记录WTF时间而产生写入dropbox的磁盘存取。解决方法为将所用代码中会发出的广播名字符串，在应用的AndroidManifest.xml明确宣告成保护性广播

## 6. 广义的严格模式问题

严格模式关注的问题本质上在主线程上执行时间过长，便会让用户感到卡顿，甚至产生ANR。但影响因素是多面向的，不是只有不当的磁盘存取问题而已。应用每个动作执行时间的长度都可能影响应用响应时间，进而让用户感到卡顿。我们可以从以下的事件日志(event log)中，审视可能的长执行时间发生点。

- 界面启动时间 am_activity_launch_time

- 数据库查询时间 content_query_sample

  与严格模式互为对照，确认应用执行的数据库存取

- 数据库内容更新时间 content_update_sample

  与严格模式互为对照，确认应用执行的数据库存取

- 跨进程通讯执行时间 binder_sample

- 线程被锁定时时间 dvm_lock_sample

  打印事件日志 adb logcat –b events

### 6.1 am_activity_launch_time(界面启动时间)

```xml
1551  1573 | am_activity_launch_time:
[0,248545817,cn.example.wx/com.android.browser.BrowserLauncher,4432,4432]
```

日志格式：界面名+时间+时间，其中第一个时间：框架触发这个界面启动直到应用画完第一帧的时间，第二个时间：假如该界面做了activity forwarding或是在启动界面期间在，启动另一个界面并自我结束，结束时间则会以最后一个界面画完为结束点。界面启动时间过长，不仅用户会感到卡顿，严重会导致Key Dispatch Timeout ANR。

解决方法：

- 优化在主线程的长执行时间的代码
- 将磁盘读写移到子线程

### 6.2 content_query_sample(数据库查询时间)

当数据库查询时间>慢执行时间门槛值500ms会印出，当数据库查询时间<500ms则随机印出数据库查询时间，日志格式如下

```x&#39;m
content_query_sample:[content://mms-sms/threadID?recipient=1310000000399,_id,,,170,cn.example.databackup,35
```

数据库的Uri+数据库查询指令细节Selected fields(* 则打印成空字符串),where sortby + 执行数据库查询的时间+假如数据库查询是执行在主线程上，会显示的应用名称；在子线程上则是空字符串 +相对与慢执行时间门槛值的占比

解决方法：

- 将在主线程上的数据库查询，移至子线程。
- 若是该数据库查询结果将显示给用户，则在子线程上的数据库查询建议少于1000ms
- 若是该数据库查询仅供后台之用，则在子线程上的数据库查询也因少于2000ms，以避免其他应用被数据库同步锁延迟过久。

### 6.3 content_update_sample(数据库内容更新时间)

当数据库更新时间>慢执行时间门槛值500ms会印出，当数据库更新时间<500ms则随机印出，一般日志中更新指令有：insert、bulkinsert 、delete 、update。

```x&#39;m
content_update_sample:[content://cn.example.wx/alarm,bulkinsert,,165,cn.example.databackup,34
```

数据库的Uri+数据库查询指令细节Selected fields(* 则打印成空字符串),where sortby + 执行数据库查询的时间+假如数据库查询是执行在主线程上，会显示的应用名称；在子线程上则是空字符串 +相对与慢执行时间门槛值的占比

解决方法：

- 将在主线程上的数据库更新，移至子线程。
- 若是该数据库更新随后将显示给用户，则在子线程上的数据库查询+数据库更新建议少于1000ms
- 若是该数据库更新仅供后台之用，则在子线程上的数据库查询也因少于2000ms，以避免其他应用被数据库同步锁延迟过久。

### 6.4 binder_sample(跨进程通讯执行时间)

跨进程通讯执行时间计算始自客户端执行接口，终于客户端收到服务端回复；当跨进程执行时间 >慢执行时间门槛值500ms 会印出，跨进程执行时间 <500ms则随机印出

```x&#39;m
binder_sample:[com.android.internal.telephony.ITelephony,40,2531,com.Qunar,100]
```

执行跨进程通信的客户端类名+跨进程执行的接口编号+跨进程执行时间+执行跨进程通讯的客户端进程名+相对与慢执行时间门槛值的占比

解决方法：

优化在服务端的长执行时间的代码。例如磁盘读写，缩短执行流程等

跨进程执行事件编号查询

AIDL（Android Interface Define Language），是android用以定义客户端和服务端之间实现进程间通信接口。AIDL在编译阶段先被编译成一IInterface类为基础的Java代码。在Java代码中，每个接口被皆被分派一个接口编号，这编号便是binder_sample上的接口编号，如何找到接口编号？(以ITelephony的接口编号40为例)

•ROM编译完后在所有的aidl编译出的对应Java代码放在out/target/common/obj 下。直接在该文件夹以及子文件夹，以grep查找类名（例如ITelephony）。文件名因为类名.java(例如ITelephony.java)，注意一个特例，IActivityManager类与源代码一起放在framework/base/core/java/android/app/IActivityManager.java

- 1. 打开ITelephony.java文件
- 2. 找到FIRST_CALL_TRANSATION位置
- 3. 所有的接口编号都由IBinder的第一个接口编号1号起算。在本例中，Itelephony的第40号接口，要找android.os.IBinder.FIRST_CALL_TRANSATION+39

#### 6.5 (dvm_lock_sample)同步锁锁定时间

同步锁锁定时间>慢执行时间门槛值500ms会印出,同步锁锁定时间<500ms则随机印出

```
dvm_lock_sample:[com.android.settings,1,main,45,ManageApplications.java,1317,ApplicationsState.java,323,9]
```

进程名+该线程是否为StrictMode会监控的对象+等待同步锁的线程名+等待同步锁的代码所在文件及行号+执行同步锁锁定的代码所在文件及行号；若是锁定与等待同步锁代码在同一个文件的话，显示空字符+相对与慢执行时间门槛值的占比

解决方法: 

审查同步锁的必要性，拿掉不需要的同步锁。假如是必要的，优化执行同步锁子线程需要持续抓住同步锁的时间。