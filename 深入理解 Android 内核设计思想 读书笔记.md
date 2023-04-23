## 深入理解 Android 内核设计思想 读书笔记

### ch4.3 进程间通信的经典实现

共享内存：创建内存共享区、映射内存共享区、访问内存共享区、进程间通信、撤销内存映射区、删除内存共享区

管道

UNIX Domain Socket：服务器监听 IPC 请求；客户端发起 IPC 申请；双方成功建立起 IPC 连接；客户端向服务端发送数据，证明 IPC 通信是有效的。

RPC（Remote Procedure Calls）：客户端进程调用 Stub 接口；Stub 根据操作系统的要求进行打包，并执行相应的系统调用；由内核来完成与服务器端的具体交互，它负责将客户端的数据包发给服务器端的内核；服务器端 Stub 解包并调用与数据包匹配的进程；进程执行操作；服务器以上述步骤的逆向过程将结果返回给客户端。

### ch4.4 同步机制的经典实现

信号量（Semaphore）与 PV 操作

Mutex

管程（Monitor）：是可以被多个进程/线程安全访问的对象（object）或模块（module）

Linux Futex（Fast Userspace muTEXes）

### ch4.5 Android 中的同步机制

Mutex

Condition

Barrier

### ch6 Binder

Binder 驱动 -> 路由器
Service Manager -> DNS
Binder Client -> 客户端
Binder Server -> 服务器

