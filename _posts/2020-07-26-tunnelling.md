---
layout: post
title: "隧穿"
description: "Deep Dive in Binder"
cover_url: https://i.loli.net/2020/07/22/Hzspafbdiw4OM2e.png
cover_meta: illustration by [Kayamori](https://www.pixiv.net/artworks/63318516)
tags: 
  - Android
---

聊到 Android 中的进程间通信，果然还是绕不开 Binder。

## 前言

### 何为进程间通信的障碍

在操作系统中，由于虚拟化，不同进程可访问的内存区域被隔离，以确保进程间不会相互干预。因虚拟化内存机制的存在，两个进程的数据传递，即进程间通信（IPC，Inter-Process Communication），必须借助相关机制完成。

显然，进程间通信的相关机制应由操作系统提供，因为用户进程并没有足够的权限去关联两个进程。

### 权限机制

现代化操作系统通常具备两种处理器模式：用户模式（User Mode）与内核模式（Kernel Mode）。操作系统（或指操作系统内核）位于内核模式，可管理计算机的全部资源，而处于用户模式的用户进程若需要执行如访问磁盘等特权操作，就需要通过系统调用（System Call）交由内核模式的操作系统执行。

原因很简单，银行是不会让取款的客户进金库自己数钱的。

Android 底层的 Linux Kernel 自然采用了这一权限机制，内核与相关服务运行的内存空间称为内核空间（Kernel Space），而用户进程所运行的内存空间称为用户空间（User Space）。当用户进程通过系统调用进行特权操作时，称此时进程为内核态（Kernel Mode，需注意根据上下文与内核模式进行区分）与用户态（User Mode）。

### 为什么是 Binder

Linux Kernel 并非没有提供原生的 IPC 机制，只不过它们无法满足 Android IPC 所需的条件：

- 可靠性。

- 安全性，操作系统必须保证两个进程连接与交互的合法性，避免恶意进程干扰其它进程的正常进行。如共享内存，无法保证其它进程的操作合法。

- 高效性，机制必须保证高传输速度。以传统 IPC 机制 Socket 为例，一般常用于网络间传输，因为需要确保复杂网络环境中通信的可靠性，需要额外花费一定的资源，所以无法满足条件。

为了满足自己可靠、安全、高效的要求的 IPC 机制，Android 自行建立了一套 IPC 机制，就是 Binder。

## 通信模型

在 Binder 中，进程双方的通信方式是一个典型的 C/S 模型。发起通信请求的进程为 Client 进程，请求目标进程为 Server 进程。每个采用 Binder 的进程会有一个或多个用于处理接收数据的线程，这些线程在一个线程池中，称为 Binder 线程池。

Client 不是直接向 Server 发送请求的，正如你无法打电话给一个你不知道电话号码的人。因此，Client 需要借助第三方以获取到 Server 的相关信息，便是 Binder 机制中的一名重要角色 —— Service Manager。

当 Server 将允许被其它线程访问的 Binder 对象，与其对应的名称，注册到 Service Manager 后，满足 Server 访问权限的 Client 便可以通过 Binder 名称，从 Service Manager 获取到 Server 端 Binder 对象的代理。

注意，除非 Client 与 Server 为同一个进程（这一情况获取到的是 Binder 对象本身，本文暂不讨论这一特殊情况），否则因为进程隔离，Client 进程拿到的实际是 Binder 对象的代理对象（后简称 Binder 代理），其对代理对象的交互会作用到 Server 进程中的 Binder 对象上。对于一脸天真的 Client 而言，它并没有意识到自己拿到的是一个代理对象。

### 哲学问题

但是问题来了，Client 进程与 Server 进程怎么联系 Service Manager 进程的？其实其它进程与 Service Manager 的通信同样采用 Binder 机制进行。

于是这又变成了一个鸡生蛋蛋生鸡的问题。

Binder 的实现方法是，默认每一个载入 Binder 机制的进程都保留了零号引用，用于获取 Service Manager 的 Binder 代理，无需再通过其它方式获取。

### 秘密通道

当然，不是所有 Server 进程都希望将自己的 Binder 信息注册到 Service Manager，从而对外公开的。Binder 代理自身是可以进行进程间传递的（后文详细说明），因此 Client 与 Server 便可借助已注册到 Service Manager 的 Binder 构建另一条私有连接，这便是匿名 Binder。反过来，注册到 Service Manager，通过名称进行获取的，称为实名 Binder。

类似 HTTPS 协议，匿名 Binder 提高了安全性，在 Client 与 Server 双方不泄漏 Binder 的情况下，其它进程便无法通过穷举等方式获取 Binder 和 Server 通信。

### 核心科技

上文所提及的 Client、Server 与 Service Manager 都是运行在用户空间中的，实际数据传输需要在内核空间进行。为 Binder 机制提供底层支持的是 Binder 驱动（Binder Driver）。

Linux Kernel 提供了可加载内核模块（LKM，Loadable Kernel Module）机制，操作系统可根据需求动态加载内核，防止未使用功能的加载浪费内存空间，也避免了内核重新编译。借助这一机制，Android 将 Binder 的底层支持作为内核模块加载到 Linux Kernel 中。

虽然被称为 Binder 驱动，但它其实和硬件设备并没有什么关系。LKM 一般用于设计硬件设备的驱动程序。与 Binder 驱动的交互采用 `ioctl()` 等系统调用进行，如同通过驱动程序操作硬件设备。换句话而言，Binder 构造了一个虚拟的设备，用于支持进程之间的相互通信。

上文提及 Service Manager，其进程在 init 进程解析脚本启动后，会向 Binder 驱动发送 `BINDER_SET_CONTEXT_MGR` 命令，将自己注册为这一特殊的存在。因为系统仅允许一个 Service Manager 存在，所以在当前 Service Manager 进程没有关闭之前，Binder 驱动不再允许其它进程注册为 Service Manager。

### 更高效的内存拷贝

在传统的 IPC 机制中，因为进程无法相互访问对方内存，所以数据传输需要借由内核进行：通过 `copy_from_user()` 将数据从发送方的用户空间拷贝到内核空间中，再通过 `copy_to_user()` 将内核空间中的数据拷贝到接收方的用户空间。在这一过程中，经历了两次数据拷贝，不仅拷贝自身耗费资源，用户态与内核态的频繁切换也会拖慢性能。

Binder 则通过内存映射的方式，将接收方用户空间的操作内存与内核空间的操作内存映射到同一块物理内存中，避免了 `copy_to_user()` 操作，以提高传输效率。

啥？为什么不将三者的操作内存映射到同一块物理内存中？那不就是共享内存么？

## AIDL

AIDL 通信是基于 Binder 的，从 AIDL 入手分析 Binder 机制是个不错的选择。

### 相关类型

不过在此之前，先来看看一些类型的含义。

**IBinder（Java）**

```
/**
 * Base interface for a remotable object, the core part of a lightweight
 * remote procedure call mechanism designed for high performance when
 * performing in-process and cross-process calls.  This
 * interface describes the abstract protocol for interacting with a
 * remotable object.  Do not implement this interface directly, instead
 * extend from {@link Binder}.
 *
 * ...
 */
```

`IBinder` 被 Binder 与 Binder 代理所实现，实现该接口的类会实现与 Binder 驱动进行数据交互的方法。

**IInterface**

```
/**
 * Base class for Binder interfaces.  When defining a new interface,
 * you must derive it from IInterface.
 */
```

`IInterface` 用于定义 Server 可被操作的合法方法，根据 AIDL 生成的接口会继承此接口。

**Binder（Java）**

```
/**
 * Base class for a remotable object, the core part of a lightweight
 * remote procedure call mechanism defined by {@link IBinder}.
 * This class is an implementation of IBinder that provides
 * standard local implementation of such an object.
 *
 * ...
 */
```

`Binder` 代表了 Server 进程中的 Binder 本地对象，实现了 `IBinder` 接口。

**BinderProxy**

```
/**
 * Java proxy for a native IBinder object.
 * Allocated and constructed by the native javaObjectforIBinder function. Never allocated
 * directly from Java code.
 *
 * @hide
 */
```

`BinderProxy` 代表了 Client 进程中的 Binder 代理对象，实现了 `IBinder` 接口。

### 生成代码解析

这里以 Android Studio 默认创建的 AIDL 接口模版为例，通过阅读生成代码来解析 Binder。

``` java
// ISampleInterface.aidl
package moe.aoramd.binder;

// Declare any non-default types here with import statements

interface ISampleInterface {
    /**
     * Demonstrates some basic types that you can use as parameters
     * and return values in AIDL.
     */
    void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat,
            double aDouble, String aString);

}
```

点击 Build 后会生成如下 Java 代码：

``` java
/*
 * This file is auto-generated.  DO NOT MODIFY.
 */
package moe.aoramd.binder;
// Declare any non-default types here with import statements

public interface ISampleInterface extends android.os.IInterface {
    /**
     * Default implementation for ISampleInterface.
     */
    public static class Default implements moe.aoramd.binder.ISampleInterface {
        /**
         * Demonstrates some basic types that you can use as parameters
         * and return values in AIDL.
         */
        @Override
        public void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat, double aDouble, java.lang.String aString) throws android.os.RemoteException {
        }

        @Override
        public android.os.IBinder asBinder() {
            return null;
        }
    }

    /**
     * Local-side IPC implementation stub class.
     */
    public static abstract class Stub extends android.os.Binder implements moe.aoramd.binder.ISampleInterface {
        private static final java.lang.String DESCRIPTOR = "moe.aoramd.binder.ISampleInterface";

        /**
         * Construct the stub at attach it to the interface.
         */
        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }

        /**
         * Cast an IBinder object into an moe.aoramd.binder.ISampleInterface interface,
         * generating a proxy if needed.
         */
        public static moe.aoramd.binder.ISampleInterface asInterface(android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (((iin != null) && (iin instanceof moe.aoramd.binder.ISampleInterface))) {
                return ((moe.aoramd.binder.ISampleInterface) iin);
            }
            return new moe.aoramd.binder.ISampleInterface.Stub.Proxy(obj);
        }

        @Override
        public android.os.IBinder asBinder() {
            return this;
        }

        @Override
        public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
            java.lang.String descriptor = DESCRIPTOR;
            switch (code) {
                case INTERFACE_TRANSACTION: {
                    reply.writeString(descriptor);
                    return true;
                }
                case TRANSACTION_basicTypes: {
                    data.enforceInterface(descriptor);
                    int _arg0;
                    _arg0 = data.readInt();
                    long _arg1;
                    _arg1 = data.readLong();
                    boolean _arg2;
                    _arg2 = (0 != data.readInt());
                    float _arg3;
                    _arg3 = data.readFloat();
                    double _arg4;
                    _arg4 = data.readDouble();
                    java.lang.String _arg5;
                    _arg5 = data.readString();
                    this.basicTypes(_arg0, _arg1, _arg2, _arg3, _arg4, _arg5);
                    reply.writeNoException();
                    return true;
                }
                default: {
                    return super.onTransact(code, data, reply, flags);
                }
            }
        }

        private static class Proxy implements moe.aoramd.binder.ISampleInterface {
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

            /**
             * Demonstrates some basic types that you can use as parameters
             * and return values in AIDL.
             */
            @Override
            public void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat, double aDouble, java.lang.String aString) throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    _data.writeInt(anInt);
                    _data.writeLong(aLong);
                    _data.writeInt(((aBoolean) ? (1) : (0)));
                    _data.writeFloat(aFloat);
                    _data.writeDouble(aDouble);
                    _data.writeString(aString);
                    boolean _status = mRemote.transact(Stub.TRANSACTION_basicTypes, _data, _reply, 0);
                    if (!_status && getDefaultImpl() != null) {
                        getDefaultImpl().basicTypes(anInt, aLong, aBoolean, aFloat, aDouble, aString);
                        return;
                    }
                    _reply.readException();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }

            public static moe.aoramd.binder.ISampleInterface sDefaultImpl;
        }

        static final int TRANSACTION_basicTypes = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);

        public static boolean setDefaultImpl(moe.aoramd.binder.ISampleInterface impl) {
            if (Stub.Proxy.sDefaultImpl == null && impl != null) {
                Stub.Proxy.sDefaultImpl = impl;
                return true;
            }
            return false;
        }

        public static moe.aoramd.binder.ISampleInterface getDefaultImpl() {
            return Stub.Proxy.sDefaultImpl;
        }
    }

    /**
     * Demonstrates some basic types that you can use as parameters
     * and return values in AIDL.
     */
    public void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat, double aDouble, java.lang.String aString) throws android.os.RemoteException;
}
```

暂时先把它变成简化版，看一看生成代码的结构。

``` java
public interface ISampleInterface extends android.os.IInterface {
    
    public static class Default implements moe.aoramd.binder.ISampleInterface { /* ... */ }

    public static abstract class Stub extends android.os.Binder implements moe.aoramd.binder.ISampleInterface {

        /* ... */

        private static class Proxy implements moe.aoramd.binder.ISampleInterface { /* ... */ }

        /* ... */
    }

    public void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat, double aDouble, java.lang.String aString) throws android.os.RemoteException;
}
```

可见编译器帮我们生成了三个实现了 `ISampleInterface` 的静态内部类。

- `Stub`，继承于 `Binder`，表明这是 Binder 本地对象。在使用 AIDL 时我们会在 Server 代码继承该类。其中 `onTransact()` 方法会把从 Client 传来的数据作解析（类似于通信中的解调），然后调用指定的方法。

- `Stub.Proxy`，Client 持有的 Binder 代理所属类，在 `Stub.asInterface()` 中被构造。当接口方法被调用时会对相关数据进行处理（类似通信中的调制），之后委托给构造方法中传入的 Binder 代理。注意！`Stub.Proxy` 并不是 Binder 代理的实现，而是 Binder 代理的包装，它将操作委托给了 Binder 代理。

- `Default`，顾名思义，这是接口默认实现的默认实现（是不是很绕口）。`Stub` 提供了静态的 getter/setter 用于设置接口的全局默认实现，当 Server 无法成功处理调用时会使用默认实现。

讲完这三个类的功能之后，回头看 AIDL 生成的代码，会发现该讲的已经讲完一大半了：Client 端代码调用 `Proxy.basicType()` 时，参数被设置到 `Parcel` 对象后委托给 `BinderProxy.transact()` 传递到 Server，Server 在 `onTransact()` 获取到数据后，通过对数据进行解析，得知 Client 调用的是哪一个方法、以及方法的参数，随后执行方法。

因此，若要进一步了解 Binder 是如何传递数据的，我们的重心自然应放在 `transact()` 与 `onTransact()` 上。

## Binder 线程池注册

虽然如此，但现在我们先把 `transact()` 和 `onTransact()` 放一放，回头从另一个方向走一走看看。

前文提到，每个采用 Binder 的进程会有一个或多个用于处理接收数据的线程，位于 Binder 线程池。采用 Binder 机制的进程最典型的就是应用程序进程了。那应用程序进程的 Binder 线程池是在什么时候启动的呢？

应用程序进程由 Zygote 进程 fork 所得，若从 `ZygoteInit.zygoteInit()` 开始跟踪 Binder 线程池启动的话，忽略掉过程中的调用链，最终会定位到如下代码：

> 调用链：`ZygoteInit.zygoteInit()` --> `ZygoteInit.nativeZygoteInit()` ··JNI··> `com_android_internal_os_ZygoteInit_nativeZygoteInit()` --> `AppRuntime.onZygoteInit()`

``` c++
virtual void onZygoteInit()
{
    sp<ProcessState> proc = ProcessState::self();
    ALOGV("App process: starting thread pool.\n");
    proc->startThreadPool();
}
```

`ProcessState` 是 Binder 机制核心之一（敲黑板，重点，要记），它是 Binder 通信的基础，负责与 Binder 驱动的交互与 Binder 线程池的管理。它实现了单例模式，通过 `self()` 函数获取实例。在这里，函数获取了 `ProcessState` 实例后调用其 `startThreadPool()` 函数，以启动进程的 Binder 线程池。

``` c++
void ProcessState::startThreadPool()
{
    AutoMutex _l(mLock);
    if (!mThreadPoolStarted) {
        mThreadPoolStarted = true;
        spawnPooledThread(true);
    }
}

void ProcessState::spawnPooledThread(bool isMain)
{
    if (mThreadPoolStarted) {
        String8 name = makeBinderThreadName();
        ALOGV("Spawning new pooled thread, name=%s\n", name.string());
        sp<Thread> t = new PoolThread(isMain);
        t->run(name.string());
    }
}
```

`mThreadPoolStarted` 用于标识线程池是否已经启动过，以确保 Binder 线程池仅初始化一次。`spawnPooledThread()` 函数启动了一个 Binder 线程，类型为 `PoolThread`，函数参数表示这是 Binder 线程池中的第一线程。

``` c++
class PoolThread : public Thread
{
public:
    explicit PoolThread(bool isMain)
        : mIsMain(isMain)
    {
    }

protected:
    virtual bool threadLoop()
    {
        IPCThreadState::self()->joinThreadPool(mIsMain);
        return false;
    }

    const bool mIsMain;
};
```

`PoolThread` 继承自 `Thread`，在运行时通过 `IPCThreadState` 将自身线程注册到 Binder 驱动中。`IPCThreadState` 同样是 Binder 机制的核心之一，它用于管理与 Binder 通信相关线程的状态，每个 Binder 线程都会通过此将自己注册到 Binder 驱动。

`self()` 函数是一个工厂函数，用于获取 `IPCThreadState` 实例。`self()` 根据 `pthread_getspecific()` 管理每个参与 Binder 通信线程的实例，类似于 Handler 中所用的 `ThreadLocalMap`，每个参与 Binder 通信的线程其 `IPCThreadState` 对象都是相互独立的，保证了后续操作的线程安全。

## 与 Binder 驱动进行通信

之所以在聊 `transact()` 和 `onTransact()` 之前扯 Binder 线程池的初始化，是因为两者都与 `ProcessState` 和 `IPCThreadState` 密不可分。

### 相关类型

在阅读源码前，与前文一样，先来看看一些类型的含义。

**IBinder（C++）**

```
/**
 * Base class and low-level protocol for a remotable object.
 * You can derive from this class to create an object for which other
 * processes can hold references to it.  Communication between processes
 * (method calls, property get and set) is down through a low-level
 * protocol implemented on top of the transact() API.
 */
```

与 Java 层的 `IBinder` 类似，是跨进程数据交互类型的基类。Native的 Binder 与 Binder 代理均继承此类。

**BBinder**

即 Binder Native，代表 Binder 本地对象，是 Binder 代理交互的目的端，等同 `Binder` 在 Native 层功能的角色。如果根据 Framework 中的代码风格，更合适的名字为 `BnBinder`。

**BpBinder**

即 Binder Proxy，代表 Binder 代理对象，等同 `BinderProxy` 在 Native 层功能的角色。

### Transaction

如果跟踪 `transact()`，会发现之后调用了 Native 层 `BpBinder` 对象的 `transact()` 函数。

> 调用链：`BinderProxy.transact()` --> `BinderProxy.transactNative` ··JNI··> `android_os_BinderProxy_transact()` --> `BpBinder.transact()`

``` c++
// NOLINTNEXTLINE(google-default-arguments)
status_t BpBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    // Once a binder has died, it will never come back to life.
    if (mAlive) {
        
        /* ... */

        status_t status = IPCThreadState::self()->transact(
            mHandle, code, data, reply, flags);
        if (status == DEAD_OBJECT) mAlive = 0;

        return status;
    }

    return DEAD_OBJECT;
}
```

`IPCThreadState`，熟悉的面孔。在调用链中的各个函数对参数进行预处理之后，数据最终将通过 `IPCThreadState.transact()` 解析发送给 Binder 驱动。其实现如下：

``` c++
status_t IPCThreadState::transact(int32_t handle,
                                  uint32_t code, const Parcel& data,
                                  Parcel* reply, uint32_t flags)
{
    status_t err;

    /* ... */

    err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, nullptr);

    if (err != NO_ERROR) {
        if (reply) reply->setError(err);
        return (mLastError = err);
    }

    if ( /* ... */ ) {

        /* ... */

        if (reply) {
            err = waitForResponse(reply);
        } else {
            Parcel fakeReply;
            err = waitForResponse(&fakeReply);
        }
        
        /* ... */

    } else {
        err = waitForResponse(nullptr, nullptr);
    }

    return err;
}
```

核心调用为 `writeTransactionData()` 与 `waitForResponse()`，两者实现如下：

``` c++
status_t IPCThreadState::writeTransactionData(int32_t cmd, uint32_t binderFlags,
    int32_t handle, uint32_t code, const Parcel& data, status_t* statusBuffer)
{
    binder_transaction_data tr;

    /* ... */

    mOut.writeInt32(cmd);
    mOut.write(&tr, sizeof(tr));

    return NO_ERROR;
}

status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
{
    uint32_t cmd;
    int32_t err;

    while (1) {
        if ((err=talkWithDriver()) < NO_ERROR) break;
        err = mIn.errorCheck();
        if (err < NO_ERROR) break;
        if (mIn.dataAvail() == 0) continue;

        cmd = (uint32_t)mIn.readInt32();

        /* ... */

        switch (cmd) { /* ... */ }
    }

finish:
    if (err != NO_ERROR) {
        if (acquireResult) *acquireResult = err;
        if (reply) reply->setError(err);
        mLastError = err;
    }

    return err;
}
```

`mIn` 与 `mOut` 均为 `IPCThreadState` 中 `Parcel` 类型的成员变量，作为与 Binder 驱动通信的数据接收、发送窗口。在 `waitForResponse()` 中调用了 `talkWithDriver()`，从函数名可以看出它的功能了。

在这里，Binder 驱动与 `mIn`、`mOut` 的数据交互开始了。

绕个圈，回过头看 Binder 线程的注册过程，同样使用到了 `talkWithDriver()` 与 Binder 驱动进行通信。

``` c++
void IPCThreadState::joinThreadPool(bool isMain)
{
    /* ... */

    talkWithDriver(false);
}
```

### 数据交互

`IPCThreadState.talkWithDrive()` 的函数签名位于 *IPCThreadState.h*，参数用于确定是否接收信息，其默认值为 true。

``` c++
class IPCThreadState
{
/* ... */

private:
            /* ... */

            status_t            talkWithDriver(bool doReceive=true);

            /* ... */
}
```

函数实现如下：

``` c++
status_t IPCThreadState::talkWithDriver(bool doReceive)
{
    if (mProcess->mDriverFD < 0) {
        return -EBADF;
    }

    binder_write_read bwr;

    // Is the read buffer empty?
    const bool needRead = mIn.dataPosition() >= mIn.dataSize();

    // We don't want to write anything if we are still reading
    // from data left in the input buffer and the caller
    // has requested to read the next data.
    const size_t outAvail = (!doReceive || needRead) ? mOut.dataSize() : 0;

    bwr.write_size = outAvail;
    bwr.write_buffer = (uintptr_t)mOut.data();

    // This is what we'll read.
    if (doReceive && needRead) {
        bwr.read_size = mIn.dataCapacity();
        bwr.read_buffer = (uintptr_t)mIn.data();
    } else {
        bwr.read_size = 0;
        bwr.read_buffer = 0;
    }

    /* logcat ... */

    // Return immediately if there is nothing to do.
    if ((bwr.write_size == 0) && (bwr.read_size == 0)) return NO_ERROR;

    bwr.write_consumed = 0;
    bwr.read_consumed = 0;
    status_t err;
    do {
        /* logcat ... */
#if defined(__ANDROID__)
        if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0)
            err = NO_ERROR;
        else
            err = -errno;
#else
        err = INVALID_OPERATION;
#endif
        if (mProcess->mDriverFD < 0) {
            err = -EBADF;
        }
        /* logcat ... */
    } while (err == -EINTR);

    /* logcat ... */

    if (err >= NO_ERROR) {
        if (bwr.write_consumed > 0) {
            if (bwr.write_consumed < mOut.dataSize())
                /* logcat ... */
            else {
                mOut.setDataSize(0);
                processPostWriteDerefs();
            }
        }
        if (bwr.read_consumed > 0) {
            mIn.setDataSize(bwr.read_consumed);
            mIn.setDataPosition(0);
        }
        /* logcat ... */
        return NO_ERROR;
    }

    return err;
}
```

有意思的是，函数名为 `talkWithDriver()`，而不是 `talkToDriver()`，其实已经表明在函数中与 Binder 驱动的数据交互是一个双向的过程。

`mOut` 数据窗口中的数据、`mIn` 和 `mOut` 窗口的内存地址，在经过 `binder_write_read` 结构体封装后，通过 `ioctl()` 系统调用发送到 Binder 驱动。`mProcess` 即进程中的 `ProcessState` 对象，在 `IPCThreadState` 实例构造时便已初始化。`ioctl()` 通过 `ProcessState` 初始化时指定的 Binder 驱动文件描述符，将数据发送到 Binder 域 `/dev/binder` 的 Binder 驱动中。

> Android 并不只有 `/dev/binder` 一个 Binder 驱动。
>
> 在 Android 8 采用 Project Treble 后，将 Android Framework 与供应商实现分离。Android 8 即更高版本的系统总共拥有三个 Binder 域，提供给硬件供应商实现 IPC。其中：
>
> - `/dev/binder`，用于 Android Framework 与应用程序进程的 IPC，使用 AIDL。
> - `/dev/hwbinder`，用于 Android Framework 与供应商进程或供应商进程之间的 IPC，使用 HIDL。
> - `/dev/vndbinder`，用于供应商进程之间的 IPC，使用 AIDL。
>
> 详情可见 [Android 开源项目 - 使用 Binder IPC](https://source.android.com/devices/architecture/hidl/binder-ipc)

### 小结

虽然仅仅是对 Client 发送数据流程的一次走马观花，但还是可以得出一些有价值的信息，在后面的分析过程中少走一些弯路：

- `IPCThreadState` 是线程局部对象，每个参与 Binder 通信的进程通过 `IPCThreadState.self()` 获取到的实例是相互独立的。

- 从 Client 调用 Binder 代理方法到获取返回值，是一个阻塞的过程。

## 另一个世界

看完了 Client 发送数据的流程，接下来看看 Server 接收数据的流程。

### Register

要聊 Server，就应该从 Server 到 Service Manager 上户口讲起。

Server 端的代码会通过 `defaultServiceManager()` 函数获取 Service Manager 的 Binder 代理（此时的 Server 相对于 Service Manager 是 Client）。

``` c++
sp<IServiceManager> defaultServiceManager()
{
    std::call_once(gSmOnce, []() {
        sp<AidlServiceManager> sm = nullptr;
        while (sm == nullptr) {
            sm = interface_cast<AidlServiceManager>(ProcessState::self()->getContextObject(nullptr));
            if (sm == nullptr) {
                ALOGE("Waiting 1s on context object on %s.", ProcessState::self()->getDriverName().c_str());
                sleep(1);
            }
        }

        gDefaultServiceManager = new ServiceManagerShim(sm);
    });

    return gDefaultServiceManager;
}
```

`ServiceManagerShim()` 构造函数传入的是一个 `AidlServiceManager` 对象，即 `IServiceManager` 对象。而在这里调用 `ProcessState.getContextObject()` 所得到的，正是零号引用 —— Service Manager 的 Binder 代理。

``` c++
sp<IBinder> ProcessState::getContextObject(const sp<IBinder>& /*caller*/)
{
    sp<IBinder> context = getStrongProxyForHandle(0);

    /* ... */

    return context;
}

sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle)
{
    sp<IBinder> result;

    AutoMutex _l(mLock);

    handle_entry* e = lookupHandleLocked(handle);

    if (e != nullptr) {
        // We need to create a new BpBinder if there isn't currently one, OR we
        // are unable to acquire a weak reference on this current one.  The
        // attemptIncWeak() is safe because we know the BpBinder destructor will always
        // call expungeHandle(), which acquires the same lock we are holding now.
        // We need to do this because there is a race condition between someone
        // releasing a reference on this BpBinder, and a new reference on its handle
        // arriving from the driver.
        IBinder* b = e->binder;
        if (b == nullptr || !e->refs->attemptIncWeak(this)) {
            if (handle == 0) {
                // Special case for context manager...
                // The context manager is the only object for which we create
                // a BpBinder proxy without already holding a reference.
                // Perform a dummy transaction to ensure the context manager
                // is registered before we create the first local reference
                // to it (which will occur when creating the BpBinder).
                // If a local reference is created for the BpBinder when the
                // context manager is not present, the driver will fail to
                // provide a reference to the context manager, but the
                // driver API does not return status.
                //
                // Note that this is not race-free if the context manager
                // dies while this code runs.
                //
                // TODO: add a driver API to wait for context manager, or
                // stop special casing handle 0 for context manager and add
                // a driver API to get a handle to the context manager with
                // proper reference counting.

                Parcel data;
                status_t status = IPCThreadState::self()->transact(
                        0, IBinder::PING_TRANSACTION, data, nullptr, 0);
                if (status == DEAD_OBJECT)
                   return nullptr;
            }

            b = BpBinder::create(handle);
            e->binder = b;
            if (b) e->refs = b->getWeakRefs();
            result = b;
        } else {
            // This little bit of nastyness is to allow us to add a primary
            // reference to the remote proxy when this team doesn't have one
            // but another team is sending the handle to us.
            result.force_set(b);
            e->refs->decWeak(this);
        }
    }

    return result;
}
```

零号引用的 `BpBinder` 对象传入 `interface_cast()` 模版函数，会最终通过 `IMPLEMENT_META_INTERFACE()` 宏，生成 `BpServiceManager` 对象。该对象被传入 `ServiceManagerShim` 的构造函数中，成为其成员变量 `mTheRealServiceManager`。

``` c++
/**
 * If this is a local object and the descriptor matches, this will return the
 * actual local object which is implementing the interface. Otherwise, this will
 * return a proxy to the interface without checking the interface descriptor.
 * This means that subsequent calls may fail with BAD_TYPE.
 */
template<typename INTERFACE>
inline sp<INTERFACE> interface_cast(const sp<IBinder>& obj)
{
    return INTERFACE::asInterface(obj);
}

#ifndef DO_NOT_CHECK_MANUAL_BINDER_INTERFACES

#define IMPLEMENT_META_INTERFACE(INTERFACE, NAME)                       \
    static_assert(internal::allowedManualInterface(NAME),               \
                  "b/64223827: Manually written binder interfaces are " \
                  "considered error prone and frequently have bugs. "   \
                  "The preferred way to add interfaces is to define "   \
                  "an .aidl file to auto-generate the interface. If "   \
                  "an interface must be manually written, add its "     \
                  "name to the whitelist.");                            \
    DO_NOT_DIRECTLY_USE_ME_IMPLEMENT_META_INTERFACE(INTERFACE, NAME)    \

#else

#define IMPLEMENT_META_INTERFACE(INTERFACE, NAME)                       \
    DO_NOT_DIRECTLY_USE_ME_IMPLEMENT_META_INTERFACE(INTERFACE, NAME)    \

#endif

#define DO_NOT_DIRECTLY_USE_ME_IMPLEMENT_META_INTERFACE(INTERFACE, NAME)\
    /* ... */                                                           \
    ::android::sp<I##INTERFACE> I##INTERFACE::asInterface(              \
            const ::android::sp<::android::IBinder>& obj)               \
    {                                                                   \
        ::android::sp<I##INTERFACE> intr;                               \
        if (obj != nullptr) {                                           \
            intr = static_cast<I##INTERFACE*>(                          \
                obj->queryLocalInterface(                               \
                        I##INTERFACE::descriptor).get());               \
            if (intr == nullptr) {                                      \
                intr = new Bp##INTERFACE(obj);                          \
            }                                                           \
        }                                                               \
        return intr;                                                    \
    }                                                                   \
    /* ... */
```

调用 `ServiceManagerShim.addService()` 函数以将 Binder 注册到 Service Manager 中。注册的过程同样也是一次 Binder 传输，最后被注册的 Binder 到达 Service Manager 进程。

``` c++
Status ServiceManager::addService(const std::string& name, const sp<IBinder>& binder, bool allowIsolated, int32_t dumpPriority) {
    
    /* ... */

    auto entry = mNameToService.emplace(name, Service {
        .binder = binder,
        .allowIsolated = allowIsolated,
        .dumpPriority = dumpPriority,
        .debugPid = ctx.debugPid,
    });

    /* ... */
}
```

`mNameToService` 类型为 `std::map<std::string, Service>`。至此，Server 的 Binder 已被注册到 Service Manager 中。

> 看过 Android 10 及之前相关源码的同学应该会对 `BpServiceManager` 这个类比较熟悉，它是 Service Manager 在其它 Binder 线程的代理。不过这一部分代码现在已经被删除了。
>
> ```
> 80e1e6d   smoreland@google.com   2019-07-09 09:54 +08:00
>
> servicemanager: use libbinder
>
> Bug: 135768100
> Test: boot
> Test: servicemanager_test
>
> Change-Id: I9d657b6c0d0be0f763b6d54e0e6c6bc1c1e3fc7a
> (cherry picked from commit 3e092daa14c63831d76d3ad6e56b2919a0523536)
> ```
>
> 在此之后，`BpServiceManager` 不再通过手动实现，而是采用 AIDL（文件为 `IServiceManager.aidl`），生成 `IServiceManager`、`BnServiceManager`、`BpServiceManager` 的头文件及具体实现。
>
> 关于通过 AIDL 生成 C++ 代码，详见 [Generating C++ Binder Interfaces with aidl-cpp](https://android.googlesource.com/platform/system/tools/aidl/+/brillo-m10-dev/docs/aidl-cpp.md)

### Listen

Binder 线程用于在 Server 中接收处理从 Binder 驱动发送来的数据。一个线程通过前文提及的函数 `IPCThreadState.joinThreadPool()` 将自己注册到 Binder 线程池，等待接收数据。

不过关于 `joinThreadPool()` 的实现，这次的重点不再是 `talkWithDriver()`，而是循环与 `getAndExecuteCommand()` 函数。

``` c++
void IPCThreadState::joinThreadPool(bool isMain)
{
    /* ... */

    mOut.writeInt32(isMain ? BC_ENTER_LOOPER : BC_REGISTER_LOOPER);

    status_t result;
    do {
        processPendingDerefs();
        // now get the next command to be processed, waiting if necessary
        result = getAndExecuteCommand();

        /* ... */

        // Let this thread exit the thread pool if it is no longer
        // needed and it is not the main process thread.
        if(result == TIMED_OUT && !isMain) {
            break;
        }
    } while (result != -ECONNREFUSED && result != -EBADF);

    /* ... */

    mOut.writeInt32(BC_EXIT_LOOPER);
    talkWithDriver(false);
}
```

`getAndExecuteCommand()` 用于获取从 Binder 驱动传来的数据，并执行数据中所包含的命令。在数据发送的小结中提过一点，这里的情况也与其相似：**调用过程中是阻塞的**，因此在未处理完命令之前，Binder 线程无法执行下一道命令。

``` c++
status_t IPCThreadState::getAndExecuteCommand()
{
    status_t result;
    int32_t cmd;

    result = talkWithDriver();
    if (result >= NO_ERROR) {
        size_t IN = mIn.dataAvail();
        if (IN < sizeof(int32_t)) return result;
        cmd = mIn.readInt32();
        
        /* ... */

        result = executeCommand(cmd);

        /* ... */
    }

    return result;
}
```

调用 `talkWithDriver()` 从 `mIn` 窗口解析出需要执行的命令后，执行 `executeCommand()`。

``` c++
status_t IPCThreadState::executeCommand(int32_t cmd)
{
    /* ... */

    switch ((uint32_t)cmd) {

    /* ... */

    case BR_TRANSACTION:
        {
            /* ... */

            if (tr.target.ptr) {
                // We only have a weak reference on the target object, so we must first try to
                // safely acquire a strong reference before doing anything else with it.
                if (reinterpret_cast<RefBase::weakref_type*>(
                        tr.target.ptr)->attemptIncStrong(this)) {
                    error = reinterpret_cast<BBinder*>(tr.cookie)->transact(tr.code, buffer,
                            &reply, tr.flags);
                    reinterpret_cast<BBinder*>(tr.cookie)->decStrong(this);
                } else {
                    error = UNKNOWN_TRANSACTION;
                }

            } else {
                error = the_context_object->transact(tr.code, buffer, &reply, tr.flags);
            }

            /* ... */
        }
        break;

    /* ... */

    case BR_SPAWN_LOOPER:
        mProcess->spawnPooledThread(false);
        break;

    /* ... */
}
```

其中 `the_context_object` 为 `BBinder` 对象，也就是 Server 的 Binder 本体。`BBinder.transact()` 会再调用 `BBinder.onTransact()` 函数，实现 Server 进程 Binder 的调用。

这里稍微跑题，额外提一个 switch 分支：`BR_SPAWN_LOOPER`。可以看出这里与应用进程启动 Binder 线程池时类似，但在这里传入的参数为 false，表明不是第一线程。这代表着，Binder 驱动可以根据当前进程处理 Binder 数据的繁忙程度来决定是否开启更多的 Binder 线程。

如果是在 C++ 中采用 Binder，Server 接收数据的流程便已经到头了。而对于使用 Java 的 Binder，`JavaBBinder` 是 `BBinder` 的一个子类，`JavaBBinder.onTransact()` 将会通过 JNI 调用 Java 层 `Binder.onTransact()`。

``` c++
class JavaBBinder : public BBinder
{
/* ... */

protected:
    /* ... */

    status_t onTransact(
        uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags = 0) override
    {
        JNIEnv* env = javavm_to_jnienv(mVM);

        /* ... */

        IPCThreadState* thread_state = IPCThreadState::self();
        
        /* ... */

        jboolean res = env->CallBooleanMethod(mObject, gBinderOffsets.mExecTransact,
            code, reinterpret_cast<jlong>(&data), reinterpret_cast<jlong>(reply), flags);

        /* ... */
    }

    /* ... */

private:
    JavaVM* const   mVM;
    jobject const   mObject;  // GlobalRef to Java Binder

    /* ... */
};
```

## 启动 Service Manager

> `80e1e6d` 提交同样变更了 Service Manager 的实现方式，现在的实现与 Hardware Service Manager 的代码有些相似。

Service Manager 是一个系统服务进程。与 Zygote 进程相似，我们可以在 `init.rc` 中找到其启动的相关信息。

``` shell
on init

    # ... #

    # Start essential services.
    start servicemanager
    start hwservicemanager
    start vndservicemanager
```

在 `init` 触发器触发后，`servicemanager` 服务将被启动。`servicemanager` 服务的定义位于 `servicemanager.rc` 中。

``` shell
service servicemanager /system/bin/servicemanager
    class core animation
    user system
    group system readproc
    critical
    onrestart restart apexd
    onrestart restart audioserver
    onrestart restart gatekeeperd
    onrestart class_restart main
    onrestart class_restart hal
    onrestart class_restart early_hal
    writepid /dev/cpuset/system-background/tasks
    shutdown critical
```

服务运行程序为 `/system/bin/servicemanager`。这里的一个细节是修饰符 `critical`，文档中的说明如下，表明 `servicemanager` 是一个系统关键进程，如果出现问题，Android 系统是无法正常运行的：

> This is a device-critical service. If it exits more than four times in four minutes or before boot completes, the device will reboot into bootloader.

与 `systemservice` 相关的代码位于 `/frameworks/native/cmds/servicemanager`。程序的入口在 `main.cpp`。

``` c++
int main(int argc, char** argv) {
    if (argc > 2) {
        LOG(FATAL) << "usage: " << argv[0] << " [binder driver]";
    }

    const char* driver = argc == 2 ? argv[1] : "/dev/binder";

    sp<ProcessState> ps = ProcessState::initWithDriver(driver);
    ps->setThreadPoolMaxThreadCount(0);
    ps->setCallRestriction(ProcessState::CallRestriction::FATAL_IF_NOT_ONEWAY);

    sp<ServiceManager> manager = new ServiceManager(std::make_unique<Access>());
    if (!manager->addService("manager", manager, false /*allowIsolated*/, IServiceManager::DUMP_FLAG_PRIORITY_DEFAULT).isOk()) {
        LOG(ERROR) << "Could not self register servicemanager";
    }

    IPCThreadState::self()->setTheContextObject(manager);
    ps->becomeContextManager(nullptr, nullptr);

    sp<Looper> looper = Looper::prepare(false /*allowNonCallbacks*/);

    BinderCallback::setupTo(looper);
    ClientCallbackCallback::setupTo(looper, manager);

    while(true) {
        looper->pollAll(-1);
    }

    // should not be reached
    return EXIT_FAILURE;
}
```

代码逻辑可以拆分成两部分进行解析。

### Register

Service Manager 需要将自己注册到 Binder 驱动中。第一步操作是设定 `ProcessState` 的 Binder 域。

``` c++
const char* driver = argc == 2 ? argv[1] : "/dev/binder";

sp<ProcessState> ps = ProcessState::initWithDriver(driver);
```

然后创建类型为 `ServiceManager` 的 Binder 本地对象，并将自己注册到自己的 Binder 列表中，提供给其它进程获取。

``` c++
sp<ServiceManager> manager = new ServiceManager(std::make_unique<Access>());
if (!manager->addService("manager", manager, false /*allowIsolated*/, IServiceManager::DUMP_FLAG_PRIORITY_DEFAULT).isOk()) {
    LOG(ERROR) << "Could not self register servicemanager";
}
```

在当前线程的 `IPCThreadState` 对象中注册 `the_context_object`，以指定 Binder 线程对应的 Binder 对象。

``` c++
IPCThreadState::self()->setTheContextObject(manager);
```

最后，向 Binder 驱动发送 `BINDER_SET_CONTEXT_MGR`，将自己注册为唯一的 Service Manager，完成注册过程。

``` c++
ps->becomeContextManager(nullptr, nullptr);
```

`ProcessState.becomeContextManager()` 实现如下：

``` c++
bool ProcessState::becomeContextManager(context_check_func checkFunc, void* userData)
{
    /* ... */

    int result = ioctl(mDriverFD, BINDER_SET_CONTEXT_MGR_EXT, &obj);

    // fallback to original method
    if (result != 0) {

        /* ... */

        result = ioctl(mDriverFD, BINDER_SET_CONTEXT_MGR, &dummy);
    }

    /* ... */

    return result == 0;
}
```

### Listen

以前 Service Manager 通过无限循环处理消息，现在的实现采用了 Looper。

在这里，进程通过 `Looper` 处理事件。`BinderCallback` 的 `setupTo()` 调用了 `Looper.addFd()` 向 Looper 注册文件描述符，监听 `Looper.POLL_CALLBACK` 信息，其中 `BinderCallback` 监听的文件描述符为 `-1`。

``` c++
BinderCallback::setupTo(looper);
```

在循环中调用 `Looper.pollAll()`，执行文件描述符相关的 Callback。

``` c++
while(true) {
    looper->pollAll(-1);
}
```

`BinderCallback` 是 `Looper` 接收事件时的回调。实际实现等待 Binder 驱动数据的还是那个函数：`IPCThreadState.getAndExecuteCommand()`。

> 调用链：`BinderCallback.handleEvent()` --> `IPCThreadState.handlePolledCommands()` --> `IPCThreadState.getAndExecuteCommand()`

``` c++
status_t IPCThreadState::handlePolledCommands()
{
    status_t result;

    do {
        result = getAndExecuteCommand();
    } while (mIn.dataPosition() < mIn.dataSize());

    processPendingDerefs();
    flushCommands();
    return result;
}
```

## 参考链接

- [weishu - Binder 学习指南](http://weishu.me/2016/01/12/binder-index-for-newer/)

- [universus - Binder 设计与实现](https://blog.csdn.net/universus/article/details/6211589)

- [芦半山 - Binder 内存拷贝的本质和变迁](https://juejin.im/post/5e85aa5e6fb9a03c341d9737)