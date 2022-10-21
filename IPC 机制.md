## IPC 机制

### 一、Linux 中的 IPC 机制

管道（Pipe）、信号（Signal）、信号量（Semophore）、消息队列（Message）、共享内存（Share Memory）和套接字（Socket）等。

![管道的简单模型](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/%E7%AE%A1%E9%81%93%E7%9A%84%E7%AE%80%E5%8D%95%E6%A8%A1%E5%9E%8B%E5%9B%BE.drawio.png)

信号时软件层次上对中断机制的一种模拟，是一种异步通信方式。信号可以在用户空间进程和内核之间直接交互，内核可以用信号来通知用户空间的进程发生了那些系统事件。信号不适用于信息交换，比较适用于中断控制。

使用消息队列会使消息复制两次，因此对于频繁通信或者信息量大的通信不宜使用。

共享内存是在内核专门开辟的内存区域，可由需访问的进程将其映射到自己的私有空间地址，直接读写，提高效率。

### 二、 Android 中的 IPC 机制

Messenger、AIDL、Bundle、文件共享、ContentProvider 和 Binder 等。

Messenger 信使，简单的跨进程数据传递。

AIDL 还可满足跨进程的方法调用。

Activity、Service 、Receiver 都是在 Intent 中通过 Bundle 来进行数据传递的。

文件共享可满足数据同步不高的进程间通信。

ContentProvider 底层实现也为 Binder。

Android 开启多进程引出的问题：

1. 静态成员和单例模式失效；
2. 线程同步失效，因为多进程的内存地址空间不同，锁的不是同一个对象，所以不管是锁对象还是锁全局对象都无法保证线程同步；
3. SharedPreference 的可靠性下降。因为 SharedPrefercences 底层通过读写xml实现，并发读写显示不是安全的操作，甚至会出现数据错乱；
4. Application 会多次创建。

### 三、Native Binder 原理

首先是看下好文——[写给 Android 应用工程师的 Binder 原理剖析](https://zhuanlan.zhihu.com/p/35519585)

![进程通信简单模型](https://pic3.zhimg.com/80/v2-38e2ea1d22660b237e17d2a7f298f3d6_720w.webp)

![内存映射模型](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/%E5%86%85%E5%AD%98%E6%98%A0%E5%B0%84.drawio.png)

![Linux IPC 通信模型](https://pic1.zhimg.com/80/v2-aab2affe42958a659ea8a517ffaff5a0_720w.webp)

![Binder IPC 通信模型](https://pic4.zhimg.com/80/v2-cbd7d2befbed12d4c8896f236df96dbf_720w.webp)

使用 Binder 的原因：

1. 性能方面：只复制一次到内核缓存区；
2. 稳定性方面：基于 C/S 架构；
3. 安全方面：Android 为每个安装好的 App 分配了自己的 UID，通过进程的 UID 来鉴别进程身份。
4. 语言方面：面向对象思想符合 Android 应用层及 Java Framework 层。

![Binder 机制的组成](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/Binder%20%E6%9C%BA%E5%88%B6%E7%9A%84%E7%BB%84%E6%88%90.drawio.png)

### 四、ServiceManager 中的 Binder 机制

![基于 Binder 通信的 C/S 架构](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/%E5%9F%BA%E4%BA%8E%20Binder%20%E9%80%9A%E4%BF%A1%E7%9A%84%20C/S%20%E6%9E%B6%E6%9E%84.drawio.png)

#### 4.1 MediaPlayer 为例

```c
int main(int argc __unused, char **argv __unused)
{
    signal(SIGPIPE, SIG_IGN);
	// 获取 ProcessState 实例，打开 /dev/binder，使用 mmap 为 Binder 驱动分配一个虚拟地址空间来接收数据
    sp<ProcessState> proc(ProcessState::self());
    sp<IServiceManager> sm(defaultServiceManager());// 其他进程通过 IServiceManager 可与当前的 ServiceManager 交互（Binder 通信）
    ALOGI("ServiceManager: %p", sm.get());
    MediaPlayerService::instantiate();
    ResourceManagerService::instantiate();
    registerExtensions();
    ::android::hardware::configureRpcThreadpool(16, false);
    ProcessState::self()->startThreadPool();
    IPCThreadState::self()->joinThreadPool();
    ::android::hardware::joinRpcThreadpool();
}
```

ProcessState 实例作用：

1. 打开 /dev/binder 设备并设定 Binder 最大的支持线程数；
2. 通过 mmap 为 binder 分配一块虚拟地址空间，达到内存映射的目的。

![IServiceManager 家族 UML](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/IServiceManager.drawio.png)

1. BpBinder 与 BBinder 都和通信有关；
2. BpServiceManager 派生自 IServiceManager，他们都和业务有关；
3. BpRefBase 包含了 mRemote，通过 不断派生，BpServiceManager 也同样包含 mRemote，它指向了 BpBinder，通过 BpBinder 来实现通信。

![BpBinder 和 BBinder 一一对应](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/BpBinder%20%E5%92%8C%20BBinder%20%E4%B8%80%E4%B8%80%E5%AF%B9%E5%BA%94.drawio.png)

#### 4.2 MediaPlayerService 的注册过程

![ MediaPlayerService 的注册过程](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/MediaPlayerService%20%E6%B3%A8%E5%86%8C.drawio.png)

```c
// BpBinder transact to get IPCThreadState by self()
IPCThreadState* IPCThreadState::self()
{
    if (gHaveTLS.load(std::memory_order_acquire)) {
restart:
        const pthread_key_t k = gTLS; // Thread local storage，线程本地存储空间，在每个线程中都有 TLS，并在线程间不共享（ThreadLocal?）
        IPCThreadState* st = (IPCThreadState*)pthread_getspecific(k);
        if (st) return st;
        return new IPCThreadState; // 创建 IPCThreadState
    }
...
    goto restart;
}
```

```c
IPCThreadState::IPCThreadState()
      : mProcess(ProcessState::self()),
        mServingStackPointer(nullptr),
        mServingStackPointerGuard(nullptr),
        mWorkSource(kUnsetWorkSource),
        mPropagateWorkSource(false),
        mIsLooper(false),
        mIsFlushing(false),
        mStrictModePolicy(0),
        mLastTransactionBinderFlags(0),
        mCallRestriction(mProcess->mCallRestriction) {
    pthread_setspecific(gTLS, this); // 用来设置 TLS，将 IPCThreadState::self() 获得的 TLS 和自身传入
    clearCaller();
    mIn.setDataCapacity(256); // 用来接收来自 Binder 驱动的数据，默认大小 256 字节
    mOut.setDataCapacity(256); // 用来存储发往 Binder 驱动的数据，默认大小 256 字节
}
```

在 IPCThreadState 的`transact`中：

```c
writeTransactionData(BC_TRANSACTION, flags, handle, code, data, nullptr);// 传输数据，BC_TRANSACTION 为向 Binder 驱动发送的命令协议
```

向 Binder 驱动发送的命令协议，都以 BC_ 开头；而 Binder 驱动返回的命令协议都以 BR_ 开头。

```c
status_t IPCThreadState::writeTransactionData(int32_t cmd, uint32_t binderFlags,
    int32_t handle, uint32_t code, const Parcel& data, status_t* statusBuffer)
{
    binder_transaction_data tr; // 向 Binder 驱动通信的数据结构

    tr.target.ptr = 0; /* Don't pass uninitialized stack data to a remote process */
    tr.target.handle = handle; // 表示目标
    tr.code = code;
    tr.flags = binderFlags;
    tr.cookie = 0;
    tr.sender_pid = 0;
    tr.sender_euid = 0;

    const status_t err = data.errorCheck();
    if (err == NO_ERROR) {
        tr.data_size = data.ipcDataSize();
        tr.data.ptr.buffer = data.ipcData();
        tr.offsets_size = data.ipcObjectsCount()*sizeof(binder_size_t);
        tr.data.ptr.offsets = data.ipcObjects();
    } else if (statusBuffer) {
        tr.flags |= TF_STATUS_CODE;
        *statusBuffer = err;
        tr.data_size = sizeof(status_t);
        tr.data.ptr.buffer = reinterpret_cast<uintptr_t>(statusBuffer);
        tr.offsets_size = 0;
        tr.data.ptr.offsets = 0;
    } else {
        return (mLastError = err);
    }

    mOut.writeInt32(cmd);
    mOut.write(&tr, sizeof(tr)); // 写入 mOut 

    return NO_ERROR;
}
```

```c
status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
{
    uint32_t cmd;
    int32_t err;

    while (1) {
        if ((err=talkWithDriver()) < NO_ERROR) break; // 内部通过 ioctl 函数与 Binder 驱动进行通信
        err = mIn.errorCheck();
        if (err < NO_ERROR) break;
        if (mIn.dataAvail() == 0) continue;

        cmd = (uint32_t)mIn.readInt32();

        IF_LOG_COMMANDS() {
            alog << "Processing waitForResponse Command: "
                << getReturnString(cmd) << endl;
        }

        switch (cmd) {
        ...
        case BR_REPLY:
            {
                binder_transaction_data tr;
                err = mIn.read(&tr, sizeof(tr)); // 读取 Binder 驱动的数据
                ALOG_ASSERT(err == NO_ERROR, "Not enough command data for brREPLY");
                if (err != NO_ERROR) goto finish;

                if (reply) {
                    ...
                } else {
                    ...
                }
            }
            goto finish;

        default:
            err = executeCommand(cmd);
            if (err != NO_ERROR) goto finish;
            break;
        }
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

```c
status_t IPCThreadState::talkWithDriver(bool doReceive)
{
    if (mProcess->mDriverFD < 0) {
        return -EBADF;
    }

    binder_write_read bwr; // 和 Binder 驱动通信的结构体

    // Is the read buffer empty?
    const bool needRead = mIn.dataPosition() >= mIn.dataSize();

    // We don't want to write anything if we are still reading
    // from data left in the input buffer and the caller
    // has requested to read the next data.
    const size_t outAvail = (!doReceive || needRead) ? mOut.dataSize() : 0; // 仍在读不会写入数据进行通信的

    bwr.write_size = outAvail;
    bwr.write_buffer = (uintptr_t)mOut.data(); //

    // This is what we'll read.
    if (doReceive && needRead) {
        bwr.read_size = mIn.dataCapacity();
        bwr.read_buffer = (uintptr_t)mIn.data();
    } else {
        bwr.read_size = 0;
        bwr.read_buffer = 0;
    }
    ...
    // Return immediately if there is nothing to do.
    if ((bwr.write_size == 0) && (bwr.read_size == 0)) return NO_ERROR;

    bwr.write_consumed = 0;
    bwr.read_consumed = 0;
    status_t err;
    do {
        IF_LOG_COMMANDS() {
            alog << "About to read/write, write size = " << mOut.dataSize() << endl;
        }
#if defined(__ANDROID__)
        if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0) // 和 Binder 驱动进行通信
            err = NO_ERROR;
        else
            err = -errno;
#else
        err = INVALID_OPERATION;
#endif
        if (mProcess->mDriverFD < 0) {
            err = -EBADF;
        }
        IF_LOG_COMMANDS() {
            alog << "Finished read/write, write size = " << mOut.dataSize() << endl;
        }
    } while (err == -EINTR);
    ...
    return err;
}
```

MediaPlayerService 注册过程总结（一个调用链）

1. `addService`将数据打包发给 BpBinder 来进行处理；
2. BpBinder 新建一个 IPCThreadState 对象，并将通信任务交给它；
3. IPCThreadState 的`writeTransactionData`用于将命令协议和数据写入 mOut 中；
4. IPCThreadState 的`waitForResponse`两个作用：一是通过 ioctl 函数操作，mOut 与 mIn 与 Binder 驱动进行数据交互；另一个作用就是处理各种命令协议。

进程角度：

MediaPlayerService 的注册过程是以 C/S 架构为基础的，MediaPlayerService 作为客户端，请求添加系统服务；而 ServiceManager 作为服务端，用于完成系统服务的添加。

![MediaPlayerService 的注册过程](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/MediaPlayerService%20%E7%9A%84%E6%B3%A8%E5%86%8C%E8%BF%87%E7%A8%8B.drawio.png)

ServiceManager 的启动过程：

![ServiceManager 的启动过程](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/ServiceManager%20%E7%9A%84%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B.drawio.png)

MediaPlayerService 请求获取服务：

![MediaPlayerService 请求获取服务](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/MediaPlayerService%20%E8%AF%B7%E6%B1%82%E8%8E%B7%E5%8F%96%E6%9C%8D%E5%8A%A1.drawio.png)

Native Binder 的架构：

![Native Binder 的架构](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/Native%20Binder%20%E7%9A%84%E6%9E%B6%E6%9E%84.drawio.png)

### 五、Java Binder 原理

Java Binder 和 Native Binder 进行通信前，需要通过 JNI，JNI 的注册是在 Zygote 进程启动过程中进行的。此过程中，会完成虚拟机 JNI 相关的注册，负责 Java Binder 和 Native Binder 通信的函数为`register_android_os_Binder`——注册 Binder 类、注册 BinderInternal 类、注册 BinderProxy 类。

![Java Binder 关联类](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/Java%20Binder%20%E5%85%B3%E8%81%94%E7%B1%BB.drawio.png)

Java Binder 系统服务注册过程：（将 AMS 注册到 ServiceManager）

![AMS 注册到 ServiceManager](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/AMS%20%E6%B3%A8%E5%86%8C%E5%88%B0%20ServiceManager.drawio.png)

Java Binder 是需要 Native Binder 支持的，最终的目的就是向 Binder 驱动发送数据和接收数据。

Java Binder 注册的服务不是本身，而是 JavaBBinder。

总结：

![Binder 架构简图](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/Binder%20%E6%9E%B6%E6%9E%84%E7%AE%80%E5%9B%BE%E5%AE%8C%E6%95%B4.drawio.png)

