## Android 应用启动流程

### 一、Input 触控事件处理流程

`EventHub`
`InputReader`、`InputDispatcher`
`InboundQueue`(iq)、`OutboundQueue`(oq)、`WaitQueue`(wq)、`PendingInputEventQueue`(aq)
`deliveerInputEvent`
`InputResponse`

![Input 模型](https://upload-images.jianshu.io/upload_images/26874665-9a6f99f4f9970bb6.PNG)


### 二、简单应用启动流程

![应用启动流程](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/%E5%BA%94%E7%94%A8%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B.drawio.png)

### 三、创建应用进程

Android系统中一般应用进程的创建都是统一由zygote进程fork创建的，AMS在需要创建应用进程时，会通过socket连接并通知到到zygote进程在开机阶段就创建好的socket服务端，然后由zygote进程fork创建出应用进程。

![应用创建流程图](https://upload-images.jianshu.io/upload_images/26874665-d35e3ceb51181b5a.png)

AMS 发送 socket 请求。在ZygoteProcess#startViaZygote中，最后创建应用进程的逻辑：

1. openZygoteSocketIfNeeded函数中打开本地socket客户端连接到zygote进程的socket服务端；
2. zygoteSendArgsAndGetResult发送socket请求参数，带上了创建的应用进程参数信息；
3. return返回的数据结构ProcessStartResult中会有新创建的进程的pid字段。

zygote进程监听接收AMS的请求，fork创建子应用进程，然后pid为0时进入子进程空间，然后在 ZygoteInit#zygoteInit中完成进程的初始化动作：

1. 应用进程默认的java异常处理机制（可以实现监听、拦截应用进程所有的Java crash的逻辑）；
2. JNI调用启动进程的binder线程池（注意应用进程的binder线程池资源是自己创建的并非从zygote父进程继承的）；
3. 通过反射创建ActivityThread对象并调用其“main”入口方法

`ActivityThraed#main`：

1. 创建并启动主线程的loop消息循环；
2. 通过binder调用AMS的attachApplication接口将自己attach注册到AMS中。

主线程初始化完成后，主线程就有了完整的 Looper、MessageQueue、Handler，此时 ActivityThread 的 Handler 就可以开始处理 Message，包括 Application、Activity、ContentProvider、Service、Broadcast 等组件的生命周期函数，都会以 Message 的形式，在主线程按照顺序处理。可以说Android系统的运行是受消息机制驱动的，而整个消息机制是由上面所说的四个关键角色相互配合实现的。

![Android 消息机制](https://upload-images.jianshu.io/upload_images/26874665-816bcf754eef9a06.jpg)

### 四、应用 Application 和 Activity 组件的创建与初始化

应用`attachApplication`注册请求 -> `AMS#attachApplication` -> `ActivityThread#IApplicationThread`的`bindApplicatoino` -> post `BIND_APPLICATION` -> `handleBindApplication`

1. 根据框架传入的ApplicationInfo信息创建应用APK对应的LoadedApk对象;
2. 创建应用Application的Context对象；
3. 创建类加载器ClassLoader对象并触发Art虚拟机执行OpenDexFilesFromOat动作加载应用APK的Dex文件；
4. 通过LoadedApk加载应用APK的Resource资源；
5. 调用LoadedApk的makeApplication函数，创建应用的Application对象;
6. 执行应用Application#onCreate生命周期函数（APP应用开发者能控制的第一行代码）;

在创建Application的Context对象后会立马尝试去加载APK的Resource资源，而在这之前需要通过LoadedApk去创建类加载器ClassLoader对象，而这个过程最终就会触发Art虚拟机加载应用APK的dex文件。

系统对于应用APK文件资源的加载过程其实就是创建应用进程中的Resources资源对象的过程，其中真正实现APK资源文件的I/O解析作，最终是借助于AssetManager中通过JNI调用系统Native层的相关C函数实现。

应用进程在主线程执行handleBindeApplication初始化操作，然后继续执行启动应用Activity的操作。框架system_server进程最终是通过ActivityStackSupervisor#realStartActivityLocked函数中，通过LaunchActivityItem和ResumeActivityItem两个类的封装，依次实现binder调用通知应用进程这边执行Activity的Launch和Resume动作的。

应用进程这边在收到系统binder调用后，在主线程中创建Activiy的流程主要步骤如下：

1. 创建Activity的Context；
2. 通过反射创建Activity对象；
3. 执行Activity的attach动作，其中会创建应用窗口的PhoneWindow对象并设置WindowManage；
4. 执行应用Activity的onCreate生命周期函数，并在setContentView中创建窗口的DecorView对象；

应用进程这边在接收到系统Binder调用请求后，在主线程中Activiy Resume的流程主要步骤如下：

1. 执行应用Activity的onResume生命周期函数;
2. 执行WindowManager的addView动作开启视图绘制逻辑;
3. 创建Activity的ViewRootImpl对象;
4. 执行ViewRootImpl的setView函数开启UI界面绘制动作；

### 五、应用 UI 布局与绘制

![GUI_APP](https://upload-images.jianshu.io/upload_images/26874665-ab588cdefb9a311f.png)

应用主线程中在执行Activity的Resume流程的最后，会创建ViewRootImpl对象并调用其setView函数，从此并开启了应用界面UI布局与绘制的流程：

1. requestLayout()通过一系列调用触发界面绘制（measure、layout、draw）动作，下文会详细展开分析；
2. 通过Binder调用访问系统窗口管理服务WMS的addWindow接口，实现添加、注册应用窗口的操作，并传入本地创建inputChannel对象用于后续接收系统的触控事件，这一步执行完我们的View就可以显示到屏幕上了。
3. 创建WindowInputEventReceiver对象，封装实现应用窗口接收系统触控事件的逻辑；
4. 执行view.assignParent(this)，设置DecorView的mParent为ViewRootImpl。所以，虽然ViewRootImpl不是一个View,但它是所有View的顶层Parent。

```java
/*frameworks/base/core/java/android/view/ViewRootImpl.java*/
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
         // 检查当前UI绘制操作是否发生在主线程，如果发生在子线程则会抛出异常
         checkThread();
         mLayoutRequested = true;
         // 触发绘制操作
         scheduleTraversals();
    }
}

@UnsupportedAppUsage
void scheduleTraversals() {
    if (!mTraversalScheduled) {
         ...
         // 注意此处会往主线程的MessageQueue消息队列中添加同步栏删，因为系统绘制消息属于异步消息，需要更高优先级的处理
         mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
         // 通过Choreographer往主线程消息队列添加CALLBACK_TRAVERSAL绘制类型的待执行消息，用于触发后续UI线程真正实现绘制动作
         mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
         ...
     }
}
```

Choreographer 扮演 Android 渲染链路中承上启下的角色：

- 承上：负责接收和处理 App 的各种更新消息和回调，等到 Vsync 到来的时候统一处理。比如集中处理 Input(主要是 Input 事件的处理) 、Animation(动画相关)、Traversal(包括 measure、layout、draw 等操作) ，判断卡顿掉帧情况，记录 CallBack 耗时等；
- 启下：负责请求和接收 Vsync 信号。接收 Vsync 事件回调(通过 FrameDisplayEventReceiver.onVsync )，请求 Vsync(FrameDisplayEventReceiver.scheduleVsync) 。

![Choreographer工作原理](https://upload-images.jianshu.io/upload_images/26874665-a44a2100a0f46092.jpg)

ViewRootImpl中负责的整个应用界面绘制的主要流程如下：

1. 从界面View控件树的根节点DecorView出发，递归遍历整个View控件树，完成对整个View控件树的measure测量操作，由于篇幅所限，本文就不展开分析这块的详细流程；
2. 界面第一次执行绘制任务时，会通过Binder IPC访问系统窗口管理服务WMS的relayout接口，实现窗口尺寸的计算并向系统申请用于本地绘制渲染的Surface“画布”的操作（具体由SurfaceFlinger负责创建应用界面对应的BufferQueueLayer对象，并通过内存共享的方式通过Binder将地址引用透过WMS回传给应用进程这边），由于篇幅所限，本文就不展开分析这块的详细流程；
3. 从界面View控件树的根节点DecorView出发，递归遍历整个View控件树，完成对整个View控件树的layout测量操作；
4. 从界面View控件树的根节点DecorView出发，递归遍历整个View控件树，完成对整个View控件树的draw测量操作，如果开启并支持硬件绘制加速（从Android 4.X开始谷歌已经默认开启硬件加速），则走GPU硬件绘制的流程，否则走CPU软件绘制的流程；

![UI 绘制流程](https://upload-images.jianshu.io/upload_images/26874665-79225531199ba7ce.png)

### 六、RenderThread 渲染

硬件加速绘制主要包括两个阶段：

1. 从DecorView根节点出发，递归遍历View控件树，记录每个View节点的drawOp绘制操作命令，完成绘制操作命令树的构建；
2. JNI调用同步Java层构建的绘制命令树到Native层的RenderThread渲染线程，并唤醒渲染线程利用OpenGL执行渲染任务；

构建绘制命令树的过程是从View控件树的根节点DecorView触发，递归调用每个子View节点的updateDisplayListIfDirty函数，最终完成绘制树的创建，简述流程如下：

1. 利用View对象构造时创建的RenderNode获取一个SkiaRecordingCanvas“画布”；
2. 利用SkiaRecordingCanvas，在每个子View控件的onDraw绘制函数中调用drawLine、drawRect等绘制操作时，创建对应的DisplayListOp绘制命令，并缓存记录到其内部的SkiaDisplayList持有的DisplayListData中；
3. 将包含有DisplayListOp绘制命令缓存的SkiaDisplayList对象设置填充到RenderNode中；
4. 最后将根View的缓存DisplayListOp设置到RootRenderNode中，完成构建。

![绘制命令树构建](https://upload-images.jianshu.io/upload_images/26874665-1195656a32dbec9e.png)

![硬件加速下View绘制树的结构](https://upload-images.jianshu.io/upload_images/26874665-a951aa2dfda7c791.png)

UI线程利用RenderProxy向RenderThread线程发送一个DrawFrameTask任务请求，RenderThread被唤醒，开始渲染，大致流程如下：

1. syncFrameState中遍历View树上每一个RenderNode，执行prepareTreeImpl函数，实现同步绘制命令树的操作；
2. 调用OpenGL库API使用GPU，按照构建好的绘制命令完成界面的渲染（具体过程，由于本文篇幅所限，暂不展开分析）；
3. 将前面已经绘制渲染好的图形缓冲区Binder上帧给SurfaceFlinger合成和显示；

![RenderThread 线程渲染流程](https://upload-images.jianshu.io/upload_images/26874665-7218830c7bb346ae.png)

### 七、SurfaceFlinger 合成显示

SurfaceFlinger作为系统中独立运行的一个Native进程，借用Android官网的描述，其职责就是负责接受来自多个来源的数据缓冲区，对它们进行合成，然后发送到显示设备。

![SurfaceFlinger工作原理](https://upload-images.jianshu.io/upload_images/26874665-cb33efbd47f23d22.jpg)

SurfaceFlinger在Android系统的整个图形显示系统中是起到一个承上启下的作用：
- 对上：通过Surface与不同的应用进程建立联系，接收它们写入Surface中的绘制缓冲数据，对它们进行统一合成。
- 对下：通过屏幕的后缓存区与屏幕建立联系，发送合成好的数据到屏幕显示设备。

![BufferQueue 状态转换图](https://upload-images.jianshu.io/upload_images/26874665-05c18df7fb448c79.jpg)

生产者-消费者渲染送显模型：

1. 应用进程中在开始界面的绘制渲染之前，需要通过Binder调用dequeueBuffer接口从SurfaceFlinger进程中管理的BufferQueue 中申请一张处于free状态的可用Buffer，如果此时没有可用Buffer则阻塞等待；
2. 应用进程中拿到这张可用的Buffer之后，选择使用CPU软件绘制渲染或GPU硬件加速绘制渲染，渲染完成后再通过Binder调用queueBuffer接口将缓存数据返回给应用进程对应的BufferQueue（如果是 GPU 渲染的话，这里还有个 GPU处理的过程，所以这个 Buffer 不会马上可用，需要等 GPU 渲染完成的Fence信号），并申请sf类型的Vsync以便唤醒“消费者”SurfaceFlinger进行消费；
3. SurfaceFlinger 在收到 Vsync 信号之后，开始准备合成，使用 acquireBuffer获取应用对应的 BufferQueue 中的 Buffer 并进行合成操作；
4. 合成结束后，SurfaceFlinger 将通过调用 releaseBuffer将 Buffer 置为可用的free状态，返回到应用对应的 BufferQueue中。

Vsync 同步机制：为了协调BufferQueue的应用生产者生成UI数据动作和SurfaceFlinger消费者的合成消费动作，避免出现画面撕裂的Tearing现象。
Vysnc信号分为两种类型：

- app类型的Vsync：app类型的Vysnc信号由上层应用中的Choreographer根据绘制需求进行注册和接收，用于控制应用UI绘制上帧的生产节奏。Vsync信号到来后，才能往主线程的消息队列放入待绘制任务进行真正UI的绘制动作；
- sf类型的Vsync:sf类型的Vsync是用于控制SurfaceFlinger的合成消费节奏。应用完成界面的绘制渲染后，通过Binder调用queueBuffer接口将缓存数据返还给应用对应的BufferQueue时，会申请sf类型的Vsync，待SurfaceFlinger 在其UI线程中收到 Vsync 信号之后，便开始进行界面的合成操作。

![Vsync 架构](https://upload-images.jianshu.io/upload_images/26874665-7a7e75039d05d786.png)

### 八、总结

![应用冷启动流程](https://upload-images.jianshu.io/upload_images/26874665-3228fe250c4f1092.png)

### *九、说明*

*以上为博文[Android应用启动全流程分析（源码深度剖析）](https://www.jianshu.com/p/37370c1d17fc)的摘录。*




