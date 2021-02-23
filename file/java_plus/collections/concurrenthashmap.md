## ConcurrentHashMap底层原理

```ConcurrentHashMap```可能会用的相对比较少，因为它跟```HashMap```其实功能非常相似，但是它是并发安全的，而且1.7和1.8版本中的变化比较大。

---
### 1.7版本

1. 底层结构

   1.7版本的底层数据结构比较特殊，像是类似于嵌套```Map```一样，首先整个结构是有一个```Segment```数组，```Segment```是一个键值对形式的结构，然后内部存储的是```HashEntry```数组，这里面才真正的存放数值，而每次加锁的时候会锁住不同操作的那个```Segment```，也就是说如果操作的不是同一个，那肯定就不会进行加锁操作，默认会有16个```Segment```。

   ```
   static final class Segment<K,V> extends ReentrantLock implements Serializable {
        Segment(float lf, int threshold, HashEntry<K,V>[] tab) {
            this.loadFactor = lf;
            this.threshold = threshold;
            this.table = tab;
        }
        ...
   }

   static final class HashEntry<K,V> {
      final int hash;
      final K key;
      volatile V value;
      volatile HashEntry<K,V> next;
      ...
   }
   ```

   ![1.7](https://github.com/nemolpsky/note/raw/master/file/java_plus/collections/images/concurrenthashmap1.jpg)
   

2. 构造方法

   总共有好几个构造方法，但是最终都是调用这个构造方法，可以看到就是根据设置的值来创建```Segment```数组和```Segment```以及```HashEntry```数组。

   ```
    // 最大分段数量
    static final int MAX_SEGMENTS = 1 << 16;
    // 最大容量
    static final int MAXIMUM_CAPACITY = 1 << 30;
    // 默认分段数量
    static final int DEFAULT_CONCURRENCY_LEVEL = 16;

    @SuppressWarnings("unchecked")
    public ConcurrentHashMap(int initialCapacity, float loadFactor, int concurrencyLevel) {
        if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
            throw new IllegalArgumentException();
        // 限制最大分段数量
        if (concurrencyLevel > MAX_SEGMENTS)
            concurrencyLevel = MAX_SEGMENTS;
        // Find power-of-two sizes best matching arguments
        int sshift = 0;
        // ssize是计算出Segment的容量
        int ssize = 1;
        while (ssize < concurrencyLevel) {
            ++sshift;
            ssize <<= 1;
        }

        // 这两个变量在定位segment时会用到
        this.segmentShift = 32 - sshift;
        this.segmentMask = ssize - 1;

        // 计算segment中每个HashEntry数组的容量
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        int c = initialCapacity / ssize;
        if (c * ssize < initialCapacity)
            ++c;
        int cap = MIN_SEGMENT_TABLE_CAPACITY;
        while (cap < c)
            cap <<= 1;
        // create segments and segments[0]
        //创建segments数组并初始化第一个Segment，其余的Segment延迟初始化
        Segment<K,V> s0 =
            new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
                             (HashEntry<K,V>[])new HashEntry[cap]);
        Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];
        UNSAFE.putOrderedObject(ss, SBASE, s0); // ordered write of segments[0]
        this.segments = ss;
    }
   ```

3. put()方法

   其实步骤就是先根据初始化定义的两个变量来计算要放到哪个```Segment```，然后加锁，再定位到```Segment```里面的```HashEntry```数组```key```的插入位置，最后再判断是直接插入还是链表追加，加锁的方式是自旋尝试加锁。

   ```
    @SuppressWarnings("unchecked")
    public V put(K key, V value) {
        Segment<K,V> s;
        if (value == null)
            throw new NullPointerException();
        int hash = hash(key);
        // 使用这两个初始化定义的全局变量定位segment，返回的hash值无符号右移segmentShift位与段掩码进行位运算
        int j = (hash >>> segmentShift) & segmentMask;
        if ((s = (Segment<K,V>)UNSAFE.getObject          // nonvolatile; recheck
             (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
            s = ensureSegment(j);
        // 调用对应的Segment的put()方法
        return s.put(key, hash, value, false);
    }

    final V put(K key, int hash, V value, boolean onlyIfAbsent) {
        // 因为Segment继承ReentrantLock，使用tryLock()尝试获取独占锁
        HashEntry<K,V> node = tryLock() ? null : scanAndLockForPut(key, hash, value);
        // 返回被替换的旧值
        V oldValue;
        // 放入数据
        try {
            // Segment中的HashEntry数组
            HashEntry<K,V>[] tab = table;
            int index = (tab.length - 1) & hash;
            // 找到数组中对应key的第一个元素
            HashEntry<K,V> first = entryAt(tab, index);
            for (HashEntry<K,V> e = first;;) {
                // 第一个元素不为空，直接在后面插入，形成链表
                if (e != null) {
                    K k;
                    if ((k = e.key) == key ||
                        (e.hash == hash && key.equals(k))) {
                        oldValue = e.value;
                        if (!onlyIfAbsent) {
                            e.value = value;
                            ++modCount;
                        }
                        break;
                    }
                    e = e.next;
                }
                // scanAndLockForPut()操作中只有对应位置没有元素才创建node
                // 第一个元素为空，直接把这个node插入
                else {
                    if (node != null)
                        node.setNext(first);
                    else
                        node = new HashEntry<K,V>(hash, key, value, first);
                    int c = count + 1;
                    if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                        rehash(node);
                    else
                        setEntryAt(tab, index, node);
                    ++modCount;
                    count = c;
                    oldValue = null;
                    break;
                }
            }
        } finally {
            // 解锁
            unlock();
        }
        return oldValue;
    }

    private HashEntry<K,V> scanAndLockForPut(K key, int hash, V value) {
        // 获取HashEntry数组上对应key的第一个元素
        HashEntry<K,V> first = entryForHash(this, hash);
        HashEntry<K,V> e = first;
        HashEntry<K,V> node = null;
        int retries = -1; // negative while locating node
        // 自旋获取锁
        while (!tryLock()) {
            HashEntry<K,V> f; // to recheck first below
            if (retries < 0) {
                // 为空就创建一个新节点
                if (e == null) {
                    if (node == null) // speculatively create node
                        node = new HashEntry<K,V>(hash, key, value, null);
                    retries = 0;
                }
                else if (key.equals(e.key))
                    retries = 0;
                else
                    e = e.next;
            }
            // 超过自旋次数，直接锁住
            else if (++retries > MAX_SCAN_RETRIES) {
                lock();
                break;
            }
            // 头节点发生变化，重新遍历
            else if ((retries & 1) == 0 && (f = entryForHash(this, hash)) != first) {
                e = first = f; // re-traverse if entry changed
                retries = -1;
            }
        }
        return node;
    }
   ```

4. get()方法

   ```get()```方法中并没有用锁，而是使用了```UNSAFE.getObjectVolatile()```来获取，这是一个操作硬件级别的并发类，而这个方法是保证了```Volatile```语义，也就是取的时候一定会取到最新的值，所以不需要加锁。

   ```
    public V get(Object key) {
        Segment<K,V> s; // manually integrate access methods to reduce overhead
        HashEntry<K,V>[] tab;
        int h = hash(key);
        // 计算segment位置
        long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
        // 计算HashEntry数组位置中的key位置
        if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
            (tab = s.table) != null) {
            // 循环链表
            for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
                     (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
                 e != null; e = e.next) {
                K k;
                if ((k = e.key) == key || (e.hash == h && key.equals(k)))
                    return e.value;
            }
        }
        return null;
    }
   ```

---

### 1.8版本


1. 底层结构

   在1.8版本的时候数据结构又改回和```HashMap```类似的数组加链表，链表过长转化为红黑树，也不再使用```ReentrantLock```来保证并发，而是使用```synchronized```关键字和```CAS```操作保证并发问题，因为1.6开始```synchronized```锁做了优化，比如偏向锁、轻量锁和重量锁的升级，所以性能并不一定就差，但是还是保留了```Segment```来进行兼容。

   而且可以看到相较于```HashMap```它的内部节点类里的属性使用```volatile```来声明。

   ```
    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        volatile V val;
        volatile Node<K,V> next;
    }
   ```

   ![1.8](https://github.com/nemolpsky/note/raw/master/file/java_plus/collections/images/concurrenthashmap2.jpg)

2. 构造方法

   可以看到除了为了兼容1.7版本而保留的构造方法之外，其他的构造方法都和```HashMap```差不多。

   ```
    public ConcurrentHashMap(int initialCapacity) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException();
        int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
                   MAXIMUM_CAPACITY :
                   tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
        this.sizeCtl = cap;
    }

    public ConcurrentHashMap(int initialCapacity, float loadFactor) {
        this(initialCapacity, loadFactor, 1);
    }

    public ConcurrentHashMap(int initialCapacity, float loadFactor, int concurrencyLevel) {
        if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
            throw new IllegalArgumentException();
        if (initialCapacity < concurrencyLevel)   // Use at least as many bins
            initialCapacity = concurrencyLevel;   // as estimated threads
        long size = (long)(1.0 + (long)initialCapacity / loadFactor);
        int cap = (size >= (long)MAXIMUM_CAPACITY) ?
            MAXIMUM_CAPACITY : tableSizeFor((int)size);
        this.sizeCtl = cap;
    }


   ```

3. put()方法

   首先是一个循环尝试插入，如果插入的位置上是空的，直接```CAS```操作插入，如果有值则加```synchronized```锁，然后修改值，注意的是只锁住了单个索引上的链表或红黑树，也就是说除非是插入操作遇到了哈希碰撞才有加锁操作，否则是不会有加锁操作的，因为不同位置上的值的操作根本不可能有并发问题的，所以不需要加锁。

   ```
    public V put(K key, V value) {
        return putVal(key, value, false);
    }

    final V putVal(K key, V value, boolean onlyIfAbsent) {
        // key和value不能为空
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode());
        int binCount = 0;
        // 循环知道成功为止
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            // 数组为空，初始化
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
            // 当前位置还没有元素，使用Unsafe类的CAS操作放入数据，成功了就跳出循环
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            // 扩容
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            // 这里会加锁，但是注意锁住了链表或者红黑树，也就是说只单独锁住了数组中这个索引，对于其他的索引位置上的put和get都没有影响
            else {
                V oldVal = null;
                // 锁住时候就是查找位置，判断数据结构进行插入
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) {
                            binCount = 1;
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }
                if (binCount != 0) {
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        addCount(1L, binCount);
        return null;
    }
   ```

4. get()方法

   整个方法就是直接进行哈希计算查找对应位置上的数据，因为前面说了```Node```节点里的字段都是用```volatile```修饰，所以取的时候一定会取到最新的值。

   ```
    public V get(Object key) {
        Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
        int h = spread(key.hashCode());
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (e = tabAt(tab, (n - 1) & h)) != null) {
            if ((eh = e.hash) == h) {
                if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                    return e.val;
            }
            else if (eh < 0)
                return (p = e.find(h, key)) != null ? p.val : null;
            while ((e = e.next) != null) {
                if (e.hash == h &&
                    ((ek = e.key) == key || (ek != null && key.equals(ek))))
                    return e.val;
            }
        }
        return null;
    }
   ```

---


## 总结
- 线程安全，1.8版本的性能会比1.7好很多，大体原理和```HashMap```相比还是没有太多变化。
- 1.7版本使用的分段锁，1.8版本则更像是基于```HashMap```使用```synchronized```和```CAS```操作来保证并发问题，而且只锁单个索引位置，性能更好。
- ```ConcurrentHashMap```是弱一致性的，也就是有些迭代器的方法可能没办法获取最新值，这也是为了性能而舍弃强一致性，不然只能像```Hashtable```那样直接锁住全部操作了。
   