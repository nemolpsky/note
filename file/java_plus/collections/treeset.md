## TreeSet底层原理

顾名思义，```TreeSet```在保证元素唯一性的基础上，还可以对元素进行排序。

---

1. 底层原理

   底层是基于```TreeMap```来实现的，所以底层结构也是红黑树，因为他和```HashSet```不同的是不需要重写```hashCode()```和```equals()```方法，因为它去重是依靠比较器来去重，因为结构是红黑树，所以每次插入都会遍历比较来寻找节点插入位置，如果发现某个节点的值是一样的那就会直接覆盖。


2. 构造方法

   可以看到构造方法是就是创建一个```TreeMap```，然后指向```NavigableMap```类型的引用。

   ```
    private transient NavigableMap<E,Object> m;
     
    TreeSet(NavigableMap<E,Object> m) {
        this.m = m;
    }

    public TreeSet() {
        this(new TreeMap<E,Object>());
    }

    public TreeSet(Comparator<? super E> comparator) {
        this(new TreeMap<>(comparator));
    }

    public TreeSet(Collection<? extends E> c) {
        this();
        addAll(c);
    }

    public TreeSet(SortedSet<E> s) {
        this(s.comparator());
        addAll(s);
    }
   ```

3. add()/remove()/contains()方法

   可以看到这三个方法都是调用到```NavigableMap```类型的引用上，而因为初始化的时候是初始化的```TreeMap```，所以最终全部调用到```TreeMap```上的方法。

   ```
    public boolean add(E e) {
        return m.put(e, PRESENT)==null;
    }

    public boolean remove(Object o) {
        return m.remove(o)==PRESENT;
    }

    public boolean contains(Object o) {
        return m.containsKey(o);
    }
   ```

---

## 总结

- 底层是在```TreeMap```的基础上进行封装，所以结构是红黑树。
- 因为是红黑树结构，所以不需要重写```hashCode()```和```equals()```方法来保证唯一性。
- ```TreeMap```有的特性```TreeSet```都有，还在这个基础上增加了保证元素唯一性的特点。