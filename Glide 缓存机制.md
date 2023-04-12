## Glide 缓存机制

### 一、大致概况
**多级缓存**：
- 活动资源（Active Resources）：现在是否有另一个 View 正在展示这张图片
- 内存缓存（Memory Cache）：该图片是否最近被加载过并仍存在于内存中
- 资源类型（Resource）：该图片是否之前曾被解码、转换并写入过磁盘缓存
- 数据来源（Data）：构建这个图片的资源是否之前曾被写入过文件缓存

缓存键（Cache Keys）: 
1. 请求加载的 model（File，Uri，Url）
2. 一个可选的签名（Signature）

另外，步骤 1-3（活动资源、内存缓存、资源磁盘缓存）的缓存键还包含一些其他数据，包括：
1. 宽度和高度
2. 可选的变换（Transformation）
3. 额外添加的任何选项
4. 请求的数据类型（Bitmap，GIF，或其他）

活动资源和内存缓存使用的键还和磁盘资源缓存略有不同，以适应内存 选项(Options)，比如影响 Bitmap 配置的选项或其他解码时才会用到的参数。为了生成磁盘缓存上的缓存键名称，以上的每个元素会被哈希化以创建一个单独的字符串键名，并在随后作为磁盘缓存上的文件名使用。

配置缓存

`diskCacheStrategy`：`AUTOMATIC`(当你加载远程数据（比如，从URL下载）时，AUTOMATIC 策略仅会存储未被你的加载过程修改过(比如，变换，裁剪–译者注)的原始数据，因为下载远程数据相比调整磁盘上已经存在的数据要昂贵得多。对于本地数据，AUTOMATIC 策略则会仅存储变换过的缩略图，因为即使你需要再次生成另一个尺寸或类型的图片，取回原始数据也很容易。)

`onlyRetrieveFromCache(true)`：仅从缓存中加载（省流离线模式？）
`skipMemoryCache(true)`：跳过磁盘和/或内存缓存
`DiskCacheStrategy.NONE`：仅跳过磁盘缓存

**缓存的刷新**

缓存缩缩略图和提供多种变换会创建新的不同的缓存文件，跟踪删除一个图片的所有版本比较困难。

定制缓存刷新策略

```java
Glide.with(yourFragment)
    .load(yourFileDataModel)
    .signature(new ObjectKey(yourVersionMetadata))
    .into(yourImageView);

Glide.with(fragment)
    .load(mediaStoreUri)
    .signature(new MediaStoreSignature(mimeType, dateModified, orientation))
    .into(view);
```

资源管理

磁盘和内存缓存都是 LRU：
AppGlideModule 设置可用 RAM 大小

暂时调整：`Glide.get(context).clearMemory()`

```java
new AsyncTask<Void, Void, Void> {
  @Override
  protected Void doInBackground(Void... params) {
    // This method must be called on a background thread.
    Glide.get(applicationContext).clearDiskCache();
    return null;
  }
}
```

### 二、缓存机制

3级缓存`WeakReference + Lrucache + DiskLrucache`

缓存 key 参数 8 种：
```java
#Engine.load()
//生成缓存key
EngineKey key = keyFactory.buildKey(model, signature, width, height, transformations,
        resourceClass, transcodeClass, options);
```

`loadFromActiveResources`(内部是弱引用持有资源) -> `loadFromCache`（获取到会从 LruCache 中删除这张图片，把图片从 Lruccache 中转移至弱引用）-> 通过线程池加载图片

EngineJob 内部维护了线程池，用来管理资源加载，当资源加载完毕的时候通知回调； DecodeJob 是线程池中的一个任务。最后通过start()方法加载图片,实际上是在DecodeJob的run()方法中完成的，当图片加载完成，最终会回调EngineJob#onResourceReady ()

1. 图片引用计数器+1；
2. listener.onEngineJobComplete()，这个listener是EngineJobListener接口对象，这里是将结果回调给Engine#onEngineJobComplete()处理；
3. 遍历遍历加载的图片，每加载到一张图片，引用计数器+1 ，并且会将资源回调给ImageView去加载；
4. 释放资源，图片引用计数器-1 

```java
public class Engine implements EngineJobListener,
    MemoryCache.ResourceRemovedListener,
    EngineResource.ResourceListener {

@Override
  public void onResourceReleased(Key cacheKey, EngineResource<?> resource) {
    Util.assertMainThread();
    //从弱引用集合activeResources中移除资源
    activeResources.deactivate(cacheKey);
    if (resource.isCacheable()) {
      //放入LruCache缓存
      cache.put(cacheKey, resource);
    } else {
      //回收资源
      resourceRecycler.recycle(resource);
    }
  }
}
```
> 调用EngineJob#handleResultOnMainThread ()去加载图片等时候，如果加载图片成功，那么acquired>=1，说明有图片正在被引用；而等到暂停请求／退出页面的时候再次调用release()时，acquired==0才会去调用onResourceReleased ()把缓存从弱引用转移到Lrucache。

**这个acquired变量是用来记录图片被引用的次数，调用acquire()方法会让变量加1，调用release()方法会让变量减1。当调用loadFromActiveResources()、loadFromCache()、EngineJob#handleResultOnMainThread()获取图片的时候都会执行acquire()方法；当暂停请求或者加载完毕或者清除资源时会调用release()方法。**

**注意：从弱引用取缓存，拿到的话，引用计数+1；从LruCache中拿缓存，拿到的话，引用计数也是+1，同时把LruCache缓存转移到弱应用缓存池中；从EngineJob去加载图片，拿到的话，引用计数也是+1，会把图片放到弱引用。反过来，一旦没有地方正在使用这个资源，就会将其从弱引用中转移到LruCache缓存池中。这也说明了正在使用中的图片使用弱引用来进行缓存，暂时不用的图片使用LruCache来进行缓存的功能。**

磁盘缓存策略
DiskCacheStrategy.DATA: 只缓存原始图片；
DiskCacheStrategy.RESOURCE:只缓存转换过后的图片；
DiskCacheStrategy.ALL:既缓存原始图片，也缓存转换过后的图片；对于远程图片，缓存 DATA和 RESOURCE；对于本地图片，只缓存 RESOURCE；
DiskCacheStrategy.NONE：不缓存任何内容；
DiskCacheStrategy.AUTOMATIC：默认策略，尝试对本地和远程图片使用最佳的策略。当下载网络图片时，使用DATA；对于本地图片，使用RESOURCE；

如果在内存缓存中没获取到数据，就通过`DecodeJob`和`EngineJob`加载图片。EngineJob 内部维护了线程池，用来管理资源加载，当资源加载完毕的时候通知回调； DecodeJob 是线程池中的一个任务。

`DecodeJob#run()` -> `runWrapper()` -> `ResourcesCacheGenerator\SourceGenerator\DataCacheGenerator`

> - ResourceCacheGenerator:管理变换之后的缓存数据；
> - SourceGenerator:管理未经转换的原始缓存数据；
> - DataSourceGenerator：直接从网络下载解析数据。

-> `getNextStage` -> `getNextGenerator` -> `runGenerators`

如果是第一次从网络加载图片的话，最终数据的加载会交给 SourceGenerator 进行;如果是从磁盘缓存获取的话会根据缓存策略的不同从ResourceCacheGenerator或者DataCacheGenerator获取。
1. 构建获取缓存信息键
2. 从缓存中获取缓存信息

缓存写入`SourceGenerator#startNext()` -> `HttpUrlFetcher#loadData()` -> `SourceGenerator#onDataReady()` -..-> `cacheData()` -> `DiskLruCacheWrapper`

小结：
磁盘缓存是在EngineJob中的DecodeJob任务中完成的，依次通过ResourcesCacheGenerator、SourceGenerator、DataCacheGenerator来获取缓存数据。ResourcesCacheGenerator获取的是转换过的缓存数据；SourceGenerator获取的是未经转换的原始的缓存数据；DataCacheGenerator是通过网络获取图片数据再按照按照缓存策略的不同去缓存不同的图片到磁盘上。

### 二、（缓存机制）总结

**Glide缓存分为弱引用+ LruCache+ DiskLruCache，其中读取数据的顺序是：弱引用 > LruCache > DiskLruCache>网络；写入缓存的顺序是：网络 --> DiskLruCache--> (LruCache-->弱引用, acquired)**

**内存缓存分为弱引用的和 LruCache ，其中正在使用的图片使用弱引用缓存，暂时不使用的图片用 LruCache缓存，这一点是通过 图片引用计数器（acquired变量）来实现的，详情可以看内存缓存的小结。**

**磁盘缓存就是通过DiskLruCache实现的，根据缓存策略的不同会获取到不同类型的缓存图片。它的逻辑是：先从转换后的缓存中取；没有的话再从原始的（没有转换过的）缓存中拿数据；再没有的话就从网络加载图片数据，获取到数据之后，再依次缓存到磁盘和弱引用。**

> 为什么Glide内存缓存要设计2层，弱引用和LruCache？
> 用弱引用缓存的资源都是当前活跃资源 activeRource，资源的使用频率比较高，这个时候如果从LruCache取资源，LinkHashmap查找资源的效率不是很高的。所以他会设计一个弱引用来缓存当前活跃资源，来替Lrucache减压。

### 三、关联生命周期

- Glide：供外部调用的核心类，外观模式；
- RequestManagerRetriever：是关联RequestManager和SupportRequestManagerFragment／RequestManagerFragment的中间类；
- RequestManager：是用来加载、管理图片请求的，会结合Activity／Fragment生命周期对请求进行管理；
- SupportRequestManagerFragment／RequestManagerFragment：Glide内部创建无UI的fragment，会与当前Activity绑定，与RequestManager绑定，传递页面的生命周期。其中SupportRequestManagerFragment是v4包下的Fragment，RequestManagerFragment：Glide是android.app.Fragment；
- ActivityFragmentLifecycle：保存fragment和Requestmanager映射关系的类，管理LifecycleListener；
- LifecycleListener：定义生命周期管理方法的接口，onStart(), onStop(), onDestroy()。

1. 创建无UI的Fragment，并绑定到当前activity；
2. 通过builder模式创建RequestManager，并且将fragment的lifecycle传入，这样Fragment和RequestManager就建立了联系;
3. 获取Fragment对象，先根据tag去找，如果没有从内存中查找，pendingSupportRequestManagerFragments是一个存储fragment实例的HashMap，再没有的话就new一个。

1. 在创建Fragment的时候会创建ActivityFragmentLifecycle对象；
2. 在Fragment生命周期的方法中会调用Lifecycle的相关方法来通知RequestManager；
3. LifecycleListener 是一个接口，Lifecycle最终是调用了lifecycleListener来通知相关的实现类的，也就是RequestManager。

小结

1. **创建一个无UI的Fragment，具体来说是SupportRequestManagerFragment／RequestManagerFragment，并绑定到当前Activity，这样Fragment就可以感知Activity的生命周期；**
2. **在创建Fragment时，初始化Lifecycle、LifecycleListener，并且在生命周期的onStart() 、onStop()、 onDestroy()中调用相关方法；**
3. **在创建RequestManager时传入Lifecycle 对象，并且LifecycleListener实现了LifecycleListener接口；**
4. **这样当生命周期变化的时候，就能通过接口回调去通知RequestManager处理请求。**

其他

1. 传入Application Context或者在子线程使用：调用getApplicationManager(context);这样Glide的生命周期就和应用程序一样了。（而这里的主要差异是获取Lifecycle对象不一样，用的是ApplicationLifecycle，由于没有创建Fragment，这里只会调用onStart()，这种情况Glide生命周期就和Application一样长了。）
2. 这里有个问题很奇怪，刚把创建的fragment对象放入map缓存起来，但是马上又通过handler把它删除，这是什么情况？———— 原因是这样的，当调用FragmentManager的add()方法时，是通过开启事务的方式来进行绑定的，这个过程是异步的，具体来说，就是调用add方法时，并不是马上就和activity绑定好了，而是通过Hander来处理的。

### *四、摘录自*

[Android 【手撕Glide】--Glide缓存机制](https://www.jianshu.com/p/b85f89fce019)
[Android 【手撕Glide】--Glide缓存机制（面试）](https://www.jianshu.com/p/ba7f38ede854)
[Android 【手撕Glide】--Glide是如何关联生命周期的？](https://www.jianshu.com/p/79dd4953ec25)








