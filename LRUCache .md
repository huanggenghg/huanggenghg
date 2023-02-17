## LRUCache 算法

### 一、LRUCache 算法

最近最少使用算法。

维护一个缓存列表，每当一个缓存数据被访问的时候，这个数据就会被提到列表头部，这样尾部数据就是最近最不常使用的了。

### 二、源码

LruCache 通过强引用缓存一定数量的值
cacheSize

最近最少使用算法依赖的数据结构是 LinkedHashMap，LruCache 外部提供放置数据和拿取、移除数据的功能接口，及其他获取命中数量`hitCount`、移除数量的`evictionCount`，提供数据移除后`entryRemoved`的业务功能回调等业务功能的一个类，最近最少使用的算法是在 LinkedHashMap 数据结构之中。**当然，移除旧元素的逻辑是在 LruCache 之中。**

```java
    public void trimToSize(int maxSize) {
        while (true) {
            K key;
            V value;
            synchronized (this) {
                if (size < 0 || (map.isEmpty() && size != 0)) {
                    throw new IllegalStateException(getClass().getName()
                            + ".sizeOf() is reporting inconsistent results!");
                }

                if (size <= maxSize) {
                    break;
                }

                Map.Entry<K, V> toEvict = map.eldest(); // 拿到最近最少使用的那个元素
                if (toEvict == null) {
                    break;
                }

                key = toEvict.getKey();
                value = toEvict.getValue();
                map.remove(key);
                size -= safeSizeOf(key, value);
                evictionCount++;
            }

            entryRemoved(true, key, value, null);
        }
    }
```



而 LinkedHashMap 内部，是由 HashMap 及一个双向链表实现的，在每次访问到元素之后，对双向链表进行维护，也就是最近最少算法的实现，主要实现是 LinkedHashMap 重写了`newNode`、插入、移除、访问元素的方法回调。

```java
    Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) { }
    Node<K,V> replacementNode(Node<K,V> p, Node<K,V> next) { }
    TreeNode<K,V> newTreeNode(int hash, K key, V value, Node<K,V> next) { }
    TreeNode<K,V> replacementTreeNode(Node<K,V> p, Node<K,V> next) { }

	// Callbacks to allow LinkedHashMap post-actions
    void afterNodeAccess(Node<K,V> p) { }
    void afterNodeInsertion(boolean evict) { }
    void afterNodeRemoval(Node<K,V> p) { }
```

故最近最少使用算法，主要逻辑是在这些回调，即插入、移除、访问元素之后进行的，进行链表的维护。

```java
    // 一些关于红黑树树节点的处理，同样的是进行双链表节点的更新，详情可自行查看源码
    Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
        LinkedHashMapEntry<K,V> p =
            new LinkedHashMapEntry<K,V>(hash, key, value, e);
        linkNodeLast(p);
        return p;
    }
    // link at the end of list
    private void linkNodeLast(LinkedHashMapEntry<K,V> p) {
        LinkedHashMapEntry<K,V> last = tail;
        tail = p;
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
    }

	void afterNodeInsertion(boolean evict) { // possibly remove eldest
        LinkedHashMapEntry<K,V> first;
        if (evict && (first = head) != null && removeEldestEntry(first)) {
            K key = first.key;
            removeNode(hash(key), key, null, false, true);
        }
    }

	void afterNodeAccess(Node<K,V> e) { // move node to last
        LinkedHashMapEntry<K,V> last;
        if (accessOrder && (last = tail) != e) {
            LinkedHashMapEntry<K,V> p =
                (LinkedHashMapEntry<K,V>)e, b = p.before, a = p.after;
            p.after = null;
            if (b == null)
                head = a;
            else
                b.after = a;
            if (a != null)
                a.before = b;
            else
                last = b;
            if (last == null)
                head = p;
            else {
                p.before = last;
                last.after = p;
            }
            tail = p;
            ++modCount;
        }
    }

    void afterNodeRemoval(Node<K,V> e) { // unlink
        LinkedHashMapEntry<K,V> p =
            (LinkedHashMapEntry<K,V>)e, b = p.before, a = p.after;
        p.before = p.after = null;
        if (b == null)
            head = a;
        else
            b.after = a;
        if (a == null)
            tail = b;
        else
            a.before = b;
    }
```

### 三、为何需要双向链表呢？

为了删除元素时，链表操作容易。因为你删除中间元素的时候，如果只是一个链表，如 `a -> b -> c`，删除 b 时，还需遍历到 a 使其指向 c，显然有个指向先前元素能更好进行删除的动作。

``` java
    void afterNodeRemoval(Node<K,V> e) { // unlink
        LinkedHashMapEntry<K,V> p =
            (LinkedHashMapEntry<K,V>)e, b = p.before, a = p.after;
        p.before = p.after = null;
        if (b == null)
            head = a;
        else
            b.after = a;
        if (a == null)
            tail = b;
        else
            a.before = b;
    }
```

### 四、DiskLruCache 原理

与 LruCache 类似，除了是操作本地文件————`journal`日志文件

```txt
libcore.io.DiskLruCache
1
100
2

CLEAN 3400330d1dfc7f3f7f4b8d4d803dfcf6 832 21054
DIRTY 335c4c6028171cfddfbaae1a9c313c52
CLEAN 335c4c6028171cfddfbaae1a9c313c52 3934 2342
REMOVE 335c4c6028171cfddfbaae1a9c313c52
DIRTY 1ab96a171faeeee38496d8b330771a7a
CLEAN 1ab96a171faeeee38496d8b330771a7a 1600 234
READ 335c4c6028171cfddfbaae1a9c313c52
```

其中1表示diskCache的版本，100表示应用的版本，2表示一个key对应多少个缓存文件。

四种状态：

- DIRTY 创建或者修改一个缓存的时候，会有一条DIRTY记录，后面会跟一个CLEAN或REMOVE的记录。如果没有CLEAN或REMOVE，对应的缓存文件是无效的，会被删掉
- CLEAN 表示对应的缓存操作成功了，后面会带上缓存文件的大小
- REMOVE 表示对应的缓存被删除了
- READ 表示对应的缓存被访问了，因为LRU需要READ记录来调整缓存的顺序

`journalFile`为缓存操作的所有记录，在操作DiskLruCache过程中，修改内存的缓存记录的同时也会修改硬盘中的日志，这样就是下次冷启动，也可以从日志中恢复。

`lurEntries`是缓存实体在内存中的表示，而`journal`是缓存实体在日志中的表示。

DiskLruCache整体的思想跟LruCache是一样的，也是利用了LinkedHashMap,但是DiskLruCache做了很多保护措施，DiskLruCache在各种对文件的操作上都将读写分开，为了怕写失败等情况，都是先尝试写在一个中间文件，成功再重命名回目标文件。

*详情对应查看源码*