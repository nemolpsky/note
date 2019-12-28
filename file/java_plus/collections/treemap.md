## TreeMap底层原理

```TreeMap```继承了```AbstractMap```，实现了```NavigableMap```接口，底层是用红黑树实现的，也正是因为这个原因它也可以对键进行排序，而且插入、查找和删除的时间复杂度都是O(logn)，相当快。

---

1. 底层数据结构

   一个```Entry```内部类实现了红黑树的结构。

   ```
    static final class Entry<K,V> implements Map.Entry<K,V> {
        K key;
        V value;
        Entry<K,V> left;
        Entry<K,V> right;
        Entry<K,V> parent;
        boolean color = BLACK;

        /**
         * Make a new cell with given key, value, and parent, and with
         * {@code null} child links, and BLACK color.
         */
        Entry(K key, V value, Entry<K,V> parent) {
            this.key = key;
            this.value = value;
            this.parent = parent;
        }

        /**
         * Returns the key.
         *
         * @return the key
         */
        public K getKey() {
            return key;
        }

        /**
         * Returns the value associated with the key.
         *
         * @return the value associated with the key
         */
        public V getValue() {
            return value;
        }

        /**
         * Replaces the value currently associated with the key with the given
         * value.
         *
         * @return the value associated with the key before this method was
         *         called
         */
        public V setValue(V value) {
            V oldValue = this.value;
            this.value = value;
            return oldValue;
        }

        public boolean equals(Object o) {
            if (!(o instanceof Map.Entry))
                return false;
            Map.Entry<?,?> e = (Map.Entry<?,?>)o;

            return valEquals(key,e.getKey()) && valEquals(value,e.getValue());
        }

        public int hashCode() {
            int keyHash = (key==null ? 0 : key.hashCode());
            int valueHash = (value==null ? 0 : value.hashCode());
            return keyHash ^ valueHash;
        }

        public String toString() {
            return key + "=" + value;
        }
    }

   ```

2. 构造方法

   ```TreeMap```也有四个构造方法，第一个无参默认的，第二是自己设置排序规则，第三个是创建的时候设置另一个map的值进来，最后一个比较特殊，接受的```SortedMap```类型，其实```TreeMap```就是```SortedMap```类型，上面讲了实现了```NavigableMap```接口，而```NavigableMap```接口就继承了```SortedMap```接口，这个方法的作用可以直接接收一个```TreeMap```集合，然后不仅连值也在初始化的时候放入了，排序规则也一并设置了。

   ```
   TreeMap() {}
   TreeMap(Comparator<? super K> comparator) {}
   TreeMap(Map<? extends K, ? extends V> m) {}
   TreeMap(SortedMap<K, ? extends V> m) {}
   ```

3. put()方法

   添加其实就是遍历树的操作，根据排序器的规则来遍历，然后找到对应的位置插入数据。

   ```
    public V put(K key, V value) {
        // 获取根节点
        Entry<K,V> t = root;
        // 根节点为空，插入的元素就作为根节点
        if (t == null) {
            compare(key, key); // type (and possibly null) check

            root = new Entry<>(key, value, null);
            size = 1;
            modCount++;
            return null;
        }
        int cmp;
        Entry<K,V> parent;
        // split comparator and comparable paths
        Comparator<? super K> cpr = comparator;
        // 排序器不为空
        if (cpr != null) {
            // 根据排序规则遍历寻找插入位置，替换key的value
            do {
                parent = t;
                cmp = cpr.compare(key, t.key);
                if (cmp < 0)
                    t = t.left;
                else if (cmp > 0)
                    t = t.right;
                else
                    return t.setValue(value);
            } while (t != null);
        }
        // 没有排序器
        else {
            if (key == null)
                throw new NullPointerException();
            @SuppressWarnings("unchecked")
                Comparable<? super K> k = (Comparable<? super K>) key;
            // 使用自然排序来寻找插入位置，替换key的value
            do {
                parent = t;
                cmp = k.compareTo(t.key);
                if (cmp < 0)
                    t = t.left;
                else if (cmp > 0)
                    t = t.right;
                else
                    return t.setValue(value);
            } while (t != null);
        }
        // 还没有建立该key的节点，插入新节点，设置父节点
        Entry<K,V> e = new Entry<>(key, value, parent);
        if (cmp < 0)
            parent.left = e;
        else
            parent.right = e;
        // 插入之后需要判断是否矫正红黑树的结构
        fixAfterInsertion(e);
        size++;
        modCount++;
        return null;
    }
   ```

4. get()方法

   查找方法其实就是上面方法的简化版，按排序规则往下查找即可。

   ```
    public V get(Object key) {
        Entry<K,V> p = getEntry(key);
        return (p==null ? null : p.value);
    }

    final Entry<K,V> getEntry(Object key) {
        // Offload comparator-based version for sake of performance
        if (comparator != null)
            return getEntryUsingComparator(key);
        if (key == null)
            throw new NullPointerException();
        @SuppressWarnings("unchecked")
            Comparable<? super K> k = (Comparable<? super K>) key;
        Entry<K,V> p = root;
        while (p != null) {
            int cmp = k.compareTo(p.key);
            if (cmp < 0)
                p = p.left;
            else if (cmp > 0)
                p = p.right;
            else
                return p;
        }
        return null;
    }
   ```

5. remove()方法

   删除元素要稍微复杂点，因为要重新调整树的结构，规则是按红黑树的组成规则调整节点。

   ```
    public V remove(Object key) {
        Entry<K,V> p = getEntry(key);
        if (p == null)
            return null;

        V oldValue = p.value;
        deleteEntry(p);
        return oldValue;
    }

    private void deleteEntry(Entry<K,V> p) {
        modCount++;
        size--;

        // If strictly internal, copy successor's element to p and then make p
        // point to successor.
        // 被删除节点的左子节点和右子节点都不为空，那么就用被删除节点的中序后继节点代替被删除节点节点
        if (p.left != null && p.right != null) {
            Entry<K,V> s = successor(p);
            p.key = s.key;
            p.value = s.value;
            p = s;
        } // p has 2 children

        // Start fixup at replacement node, if it exists.
        //replacement为替代节点，如果被删除的左子节点存在那么就用左子节点替代，否则用右子节点替代
        Entry<K,V> replacement = (p.left != null ? p.left : p.right);

        // 找到代替节点，就用它代替被删除的节点
        if (replacement != null) {
            // Link replacement to parent
            replacement.parent = p.parent;
            if (p.parent == null)
                root = replacement;
            else if (p == p.parent.left)
                p.parent.left  = replacement;
            else
                p.parent.right = replacement;

            // Null out links so they are OK to use by fixAfterDeletion.
            p.left = p.right = p.parent = null;

            // Fix replacement
            if (p.color == BLACK)
                fixAfterDeletion(replacement);
        } 
        // 没找到就表示整个树空了，根节点置空
        else if (p.parent == null) { // return if we are the only node.
            root = null;
        }
        // 没有子节点，直接删除当前节点
        else { //  No children. Use self as phantom replacement and unlink.
            if (p.color == BLACK)
                fixAfterDeletion(p);

            if (p.parent != null) {
                if (p == p.parent.left)
                    p.parent.left = null;
                else if (p == p.parent.right)
                    p.parent.right = null;
                p.parent = null;
            }
        }
    }
   ```

6. 遍历

   在```TreeMap```中遍历只能用增加```for()```循环来操作，但是其实最后编译器是会转变成迭代器操作。
   
   ```
   for(Object key : map.keySet()) {}
   for(Map.Entry entry : map.entrySet()) {}

   for(Iterator<Map.Entry<Object, Object>> it = map.entrySet().iterator() ; map.hasNext();) {
      Entry<Integer, String> entry = it.next();
   }
   ```

   以```EntryIterator```为例子，可以看到```next()```方法是调用```nextEntry()```方法，也就是查找下个元素。最后会调用到```successor()```方法中，其实这个方法就是中序遍历，也就是先遍历到最左边的子节点，然后再开始往回走，再继续往右走，知道整棵树都遍历完，而因为红黑树任意节点的左子节点都是比它小，右子节点比它大，所以这样遍历刚好就是按照顺序遍历的。

   ```
    final class EntryIterator extends PrivateEntryIterator<Map.Entry<K,V>> {
        EntryIterator(Entry<K,V> first) {
            super(first);
        }
        public Map.Entry<K,V> next() {
            return nextEntry();
        }
    }

    final Entry<K,V> nextEntry() {
        Entry<K,V> e = next;
        if (e == null)
            throw new NoSuchElementException();
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        next = successor(e);
        lastReturned = e;
        return e;
    }

    static <K,V> TreeMap.Entry<K,V> successor(Entry<K,V> t) {
        if (t == null)
            return null;
        // 有右子节点的节点，后继节点就是右子节点的最左节点，因为最左节点是右子树的最小节点
        else if (t.right != null) {
            Entry<K,V> p = t.right;
            while (p.left != null)
                p = p.left;
            return p;
        } 
        // 如果右子树为空，则寻找当前节点所在左子树的第一个祖先节点，然后查找右子节点
        else {
            Entry<K,V> p = t.parent;
            Entry<K,V> ch = t;
            while (p != null && ch == p.right) {
                ch = p;
                p = p.parent;
            }
            return p;
        }
    }
   ```

7. firstKey()/firstEntry()/lastKey()/lastEntry()方法

   这四个方法分别是获取最小的```key```，最小的键值对对象，最大的```key```和最大的键值对对象，因为红黑树的结构是最小的值在最左边的叶子上，而最大的值则在最右边的叶子上，所以查找最小就是遍历到最左，反之就是遍历到最右。

   ```
    public K firstKey() {
        return key(getFirstEntry());
    }

    static <K> K key(Entry<K,?> e) {
        if (e==null)
            throw new NoSuchElementException();
        return e.key;
    }

    final Entry<K,V> getFirstEntry() {
        Entry<K,V> p = root;
        if (p != null)
            while (p.left != null)
                p = p.left;
        return p;
    }

    public K lastKey() {
        return key(getLastEntry());
    }

    final Entry<K,V> getLastEntry() {
        Entry<K,V> p = root;
        if (p != null)
            while (p.right != null)
                p = p.right;
        return p;
    }

   ```

8. lowerKey()/lowerEntry()/higherKey()/higherEntry()方法

   这四个方法更上面的和相似，只不过是限制了最大和最小值，比如第一个是获取小于指定```key```的最大```key```，第二是获取第一个的键值对对象，第三个是获取大于指定```key```的最小```key```，第四个是获取第三个的键值对对象，其实就是对红黑树的遍历，依赖于它任意节点的左子节点一定小于它，而右子节点大于它的特性，知道这个就好理解了。

   ```
    public K lowerKey(K key) {
        return keyOrNull(getLowerEntry(key));
    }

    static <K,V> K keyOrNull(TreeMap.Entry<K,V> e) {
        return (e == null) ? null : e.key;
    }

    final Entry<K,V> getLowerEntry(K key) {
        // 从根节点遍历
        Entry<K,V> p = root;
        while (p != null) {
            int cmp = compare(key, p.key);
            // key比根节点大，找右节点
            if (cmp > 0) {
                if (p.right != null)
                    p = p.right;
                else
                    return p;
            }
            // key比根节点小，找左节点
            else {
                // 左节点不为空表示找到了
                if (p.left != null) {
                    p = p.left;
                } 
                // 左节点为空需要找它的父节点
                else {
                    Entry<K,V> parent = p.parent;
                    Entry<K,V> ch = p;
                    while (parent != null && ch == parent.left) {
                        ch = parent;
                        parent = parent.parent;
                    }
                    return parent;
                }
            }
        }
        return null;
    }

    public K higherKey(K key) {
        return keyOrNull(getHigherEntry(key));
    }

    final Entry<K,V> getHigherEntry(K key) {
        Entry<K,V> p = root;
        while (p != null) {
            int cmp = compare(key, p.key);
            // key比根节点小，找左节点
            if (cmp < 0) {
                if (p.left != null)
                    p = p.left;
                else
                    return p;
            } 
            // key比根节点大，找右节点
            else {
                // 右节点不为空，找到目标值
                if (p.right != null) {
                    p = p.right;
                } 
                // 右节点为空，找它的父节点
                else {
                    Entry<K,V> parent = p.parent;
                    Entry<K,V> ch = p;
                    while (parent != null && ch == parent.right) {
                        ch = parent;
                        parent = parent.parent;
                    }
                    return parent;
                }
            }
        }
        return null;
    }

   ```

---

## 总结

- ```TreeMap```的底层结构是红黑树，因此可以依赖红黑树的特性保证```key```有序，所有操作其实就是对红黑树的操作。
- 而因为红黑树的数据结构，所以插入、查询和删除时间复杂度都是O(logn)，但是删除和插入后可能需要调整树结构满足红黑树的规则，需要耗费性能。
- ```TreeMap```的效率很高，还支持各种条件查找，甚至是范围查找和范围替换等等。