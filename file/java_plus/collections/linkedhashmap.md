## LinkedHashMap底层原理

```LinkedHashMap```是```HashMap```的一个子类，底层实现基本上和```HashMap```一样，只是在原来的单链表的基础上改成了双向链表，这样做的目的是为了让它能够实现插入数据的排序。也就是说如果遍历整个```LinkedHashMap```时，是会按照插入数据的顺序来遍历数据的，除此之外没有别的特殊之处，本文代码是基于1.8版本的，和1.7版本有着较大的不同。

---


1. 构造方法

   前面四个构造方法都是和```HashMap```中一样的，最后一个需要注意下，如果参数```accessOrder```传```true```是，每次```get()```操作都会把get到的那个元素的顺序放到最后，比如依次插入```[{1,1},{2,2},{3,3}]```，正常遍历顺序也是这个，但是如果参数```accessOrder```传```true```，再```get(1)```时，顺序就会变成```[{2,2},{3,3},{1,1}]```。

   ```
   LinkedHashMap(){};
   LinkedHashMap(int initialCapacity){};
   LinkedHashMap(int initialCapacity, float loadFactor){};
   LinkedHashMap(Map<? extends K, ? extends V> m){};
   LinkedHashMap(int initialCapacity,float loadFactor,boolean accessOrder){};
   ```

2. 继承父类的节点

   可以看到```LinkedHashMap```中有一个新的内部类，是继承自```HashMap```中```Node```节点，新增了两个属性，```before```和```after```，这两个属性就是用来维护链表的插入顺序的，此外还有两个节点，头结点和尾节点，也是这个目的，这样就实现了一个双向链表，也就是每个节点上有着多层关系，既有那种存储顺序，也有着插入顺序。

   ```
    // 头结点
    transient LinkedHashMap.Entry<K,V> head;

    // 尾节点
    transient LinkedHashMap.Entry<K,V> tail;

    static class Entry<K,V> extends HashMap.Node<K,V> {
        Entry<K,V> before, after;
        Entry(int hash, K key, V value, Node<K,V> next) {
            super(hash, key, value, next);
        }
    }
   ```

2. put()/get()方法

   查看源码的时候会发现，```LinkedHashMap```连```put()```方法都没有，直接用的父类```HashMap```的方法，而```get()```方法倒是有了，但是也是直接调用的```HashMap```中的查找方法，只是后面调用了一个```afterNodeAccess()```方法。

   ```
    public V get(Object key) {
        Node<K,V> e;
        if ((e = getNode(hash(key), key)) == null)
            return null;
        if (accessOrder)
            afterNodeAccess(e);
        return e.value;
    }
   ```

3. afterNodeAccess()/afterNodeInsertion()/afterNodeRemoval()

   仔细一点会发现，```HashMap```中定义了三个空方法，还有注释表明了是专门给```LinkedHashMap```使用的回调方法，在```LinkedHashMap```中进行重写。

   ```
    // Callbacks to allow LinkedHashMap post-actions
    void afterNodeAccess(Node<K,V> p) { }
    void afterNodeInsertion(boolean evict) { }
    void afterNodeRemoval(Node<K,V> p) { }
   ```

   先来看看```afterNodeInsertion()```这个方法，在```HashMap```中很多地方都有调用，在```put()```方法中也调用过，其中```removeNode()```方法还是```HashMap```里面的，对插入顺序关系进行修改，要注意的是```removeEldestEntry()```是直接返回了```false```，可以对这个方法进行重写，网上有一些文章使用```LinkedHashMap```实现简单的缓存就是需要自己重写这个方法，定义相关的废弃规则。

   ```
    void afterNodeInsertion(boolean evict) { // possibly remove eldest
        LinkedHashMap.Entry<K,V> first;
        if (evict && (first = head) != null && removeEldestEntry(first)) {
            K key = first.key;
            removeNode(hash(key), key, null, false, true);
        }
    }
   ```

   而```afterNodeAccess()```方法则是在```get()```方法之后调用的，可以看到判断了```accessOrder```参数，也就是说在这里如果初始化的时候设置了```accessOrder```参数为```true```，这个方法就会实现上面那种```get```完一个元素后将它的顺序移到最后的效果。

   ```
    void afterNodeAccess(Node<K,V> e) { // move node to last
        LinkedHashMap.Entry<K,V> last;
        if (accessOrder && (last = tail) != e) {
            LinkedHashMap.Entry<K,V> p =
                (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
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
   ```

   ```afterNodeRemoval()```这个方法也是在```HashMap```中```remove()```方法调用的```removeNode()```方法中调用，也是对插入顺序之间关系的修改。

   ```
    void afterNodeRemoval(Node<K,V> e) { // unlink
        LinkedHashMap.Entry<K,V> p =
            (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
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

---

## 总结

- 和```HashMap```是继承关系，是```HashMap```的子类，所以底层原理没有太大的变化。
- 因为是继承关系，底层的实现是在```HashMap```中定义了空方法和调用，```LinkedHashMap```中进行重写，然后核心方法直接调用```HashMap```的即可。
- 在原来的数据结构上改为了双向链表，专门维护插入顺序。
- ```removeEldestEntry()```可以自己进行重写，专门定义自己的废弃规则，从而实现简单的缓存系统。