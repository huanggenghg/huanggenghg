## Android 开发艺术探索 读书笔记

### Ch1 Activity 的生命周期和启动模式

![Activity 生命周期](https://developer.android.com/images/activity_lifecycle.png)

Activity with transparent theme will no called onStop when another activity comes into the foreground.

![Activity 异常情况重建流程](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/Activity%20%E5%BC%82%E5%B8%B8%E6%83%85%E5%86%B5%E9%87%8D%E5%BB%BA.drawio.png)

LaunchMode：standard、singleTop、singleTask、singleInstance

Activity State and Fragment Callbacks relations:

![Fragment 与 Activity 生命周期对应](https://developer.android.com/static/images/activity_fragment_lifecycle.png?hl=zh-cn)

Lifecycle 原理：底层内部是根据 Activity 生命周期状态反射对应的方法，这样就完成了整个流程，如下图。

![Lifecycle 原理流程](https://upload-images.jianshu.io/upload_images/6345209-6f1fac30f98e8e94.png)

### Ch2 IPC 机制

略过，可查看 [IPC 机制](https://github.com/huanggenghg/huanggenghg/blob/main/IPC%20%E6%9C%BA%E5%88%B6.md)

### Ch3 View 的事件体系

