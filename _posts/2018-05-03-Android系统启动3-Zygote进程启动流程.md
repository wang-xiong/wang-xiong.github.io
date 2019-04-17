---
layout: post
title: "「Android系统启动」三、Zygote进程启动流程"
subtitle: 'Android系统启动过程学习'
date:       2018-05-03
author: "Wangxiong"
header-style: text
tags:
  - Android
  - Android系统启动
---

## 1.zygote简介

在Android系统中，DVM(Dalivk虚拟机)，应用程序进程已经运行的系统关键服务的SystemServer进程都是Zygote进程创建的，也称zygote为孵化器。zygote通过fork的方式创建应用进程和SystemServer进程，因为zygote进程启动时会创建DVM，因此fork创建的进程在内部都可以获取一个DVM的实例拷贝。

## 2.AppRuntime

init启动zygote时调用了app_main.cpp的main函数中的AppRuntime的start启动zygote进程，代码位置*frameworks/base/cmds/app_process/app_main.cpp*

```c
int main(int argc, char* const argv[])
{
...

    AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));
   ...
     Vector<String8> args;
    if (!className.isEmpty()) {
        args.add(application ? String8("application") : String8("tool"));
        runtime.setClassNameAndArgs(className, argc - i, argv + i);
    } else {
        // We're in zygote mode.
        maybeCreateDalvikCache();
        if (startSystemServer) {
            //1.startSystemServer为true的话(默认为true)，将”start-system-server”放入启动的参数args
            args.add(String8("start-system-server"));
        }
        char prop[PROP_VALUE_MAX];
        if (property_get(ABI_LIST_PROPERTY, prop, NULL) == 0) {
            LOG_ALWAYS_FATAL("app_process: Unable to determine ABI list from property %s.",
                ABI_LIST_PROPERTY);
            return 11;
        }
        String8 abiFlag("--abi-list=");
        abiFlag.append(prop);
        args.add(abiFlag);
        for (; i < argc; ++i) {
            args.add(String8(argv[i]));
        }
    }
    if (!niceName.isEmpty()) {
        runtime.setArgv0(niceName.string());
        set_process_name(niceName.string());
    }
    if (zygote) {
        //2.调用runtime的start函数来启动zygote进程，并将args传入，这样启动zygote进程后，zygote进程会将SystemServer进程启动
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
    } else if (className) {
        runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
    } else {
        fprintf(stderr, "Error: no class name or --zygote supplied.\n");
        app_usage();
        LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
        return 10;
    }
}
```

AppRuntime声明也在app_main.cpp中，它继承AndroidRuntime，也就是我们调用start其实是调用AndroidRuntime的start函数，代码位置：*frameworks/base/core/jni/AndroidRuntime.cpp*

```c
void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote)
{
    ...
    /* start the virtual machine */
    JniInvocation jni_invocation;
    jni_invocation.Init(NULL);
    JNIEnv* env;
    //1.调用startVm函数创建DVM
    if (startVm(&mJavaVM, &env, zygote) != 0) {
        return;
    }
    onVmCreated(env);
    //2.调用startReg函数为DVM注册JNI
    if (startReg(env) < 0) {
        ALOGE("Unable to register all android natives\n");
        return;
    }
    jclass stringClass;
    jobjectArray strArray;
    jstring classNameStr;

    stringClass = env->FindClass("java/lang/String");
    assert(stringClass != NULL);
    //创建数组
    strArray = env->NewObjectArray(options.size() + 1, stringClass, NULL);
    assert(strArray != NULL);
    //从app_main的main函数得知className为com.android.internal.os.ZygoteInit
    classNameStr = env->NewStringUTF(className);
    assert(classNameStr != NULL);
    env->SetObjectArrayElement(strArray, 0, classNameStr);

    for (size_t i = 0; i < options.size(); ++i) {
        jstring optionsStr = env->NewStringUTF(options.itemAt(i).string());
        assert(optionsStr != NULL);
        env->SetObjectArrayElement(strArray, i + 1, optionsStr);
    }
    char* slashClassName = toSlashClassName(className);
    jclass startClass = env->FindClass(slashClassName);
    if (startClass == NULL) {
        ALOGE("JavaVM unable to locate class '%s'\n", slashClassName);
        /* keep going */
    } else {
    	//3.找到ZygoteInit的main函数
        jmethodID startMeth = env->GetStaticMethodID(startClass, "main", "([Ljava/lang/String;)V");
        if (startMeth == NULL) {
            ALOGE("JavaVM unable to find main() in '%s'\n", className);
            /* keep going */
        } else {
        	//4.通过JNI调用ZygoteInit的main函数
            env->CallStaticVoidMethod(startClass, startMeth, strArray);
			//if 0
            if (env->ExceptionCheck())
                threadExitUncaughtException(env);
			//endif
        	}
    	}
  	...
}
```

通过JNI调用ZygoteInit的main函数，因为ZygoteInit的main函数是Java编写的，因此需要通过JNI调用。

## 3.Zygote的Java框架层

代码位于：*frameworks/base/core/java/com/android/internal/os/ZygoteInit.java*

```java
public static void main(String argv[]) {
    ...
    try {
        ...       
        //1.注册Zygote用的Socket
        registerZygoteSocket(socketName);
        ...
        //2.预加载类和资源
        preload();
        ...
        if (startSystemServer) {
        	//3.启动SystemServer进程
            startSystemServer(abiList, socketName);
        }
        Log.i(TAG, "Accepting command socket connections");
        //4.等待客户端请求
        runSelectLoop(abiList);
        closeServerSocket();
    } catch (MethodAndArgsCaller caller) {
        caller.run();
    } catch (RuntimeException ex) {
        Log.e(TAG, "Zygote died with exception", ex);
        closeServerSocket();
        throw ex;
    }
}
```

ZygoteInit的mian函数主要做了四件事

### 3.1 调用registerZygoteSocket函数

调用registerZygoteSocket函数创建了一个Server端的Socket，socket的name为zygote，当Zygote进程将SystemServer进程启动后，就会在这个服务端的Socket上来等待ActivityManagerService请求Zygote进程来创建新的应用程序进程。

```java
private static void registerZygoteSocket(String socketName) {
     if (sServerSocket == null) {
         int fileDesc;
         final String fullSocketName = ANDROID_SOCKET_PREFIX + socketName;
         try {
             String env = System.getenv(fullSocketName);
             fileDesc = Integer.parseInt(env);
         } catch (RuntimeException ex) {
             throw new RuntimeException(fullSocketName + " unset or invalid", ex);
         }
         try {
             FileDescriptor fd = new FileDescriptor();
             fd.setInt$(fileDesc);
             //1.创建一个LocalServerSocket，也就是Server端的socket
             sServerSocket = new LocalServerSocket(fd);
         } catch (IOException ex) {
             throw new RuntimeException(
                     "Error binding to local socket '" + fileDesc + "'", ex);
         }
     }
```

### 3.2 调用preload函数

调用preload函数用来预加载类和资源。

### 3.3 调用startSystemServer函数

调用startSystemServer函数启动SystemServer进程，startSystemServer函数的代码如下所示。

```java
 private static boolean startSystemServer(String abiList, String socketName)
            throws MethodAndArgsCaller, RuntimeException {
     ...
     /* Hardcoded command line to start the system server */
     //1.创建args数组，SystemServer进程的用户id和用用户组id都设置为1000，并且拥有用户组1001-1010，1018,1021,1032，3001-3010的权限。进程名为system_server，启动类名为com.android.server.SystemServer。
     String args[] = { "--setuid=1000", "--setgid=1000"
         , "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1021,1032,3001,3002,3003,3006,3007,3009,3010"
         , "--capabilities=" + capabilities + "," + capabilities
         , "--nice-name=system_server", "--runtime-args", "com.android.server.SystemServer",
                     };
     ZygoteConnection.Arguments parsedArgs = null;

     int pid;
        
     try {
         //2.封装args为parsedArgs对象。
         parsedArgs = new ZygoteConnection.Arguments(args);
         ZygoteConnection.applyDebuggerSystemProperty(parsedArgs);
         ZygoteConnection.applyInvokeWithSystemProperty(parsedArgs);
            
         //3.调用Zygote的forkSystemServer函数在当前进程创建一个子进程，如果返回的pid为0，表示在新创建的子进程中执行
         pid = Zygote.forkSystemServer(
             parsedArgs.uid, parsedArgs.gid,
             parsedArgs.gids,
             parsedArgs.debugFlags,
             null,       
             parsedArgs.permittedCapabilities,       
             parsedArgs.effectiveCapabilities);   
     } catch (IllegalArgumentException ex) {   
         throw new RuntimeException(ex);   
     }
     //如果返回的pid为0，表示在新创建的子进程中执行   
     if (pid == 0) {
         if (hasSecondZygote(abiList)) {
             waitForSecondaryZygote(socketName);
         } 
         //4.执行handleSystemServerProcess函数启动SystemServer进程。
         handleSystemServerProcess(parsedArgs);  
     }   
     return true;   
 }
```

### 3.4 调用runSelectLoop函数

启动SystemServer进程后，调用runSelectLoop函数

```java
private static void runSelectLoop(String abiList) throws MethodAndArgsCaller {    
    ArrayList<FileDescriptor> fds = new ArrayList<FileDescriptor>();
    ArrayList<ZygoteConnection> peers = new ArrayList<ZygoteConnection>();
    //1.sServerSocket是在registerZygoteSocket函数中创建的服务端Socket，调用sServerSocket.getFileDescriptor()用来获得该Socket的fd字段的值并添加到fd列表fds中。
    fds.add(sServerSocket.getFileDescriptor());    
    peers.add(null);
    //2.无限循环用来等待ActivityManagerService请求Zygote进程创建新的应用程序进程
    while (true) {    
        StructPollfd[] pollFds = new StructPollfd[fds.size()];
        //3.通过遍历fds存储的信息转移到pollFds数组中。
        for (int i = 0; i < pollFds.length; ++i) {
            pollFds[i] = new StructPollfd();
            pollFds[i].fd = fds.get(i);
            pollFds[i].events = (short) POLLIN;   
        }
        try {
            Os.poll(pollFds, -1);
        } catch (ErrnoException ex) {
            throw new RuntimeException("poll failed", ex);  
        }
        //4.遍历pollFds，
        for (int i = pollFds.length - 1; i >= 0; --i) {
            if ((pollFds[i].revents & POLLIN) == 0) {
                continue;  
            }
            //5.如果i==0则说明，服务端socket与客户端连接上，也就是zygote进程与ActivityManagerService建立了连接
            if (i == 0) {
                //6.调用acceptCommandPeer函数得到ZygoteConnection类对象，添加到socket连接列表peers中，并将ZygoteConnection的fd添加到fd列表fds中，以便可以接收到ActivityManagerService的请求。
                ZygoteConnection newPeer = acceptCommandPeer(abiList);
                peers.add(newPeer);
                fds.add(newPeer.getFileDesciptor());
            } else {
                //6.如果i值大于0，则说明ActivityManagerService向Zygote进程发送了一个创建应用进程的请求，因此调用ZygoteConnection的runOnce函数来创建一个新的应用程序进程。并在成功创建后将这个连接从Socket连接列表peers和fd列表fds中清除。
                boolean done = peers.get(i).runOnce();
                if (done) {
                    peers.remove(i);
                    fds.remove(i);   
                }   
            }   
        }   
    }
}
```

4.zygote进程总结

zygote进程总共做了如下几件事

1. 创建AppRuntime并调用start方法，启动zygote进程。
2. 创建DVM并为DVM注册JNI
3. 通过调用JNI调用ZygoteInit的mian函数进入Zygote的Java框架层。
4. 通过registerZygoteSocket函数创建服务端Socket。
5. 启动SystemServer进程。
6. 并通过runSelectLoop函数等待ActivityManagerService的请求来创建新的应用程序进程。