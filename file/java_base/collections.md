## List、Set和Map的区别？
- List是保证取出的顺序是和存入顺序一致的，而且允许有重复元素
- Set则是保证了元素的唯一性
- Map则是使用key-value的形式来维护数据

---

## RandomAccess接口
只是起一个标识作用，凡是实现了这个接口的集合都具有随机访问的功能。在Collections类的binarySearch()方法中，它要判断传入的list是否RamdomAccess的实例，如果是则调用indexedBinarySearch()，如果不是就调用iteratorBinarySearch()方法。
- 支持随机访问的集合

  优先使用普通get(i)模式的for循环，因为时间复杂度是O(1)
- 不支持随机访问的集合

  优先使用iterator迭代器或者forEach方法，因为如果使用普通的for循环每次查找都是从头开始，尤其是数据量特别大的时候，特别慢

---

## List集合
1. ArrayList
   - 非线程安全
   - 底层是Object数据实现，所以支持随机访问
   - 插入数据时间复杂度受插入位置的影响，末尾添加就是O(1)，头部添加就是O(n)，因为后面所有的元素的索引都要往后移动
2. LinkedList
   - 非线程安全
   - 底层是双向链表实现，不支持随机访问
   - 所有的插入操作时间复杂度都是O(1)，读取数据因为要遍历整条链表所以是O(n)

3. Vector
   - 线程安全，所有的方法都是使用synchronized同步保证线程安全
   - 很陈旧的类，不推荐使用

--- 

## Map集合
1. HashMap
   - 非线程安全，多线程下可能会在扩容的时候造成底层链表的死循环
   - 可以有一个空值key，多个空值value
   - 默认初始值16，每次扩充两倍，使用百分之75%的容量之后开始扩充
   - 1.8前底层是数组+链表，也就是当散列冲突之后，同一个位置使用链表存储冲突的多个元素，1.8后进行了优化，如果链表长度超过8之后表示冲突非常严重就把链表转换为红黑树，以便快速查询
2. Hashtable(没有实现Map)
   - 线程安全，使用synchronized修饰
   - key和value都不支持空值
   - 很陈旧的类，不推荐使用
3. ConcurrentHashMap
   - 线程安全，底层实现和HashMap类似，1.8前用分段锁实现线程安全，1.8后用synchronized和CAS实现
   - 1.8前是有一个Segment数组，每组有各自的锁保证线程安全，可以同事操作不同组数据
   - 1.8后synchronized是只锁住链表或红黑树首节点，如果没有哈希冲突就不会锁住，而是使用CAS保证线程安全
4. LinkedHashMap
   - 继承HashMap，底层结构也是相同的
   - 额外增加了一条双向链表保持插入时的顺序
5. TreeMap
   - 底层是红黑树结构
   - 可以根据key和value的大小排序

---

## Set集合
1. HashSet
   - 非线程安全
   - 底层基于HashMap实现
   - 保证元素唯一性，先检查hascode值，如果一样就再调用equals二次检查
2. LinkedHashSet
   - 继承于HashSet，内部是基于LinkedHashMap实现。

3. TreeSet
   -元素是有序的，底层结构是红黑树

