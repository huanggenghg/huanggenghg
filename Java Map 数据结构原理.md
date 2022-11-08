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

#### 四、ConcurrentHashMap


