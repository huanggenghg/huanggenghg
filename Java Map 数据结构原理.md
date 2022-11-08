## Java Map 数据结构原理

### 一、HashMap

基于 Map 接口实现、允许 null 键/值、非同步、不保证有序（如插入的顺序）、也不保证序不随时间变化。

```java
    public HashMap(int initialCapacity, float loadFactor) {
        ...
        this.loadFactor = loadFactor; // buckets 填满程度的最大比例
        this.threshold = tableSizeFor(initialCapacity); // initialCapacity buckets 的数目
    }
```

`put`实现：

![put流程](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/HashMap_put.drawio.png)

具体代码实现自行查看源码。

`get`实现：

1. bucket 里的第一个节点，直接命中；
2. 如果有冲突，则通过`key.equals(k)`去查找对应的 entry。树时间复杂度：O(logn)；链表时间复杂度：O(n)。

`hash`实现：

![hash实现](https://cloud.githubusercontent.com/assets/1736354/6957712/293b52fc-d932-11e4-854d-cb47be67949a.png)

```java
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```

高16bit不变，低16bit和高16bit做了一个异或。

计算下标：

```java
(n - 1) & hash
```

这个方法易发生碰撞。通过`hash(Object key)`减少因 hashCode 高位没有参与下标的计算，在 table 长度比较小时，容易造成的冲突。

`resize`实现：2 次幂扩展，元素的位置要么是在原位置，要么是在原位置再移动 2 次幂的位置。

![16_32](https://cloud.githubusercontent.com/assets/1736354/6958256/ceb6e6ac-d93b-11e4-98e7-c5a5a07da8c4.png)

resize 之后，n 变为原来的 2 倍，则 n-1 的 mask 范围在高位多 1 bit。

![index resize 后的变化](https://cloud.githubusercontent.com/assets/1736354/6958301/519be432-d93c-11e4-85bb-dff0a03af9d3.png)

具体代码实现自行查看源码。

以上一些摘录自[Java HashMap工作原理及实现](https://yikun.github.io/2015/04/01/Java-HashMap%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86%E5%8F%8A%E5%AE%9E%E7%8E%B0/)，另外红黑树相关的可查看[HashMap 分析之红黑树树化过程](https://www.cnblogs.com/finite/p/8251587.html)

### 二、LinkedHashMap

可查看[LRUCache 算法](https://github.com/huanggenghg/huanggenghg/blob/main/LRUCache%20.md)中关于 LinkedHashMap 内部的描述。

### 三、HashTable

是一个线程安全的哈希表，它通过使用`synchronized`对方法进行加锁，从而保证了线程安全，但这也导致了在单线程环境中效率低下等问题。与 HashMap 不同，不允许插入 null 值及 null 键。

```java
    public Hashtable() {
        this(11, 0.75f); // 初始化容量 11
    }

    public synchronized V get(Object key) {
    	...
    	int hash = key.hashCode()
    	int index = (hash & 0x7FFFFFFF) % tab.length
    }

    public synchronized V put(K key, V value) {
        // Make sure the value is not null
        if (value == null) {
            throw new NullPointerException();
        }

        // Makes sure the key is not already in the hashtable.
        Entry<?,?> tab[] = table;
        int hash = key.hashCode();
        int index = (hash & 0x7FFFFFFF) % tab.length;
        @SuppressWarnings("unchecked")
        Entry<K,V> entry = (Entry<K,V>)tab[index];
        for(; entry != null ; entry = entry.next) {
            if ((entry.hash == hash) && entry.key.equals(key)) { // 存在相同的 key
                V old = entry.value;
                entry.value = value;
                return old;
            }
        }

        addEntry(hash, key, value, index);
        return null;
    }

```

HashTable 扩容操作`rehash`过程需要对每个键值对都重新计算哈希值，相比 HashMap 的异或和与操作，取模是一个非常耗时的操作，这也是其效率低下的原因之一。

这部分摘录于[Hashtable 原理分析](https://www.jianshu.com/p/8be9814172e7)。

### 四、ConcurrentHashMap

#### JDK 8：

底层数据结构：数组 + 链表 + 红黑树

实现线程安全：synchronized + CAS 算法（`Unsafe.compareAndSwapXXX`）

扩容方法`transfer()`：过程中没有进行加锁，并且支持多线程进行扩容操作。`sizeCtl`和`transferIndex`。步骤：

1. 根据 CPU 核数和数组长度，计算每个线程应该处理的桶数量，如果CPU为单核，则使用一个线程处理所有桶。

2. 根据当前数组长度n，新建一个两倍长度的数组 nextTable（这个步骤是单线程执行的）

3. 将原来 table 中的元素复制到 nextTable 中，这里允许多线程进行操作：

   ① 初始化 ForwardingNode 对象，充当占位节点，hash 值为 -1，该占位对象存在时表示集合正在扩容状态；

   >ForwardingNode 的 key、value、next 属性均为 null ，nextTable 属性指向扩容后的数组，它的作用主要有以下两个：
   >
   >占位作用，用于标识数组该位置的桶已经迁移完毕
   >作为一个转发的作用，扩容期间如果遇到查询操作，遇到转发节点，会把该查询操作转发到新的数组上去，不会阻塞查询操作。

   ② 通过 for 循环从右往左依次迁移当前线程所负责数组：
   	a. 如果当前桶没有元素，则直接通过 CAS 放置一个 ForwardingNode 占位对象，以便查询操作的转发和标识当前位置已经被处理过；
   	b. 如果线程遍历到节点的 hash 值为 MOVE，也就是 -1（即 ForwardingNode 节点），则直接跳过，继续处理下一个桶中的节点；

   ​	c. 如果不满足上面两种情况，则直接给当前桶节点加上 synchronized 锁，然后重新计算该桶的元素在新数组中的应该存放的位置，并进行数据迁移。重计算节点的存放位置时，通过 CAS 把低位节点 lowNode 设置到新数组的 i 位置，高位节点 highNode 设置到 i+n 的位置（i 表示在原数组的位置，n表示原数组的长度）；

   ​	d. 当前桶位置的数据迁移完成后，将 ForwardingNode 占位符对象设置到当前桶位置上，表示该位置已经被处理了。

4. 每当一条线程扩容结束就会更新一次 sizeCtl 的值，进行减 1 操作，当最后一条线程扩容结束后，需要重新检查一遍数组，防止有遗漏未成功迁移的桶。扩容结束后，将 nextTable 设置为 null，表示扩容已结束，将 table 指向新数组，sizeCtl 设置为扩容阈值。

   > sizeCtl：是一个控制标识符，在不同的地方有不同用途，它不同的取值不同也代表不同的含义。在扩容时，它代表的是当前并发扩容的线程数量
   >
   > - 负数代表正在进行初始化或扩容操作：-1代表正在初始化，-N 表示有N-1个线程正在进行扩容操作
   > - 正数或0代表hash表还没有被初始化或下一次进行扩容的大小，这一点类似于扩容阈值的概念。

`put()`方法的`helpTransfer()`协助扩容：

- 如果检测到 ConcurrentHashMap 正在进行扩容操作，也就是当前桶位置上被插入了 ForwardingNode 节点，那么当前线程也要协助进行扩容，协助扩容时会调用 helpTransfer() 方法，当方法被调用的时候，当前 ConcurrentHashMap 一定已经有了 nextTable 对象，首先拿到这个 nextTable 对象，调用transfer方法。
- 如果检测到要插入的节点是非空且不是 ForwardingNode  节点，就对这个节点加锁，这样就保证了线程安全。尽管这个有一些影响效率，但是还是会比hashTable 的 synchronized 要好得多。

#### JDK 7:

底层数据结构：“Segment 数组 + HashEntry 数组 + 链表”，一个 ConcurrentHashMap 实例中包含若干个 Segment 实例组成的数组，每个 Segment 实例又包含若干个桶，每个桶中都是由若干个 HashEntry 对象链接起来的链表。

实现线程安全：Segment 类继承自 ReentrantLock 类，充当锁的角色。默认支持 16 个线程执行并发写操作，及任意数量线程的读操作。

![ConcurrentHashMap 实例结构](https://img-blog.csdn.net/20181018022508312?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2E3NDUyMzM3MDA=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

`put()`添加元素的过程：

1. 对 key 值进行重哈希，并使用重哈希的结果与 segmentFor() 方法， 计算出元素具体分到哪个 segment 中。插入元素前，先使用 lock() 对该 segment 加锁，之后再使用头插法插入元素。如果其他线程经过计算也是放在这个 segment 下，则需要先获取锁，如果计算得出放在其他的 segment，则正常执行，不会影响效率，以此实现线程安全，这样就能够保证只要多个修改操作不发生在同一个 segment  时，它们就可以并发进行。

2. 在将元素插入到 segment 前，会检查本次插入会不会导致 segment 中元素的数量超过阈值，如果会，那么就先对 segment 进行扩容和重哈希操作，然后再进行插入。而重哈希操作，实际上是对 ConcurrentHashMap 的某个 segment 的重哈希，因此 ConcurrentHashMap 的每个 segment 段所包含的桶位也就不尽相同。

   > 如果元素的 hash 值与原数组长度进行位与运算，得到的结果为0，那么元素在新桶的序号就是和原桶的序号是相等的；否则元素在新桶的序号就是原桶的序号加上原数组的长度。与 HashMap 一样。

ConcurrentHashMap 读操作为什么不需要加锁：

1. 在 HashEntry 类中，key，hash 和 next 域都被声明为 final 的，value 域被 volatile 所修饰，因此 HashEntry 对象几乎是不可变的，通过 HashEntry 对象的不变性来降低读操作对加锁的需求；

   > next 域被声明为 final，意味着不能从hash链的中间或尾部添加或删除节点，因为这需要修改 next 引用值，因此所有的节点的修改只能从头部开始。但是对于 remove 操作，需要将要删除节点的前面所有节点整个复制一遍，最后一个节点指向要删除结点的下一个结点。

2. 用 volatile 变量协调读写线程间的内存可见性；

3. 若读时发生指令重排序现象（也就是读到 value 域的值为 null 的时候），则加锁重读；

ConcurrentHashMap 的跨端操作：

比如 size() 计算集合中元素的总个数。首先为了并发性的考虑，ConcurrentHashMap 并没有使用全局计数器，而是分别在每个 segment 中使用一个 volatile 修饰的计数器count，这样当需要更新计数器时，不用锁定整个 ConcurrentHashMap。而 size() 在统计时，是先尝试 RETRIES_BEFORE_LOCK 次（默认是两次）通过不锁住 Segment 的方式来统计各个 Segment 大小，如果统计的过程中，容器的count发生了变化，则再采用对所有 segment 段加锁的方式来统计所有Segment的大小。

#### 区别：

1. 数据结构：JDK7 的数据结构是 Segment数组 + HashEntry数组 + 链表，JDK8 的数据结构是 HashEntry数组 + 链表 + 红黑树，当链表的长度超过8时，链表就会转换成红黑树，从而降低时间复杂度（由O(n) 变成了 O(logN)），提高了效率
2. 锁的实现：JDK7的锁是segment，是基于ReentronLock实现的，包含多个HashEntry；而JDK8 降低了锁的粒度，采用 table 数组元素作为锁，从而实现对每行数据进行加锁，进一步减少并发冲突的概率，并使用 synchronized 来代替 ReentrantLock，因为在低粒度的加锁方式中，synchronized 并不比 ReentrantLock 差，在粗粒度加锁中ReentrantLock 可以通过 Condition 来控制各个低粒度的边界，更加的灵活，而在低粒度中，Condition的优势就没有了。
3. 统计集合中元素个数 size 的方式：JDK7 是先尝试 2次通过不锁住 segment 的方式来统计各个 segment 大小，如果统计的过程中，容器的 count 发生了变化，则再采用加锁的方式来统计所有Segment的大小；在 JDK8 中，对于size的计算，在扩容和 addCount() 方法中就已经有处理了，等到调用 size() 时直接返回元素的个数。

![Map 类图](https://img-blog.csdnimg.cn/20210815044025280.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2E3NDUyMzM3MDA=,size_16,color_FFFFFF,t_70)

摘录自[Java集合篇：HashMap 与 ConcurrentHashMap 原理总结](https://blog.csdn.net/a745233700/article/details/119709104#:~:text=1%E3%80%81ConcurrentHashMap%20%E7%9A%84%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86%EF%BC%9A&text=%E8%BF%99%E4%B8%AA%E7%AE%97%E6%B3%95%E7%9A%84%E5%9F%BA%E6%9C%AC%E6%80%9D%E6%83%B3,%E5%85%B6%E4%BB%96%E7%BA%BF%E7%A8%8B%E4%BF%AE%E6%94%B9%E7%9A%84%E7%BB%93%E6%9E%9C%E3%80%82)。

