## RecyclerView 缓存机制

### 一、why ViewHolder

避免多次`findViewById()`；ItemView 和 ViewHolder 一对一关系。

### 二、ListView 缓存机制

![ListView 缓存机制](https://upload-images.jianshu.io/upload_images/2477378-826eee13beb0e270.png?imageMogr2/auto-orient/strip|imageView2/2/format/webp)

![ListView 缓存机制](https://upload-images.jianshu.io/upload_images/2477378-e4406d2c3ed6cce6.png?imageMogr2/auto-orient/strip|imageView2/2/format/webp)

### 三、RecyclerView 缓存机制

![RecyclerView 缓存机制](https://upload-images.jianshu.io/upload_images/2477378-189d9cf6ea337cb5.png?imageMogr2/auto-orient/strip|imageView2/2/format/webp)

- Scrap：对应ListView 的Active View，就是屏幕内的缓存数据，就是相当于换了个名字，可以直接拿来复用

- Cache：刚刚移出屏幕的缓存数据，默认大小是2个，当其容量被充满同时又有新的数据添加的时候，会根据FIFO原则，把优先进入的缓存数据移出并放到下一级缓存中，然后再把新的数据添加进来。Cache里面的数据是干净的，也就是携带了原来的ViewHolder的所有数据信息，数据可以直接来拿来复用。需要注意的是，cache是根据position来寻找数据的，这个postion是根据第一个或者最后一个可见的item的position以及用户操作行为（上拉还是下拉）。
举个栗子：当前屏幕内第一个可见的item的position是1，用户进行了一个下拉操作，那么当前预测的position就相当于（1-1=0），也就是position=0的那个item要被拉回到屏幕，此时RecyclerView就从Cache里面找position=0的数据，如果找到了就直接拿来复用。

- ViewCacheExtension：自定义缓存，慎用。

- RecycledViewPool：Cache默认的缓存数量是2个，当Cache缓存满了以后会根据FIFO（先进先出）的规则把Cache先缓存进去的ViewHolder移出并缓存到RecycledViewPool中，RecycledViewPool默认的缓存数量是5个。RecycledViewPool与Cache相比不同的是，从Cache里面移出的ViewHolder再存入RecycledViewPool之前ViewHolder的数据会被全部重置，相当于一个新的ViewHolder，而且Cache是根据position来获取ViewHolder，而RecycledViewPool是根据itemType获取的，如果没有重写getItemType（）方法，itemType就是默认的。因为RecycledViewPool缓存的ViewHolder是全新的，所以取出来的时候需要走onBindViewHolder（）方法。

![RecyclerView 缓存机制](https://upload-images.jianshu.io/upload_images/2477378-de3adb888fd5f9ec.png?imageMogr2/auto-orient/strip|imageView2/2/format/webp)

### 四、RecyelerView 缓存机制源码

`LayoutManager#next()` -> `Recycler` -> `RecycledViewPool` -> `recycler.getViewForPosition(mCurrentPosition)` -> `getScrapOrHiddenOrCachedHolderForPosition(position)` -> `getRecycledViewPool().getRecycledView(type)`

### 五、总结

ListView有两级缓存，分别是Active View和Scrap View，缓存的对象是ItemView；而RecyclerView有四级缓存，分别是Scrap、Cache、ViewCacheExtension和RecycledViewPool，缓存的对象是ViewHolder。

Scrap和Cache分别是通过position去找ViewHolder可以直接复用；ViewCacheExtension自定义缓存，目前来说应用场景比较少却需慎用；RecycledViewPool通过type来获取ViewHolder，获取的ViewHolder是个全新，需要重新绑定数据。

关于RecyclerView的缓存分为四级，Scrap、Cache、ViewCacheExtension和RecycledViewPool。Scrap是屏幕内的缓存一般我们不怎么需要特别注意；Cache可直接拿来复用的缓存，性能高效；ViewCacheExtension需要开发者自定义的缓存，API设计比较奇怪，慎用；RecycledViewPool四级缓存，可以避免用户调用onCreateViewHolder（）方法，提高性能。

**缓存区别**

- 层级不同：
ListView有两级缓存，在屏幕与非屏幕内。mActivityViews + mScrapViews
RecyclerView比ListView多两级缓存：支持开发者自定义缓存处理逻辑，RecyclerViewPool(缓存池)。并且支持多个离屏ItemView缓存（缓存屏幕外2个 itemView）。 mAttachedScrap + mCacheViews + mViewCacheExtension + mRecyclerPool

- 缓存内容不同：
ListView缓存View。
RecyclerView缓存RecyclerView.ViewHolder

- RV优势
a.mCacheViews的使用，可以做到屏幕外的列表项ItemView进入屏幕内时也无须bindView快速重用；
b.mRecyclerPool可以供多个RecyclerView共同使用，在特定场景下，如viewpaper+多个列表页下有优势.