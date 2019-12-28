## LinkedHashSet底层原理

```LinkedHashSet```是在```HashSet```基础上可以保持插入顺序的一种```Set```集合。


---

1. 底层原理

   是直接继承了```HashSet```，而```HashSet```又是```HashMap```包装了一层，而```LinkedHashMap```是```HashMap```的子类，所以```HashSet```的实现其实是依赖了```LinkedHashMap```。


2. 构造方法

   总共有四个构造方法，会发现其实是调用到了```HashSet```中一个特殊的构造方法中，这个方法初始化的不是```HashMap```而是```LinkedHashMap```。

   ```
    public LinkedHashSet(int initialCapacity, float loadFactor) {
        super(initialCapacity, loadFactor, true);
    }

    public LinkedHashSet(int initialCapacity) {
        super(initialCapacity, .75f, true);
    }

    public LinkedHashSet() {
        super(16, .75f, true);
    }

    public LinkedHashSet(Collection<? extends E> c) {
        super(Math.max(2*c.size(), 11), .75f, true);
        addAll(c);
    }

    HashSet(int initialCapacity, float loadFactor, boolean dummy) {
        map = new LinkedHashMap<>(initialCapacity, loadFactor);
    }
   ```

3. add()/remove()/contains()方法
   
   这三个方法在LinkedHashSet中都没有重写，直接使用的是```HashSet```中的，最终调用到```HashMap```中，而初始化的时候因为初始化的是```LinkedHashMap```，所以底层既可以保证唯一性又可以维护插入的顺序，也是依靠双向链表实现。


---

## 总结

- ```LinkedHashSet```就是在```HashMap、HashSet和LinkedHashMap```的基础上进行了下封装，没有加任何变化。
- 在保证元素唯一性的情况下还可以保证遍历顺序是插入顺序。