## LiveDataBus

### EventBus

消息的发布与订阅

![EventBus 消息的发布与订阅](https://p1.meituan.net/travelcube/87d7f0f7e01b188aa312c91b2be45fe828898.png)

`EventBus.getDefault().register(this);`

![register](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/10/18/16ddde2e9b75be3d~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.image)


METHOD_CACHE---- ConcurrentHashMap

**反射生成。--- 运行时 获取注解信息**
**APT 生成，加速。 --- 生成啥 SubscriberInfoIndex，编译时注解信息 indexMap SUBSCRIBER_INDEX**
订阅事件方法。

> Java 反射效率低的主要原因：
> [大家都说 Java 反射效率低，你知道原因在哪里么](https://juejin.cn/post/6844903965725818887)
>
> - Method#invoke 方法会对参数做封装和解封操作
> - 需要检查方法可见性
> - 需要校验参数
> - 反射方法难以内联
> - JIT 无法优化

`unregister`

移除

`post`

根据处理线程进行处理，反射，或是 handler 加到消息队列中于 handleMessages 中处理，或是 backgroud 交给线程池通过反射调用。

`postSticky`

map 存储，有新订阅时查看有没有，有就发送事件。`checkPostStickyEventToSubscription`

### RxBus

### LiveDataBus

![LiveDataBus 订阅分发流程](https://pic3.zhimg.com/80/v2-e2c9c2051b16fe7030a97f2d65420172_1440w.webp)

粘性事件：下次订阅还会收到上一次的事件。

首先发送数据 postValue ，次数会让进行 mVersion++ 操作，然后遍历观察者进行分发。
然后是进行监听操作，在进行监听的时候，会使用LifecycleBoundObserver 对观察者进行包装一下，在这个操作里面，LifecycleBoundObserver 的父类 ObserverWrapper 定义了 mLastVersion 为-1 。在数据最后进行分发的时候，mLastVersion 是小于 mVersion 的，所以未拦截，然后进行了数据的分发。
然后就产生了粘性事件。

粘性事件的处理：[关于LiveData粘性事件所带来问题的解决方案](https://www.jianshu.com/p/d0244c4c7cc9)

