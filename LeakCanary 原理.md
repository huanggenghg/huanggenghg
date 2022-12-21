## LeakCanary 原理

### 一、理论依据

当一个`Activity`的`onDestory`方法被执行后，说明该`Activity`的生命周期已经走完，在下次`GC`发生时，该`Activity`对象应将被回收。

通过上面对引用的学习，可以考虑在`onDestory`发生时创建一个弱引用指R向`Activity`，并关联一个`RefrenceQuence`,当`Activity`被正常回收，弱引用实例本身应该出现在该`RefrenceQuence`中，否则便可以判断该`Activity`存在内存泄漏。

通过`Application.registerActivityLifecycleCallbacks()`方法可以注册`Activity`生命周期的监听，每当一个`Activity`调用`onDestroy`进行页面销毁时，去获取到这个`Activity`的弱引用并关联一个`ReferenceQuence`，通过检测`ReferenceQuence`中是否存在该弱引用判断这个`Activity`对象是否正常回收。

当`onDestory`被调用后，初步观察到`Activity`未被`GC`正常回收时，手动触发一次`GC`，由于**手动发起`GC`请求后并不会立即执行垃圾回收**，所以需要在一定时延后再二次确认`Activity`是否已经回收，如果再次判断`Activity`对象未被回收，则表示`Activity`存在内存泄漏

### 二、源码解析

方法调用链：`LeakCanary#install(Application application)` -> `AndroidRefWatcherBuilder#buildAndInstall()` -> `ActivityRefWatcher#installOnIcsPlus()` -> `RefWatcher#watch(Activity activity)`

当`Activity`的`onDestory`方法被调用后，`LeakCanary`将在`RefWatcher`的`retainedKeys`加入一条全局唯一的`UUID`，同时创建一个该`Activity`的弱引用对象`KeyedWeakReference`，并将`UUID`写入`KeyedWeakReference`实例中，同时`KeyedWeakReference`与引用队列`queue`进行关联，这样当`Activity`对象正常回收时，该弱引用对象将进入队列当中。 循环遍历获取`queue`队列中的`KeyedWeakReference`对象`ref`，将`ref`中的`UUID`取出，在`retainedKeys`中移除该`UUID`。如果遍历完成后`retainedKeys`中仍然存在该弱引用的`UUID`,则说明该`Activity`对象在`onDestory`调用后没有被正常回收。此时通过`GcTrigger`手动发起一次`GC`,再等待100ms，然后再次判断`Activity`是否被正常回收，如果没有被回收，则开始内存泄漏和展示信息的工作。

*以上摘录自[LeakCanary原理从0到1](https://juejin.cn/post/6936452012058312740)*