---
layout: post
title: "「IPC系列」二、Binder机制"
subtitle: 'Android 系统Binder机制'
date:       2018-06-11
author: "Wangxiong"
header-style: text
tags:
  - Android
  - IPC
---
## 1. 认识Binder

- 从类来讲Binder是Android中的一个类，实现了IBinder接口
- 从IPC角度来讲，Binder是Android中一种跨进程的通信方式，Linux没有
- 从Android Framework角度来讲，Binder是ServiceManger连接各种Manager和ManagerService的桥梁
- 从Android应用层来说，Binder是客户端和服务端进行通信的媒介，当bindService时，服务端返回一个Binder对象，客户端利用Binder对象获取服务端的服务或者数据。

## 2. 使用Binder的原因

Android是基于Linux系统的，Linux系统的内置IPC方式有管道、信号量、共享内存、消息队列、Socket。但是Android使用Binder机制，原因在于

1. 功能上，Android系统为了向开发者提供丰富的功能，广泛的使用了Client-Service通信方式，如多媒体、传感器等，利用Client-Service的模式应用程序只需要作为client端，与这些service建立连接，便可以使用这些服务。而在Linux的IPC方式中只有Socket支持这种通信方式。
2. 通信方式上，Android系统希望使用Client-Service通信方式，但是只有Socket的支持这种通信方式。
3. 传输性能上，Socket作为一款通用接口，传输效率低，开销大，主要用在跨网络的进程间通信和本机上进程间的低速通信；消息队列和管道采用存储-转发的方式，即数据先从发送方拷贝到开辟的内核缓存区，再从内核缓存区拷贝到接收方缓存区，至少要两次拷贝；共享内存虽然不需要拷贝过程，但是控制复杂；而Binder只需要拷贝一次。
4. 从安全性上，Linux传统的IPC方式没有任何的安全措施，完全依赖上传协议来确保，具体表现在以下两点：第一、传统IPC接受方无法获取（UID/PID），从而无法鉴别身份，使用传统的IPC只能由用户在数据包中携带UID/PID，但是不安全容易被恶意程序拦截；第二、传统的IPC的访问接入点是开发的，无法建立私有通信，只要知道这些接入点的程序都可以喝对应端建立连接。

## 3. Binder的组成

Binder由四部分组成：Binder客户端，Binder服务端，Binder驱动，Service Manager(服务登记查询模块)

- Binder客户端

  Binder客户端是想要使用服务的进程

- Binder服务端

  Binder服务端是提供服务的进程。

- Binder驱动：

  客户端先通过Binder拿到一个服务端进程的一个对象的引用，通过这个引用直接调用方法。在这个引用对象执行方式时，是先将方法的调用请求传递给Binder驱动，然后Binder驱动再将请求传递给服务端进程，服务端进程收到请求后，调用服务端真正的对象来执行所调用的方法，得出结果，将结果发给Binder驱动，Binder驱动再将结果发送给客户端，最后客户端得到结构，Binder驱动相当于中转者的角色，通过这个中转者调用其他进程的对象。

- Service Manager

  调用其他进程首先要获取这个对象，这个对象其实就是代表了另一个进程能提供的方法。首先服务端进程需要登记注册，告诉系统我有个对象可以公开给其他进程提供服务。当客户端进程需要这个服务时，就去这个登记的地方通过查询获取这个对象。

## 4. Binder机制简述

![Android-IPC1.jpg](https://upload-images.jianshu.io/upload_images/10547376-a7f2685deec75e52.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如上图所示进程A通过聚合BpBinder的接口代理"BpInterface"，调用BpInterface的函数完成远程调用，进程B则通过Binder响应方BBinderd的继承类接口实现提BnInterface来响应进程A发送的来的请求，如此依赖，在远程调用的操作中，客户端需要完成Binder代理和接口代理，服务端需要完成接口的实现体。其中BpBinder、BpInterface、BBinder和BnInterface都是C++中的层次概念。

## 5. Binder的工作流程

由于进程的隔离性，Client不能直接读取Service的内容，但是内核可以，而Binder驱动就运行在内核中，因此通过Binder驱动进行中转。为了实现功能的单一性，为Client何Server分别设置一个代理，Client的代理Proxy，Service的代理Stub。这样Client进程中的Proxy通过Binder驱动与Service进程中的Stub进行数据交流。Client直接调用Proxy这个聚合了Binder的类，一般服务端会使用一些列的Manager来屏蔽掉Service的Binder实现细节，Client直接调用Manager的方法获取数据，这样Client就不需要知道Binder具体怎么工作。客户端如何获取服务端的代理或者Manager，就是通过Service Manager进程获取的，Service Manager是第一个启动的服务，其他的服务可以在Service Manager中注册。

![Android-IPC2.jpg](https://upload-images.jianshu.io/upload_images/10547376-130028c55f5ec22d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 6. Binder的示例

AIDL和基于AIDL的Messager涉及到了进程间的通信，AIDL文件生成的类

```java
package my.itgungnir.ipc.binder;

public interface IBookManager extends android.os.IInterface {
    /**
     * Local-side IPC implementation stub class.
     */
    public static abstract class Stub extends android.os.Binder implements my.itgungnir.ipc.binder.IBookManager {
        private static final java.lang.String DESCRIPTOR = "my.itgungnir.ipc.binder.IBookManager";

        /**
         * Construct the stub at attach it to the interface.
         */
        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }

        /**
         * Cast an IBinder object into an my.itgungnir.ipc.binder.IBookManager interface,
         * generating a proxy if needed.
         */
        public static my.itgungnir.ipc.binder.IBookManager asInterface(android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (((iin != null) && (iin instanceof my.itgungnir.ipc.binder.IBookManager))) {
                return ((my.itgungnir.ipc.binder.IBookManager) iin);
            }
            return new my.itgungnir.ipc.binder.IBookManager.Stub.Proxy(obj);
        }

        @Override
        public android.os.IBinder asBinder() {
            return this;
        }

        @Override
        public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
            switch (code) {
                case INTERFACE_TRANSACTION: {
                    reply.writeString(DESCRIPTOR);
                    return true;
                }
                case TRANSACTION_getBook: {
                    data.enforceInterface(DESCRIPTOR);
                    my.itgungnir.ipc.binder.Book _result = this.getBook();
                    reply.writeNoException();
                    if ((_result != null)) {
                        reply.writeInt(1);
                        _result.writeToParcel(reply, android.os.Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
                    } else {
                        reply.writeInt(0);
                    }
                    return true;
                }
            }
            return super.onTransact(code, data, reply, flags);
        }

        private static class Proxy implements my.itgungnir.ipc.binder.IBookManager {
            private android.os.IBinder mRemote;

            Proxy(android.os.IBinder remote) {
                mRemote = remote;
            }

            @Override
            public android.os.IBinder asBinder() {
                return mRemote;
            }

            public java.lang.String getInterfaceDescriptor() {
                return DESCRIPTOR;
            }

            @Override
            public my.itgungnir.ipc.binder.Book getBook() throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                my.itgungnir.ipc.binder.Book _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    mRemote.transact(Stub.TRANSACTION_getBook, _data, _reply, 0);
                    _reply.readException();
                    if ((0 != _reply.readInt())) {
                        _result = my.itgungnir.ipc.binder.Book.CREATOR.createFromParcel(_reply);
                    } else {
                        _result = null;
                    }
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }
        }

        static final int TRANSACTION_getBook = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
    }

    public my.itgungnir.ipc.binder.Book getBook() throws android.os.RemoteException;
}
```

最外层的gerBook：一个抽象方法，我们自己在.aidl文件中申明的。

TRANSACTION_getBook：一个整形的Id，表示客户端请求哪个方法，默认一个方法为0，依次累加。

DESCRIPTOR：Binder的唯一标识，一般用当前Binder的包路径表示。

asInterface方法：判断当前进程是服务端进程返回Stub对象，客户端进程返回Stub.Proxy对象

asBinder方法：返回当前的Binder对象。

onTransact(int code, Parcle data，Parcel reply, int flag)：这个方法运行在远程服务端的Binder线程中，当客户端发起请求就是用此方法处理，code代表了客户端要访问的目标方法，data是方法需要的参数，服务端端将返回值写入reply中，如果此方法返回false，代表客户端请求失败。

Proxy内部的getBook方法：运行在客户端内，先创建三个对象_data、_reply、_result，分别代表方法的参数，服务端返回的数据，方法的返回值。调用onTransact方法后客户端当前线程挂起，服务端返回后，当前线程继续执行，经过处理返回result结果。