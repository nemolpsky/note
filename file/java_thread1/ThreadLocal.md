## ThreadLocal

ThreadLocal用于线程间的隔离，当在多线程情况下声明一个共享变量，而这个变量在每个线程中都会使用到，但是每个线程的变量又不会相互影响，虽然可以每个线程内部都定义这个变量再相互传递，但是这会很麻烦，而ThreadLocal的作用是就是解决这种情况，它会为每个线程都准备一份单独的数据，每个线程之间互不影响。

1. set()方法

   set操作就是调用getMap()方法查找当前对应线程的ThreadLocalMap，有就set操作，没有就创建一个再set。

   ```
    public void set(T value) {
        // 获取当前线程
        Thread t = Thread.currentThread();
        // 调用getMap()方法获取一个ThreadLocalMap
        ThreadLocalMap map = getMap(t);
        // 如果不为空就直接set值，为空则创建一个新的ThreadLocalMap
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }

    // 为当前线程声明一个ThreadLocalMap
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
   ```

2. remove()方法

   remove操作也是调用ThreadLocalMap方法来清空线程对应的ThreadLocalMap里的值。

   ```
     public void remove() {
         获取当前线程的ThreadLocalMap，如果存在就清空里面对应的值
         ThreadLocalMap m = getMap(Thread.currentThread());
         if (m != null)
             m.remove(this);
     }
   ```
3. get()方法

   get操作稍稍复杂一点，调用getMap()方法来获取线程对应的ThreadLocalMap进行get操作。

   ```
    public T get() {
        // 获取当前线程
        Thread t = Thread.currentThread();
        // 调用getMap()方法获取一个ThreadLocalMap
        ThreadLocalMap map = getMap(t);
        // map不为空则获取里面对应的值，为空则调用初始化方法
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }

    private T setInitialValue() {
        // 初始化为null
        T value = initialValue();
        // 调用getMap()方法获取一个ThreadLocalMap
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        // 为空则set一个null值，不为空初始化再set一个null值
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }
   ```

4. getMap()/createMap()方法
   
   上面三个操作发现基本都是调用getMap()/createMap()来获取一个ThreadLocalMap，而其实这两个方法是对Thread内部的一个threadLocals进行操作。在Thread的源码中会发现threadLocals其实是定义在ThreadLocal中的一个ThreadLocalMap类型。

   ```
    ThreadLocal.ThreadLocalMap threadLocals = null;

    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
   
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }

   ```

5. ThreadLocalMap类

   仔细看源码，就是一个弱化版的HashMap，底层是数组，有初始值，有容量计算。

   ```
    static class ThreadLocalMap {
        
        // 内部节点类，用于存储值
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;
            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }

        // 初始大小
        private static final int INITIAL_CAPACITY = 16;
        // 存储节点的数组
        private Entry[] table;
        // 计算容量
        private void setThreshold(int len) {
            threshold = len * 2 / 3;
        }

        // 构造方法
        ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
            table = new Entry[INITIAL_CAPACITY];
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
            table[i] = new Entry(firstKey, firstValue);
            size = 1;
            setThreshold(INITIAL_CAPACITY);
        }
    }
   ```
   
   set()方法
   
   ```
        private void set(ThreadLocal<?> key, Object value) {

            Entry[] tab = table;
            int len = tab.length;
            // key是操作ThreadLocal的线程，根据key计算哈希值算出，存储数据的位置
            int i = key.threadLocalHashCode & (len-1);

            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();

                if (k == key) {
                    e.value = value;
                    return;
                }

                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
   ```
   get()方法

   ```
    private Entry getEntry(ThreadLocal<?> key) {
        // 计算哈希值，算出数据存储位置
        int i = key.threadLocalHashCode & (table.length - 1);
        Entry e = table[i];
        if (e != null && e.get() == key)
            return e;
        else
            return getEntryAfterMiss(key, i, e);
    }

   ```
   
   remove()方法

   ```
    private void remove(ThreadLocal<?> key) {
        Entry[] tab = table;
        int len = tab.length;
        // 计算哈希值
        int i = key.threadLocalHashCode & (len-1);
        for (Entry e = tab[i];
            e != null;
            e = tab[i = nextIndex(i, len)]) {
            if (e.get() == key) {
                // 清理数据
                e.clear();
                expungeStaleEntry(i);
                return;
            }
        }
    }
   ```

6. ThreadLocal的实际应用

   在Spring中其实就大量运用到ThreadLocal，比如创建Bean的时候可以声明singleton作用域，也就是单例模式，这样可以避免对象的重复创建，但是实际中单例模式也没有出现任何并发问题，就是运用的ThreadLocal来解决这个问题的。


---

## 总结

- ThreadLocal可以保证多个线程读写同一个变量而没有并发问题。
- ThreadLocal底层其实是有一个类似于HashMap的ThreadLocalMap内部类，而线程的源码中实现定义了这样一个map的引用，每个线程对ThreadLocal的操作其实就是对它自己的ThreadLocalMap进行操作，所以数据其实都是放在这个map中，也就是每个线程各有一份自己的数据，所以不会有并发问题。
- 如果线程使用完之后不再需要ThreadLocal类了，最好手动调用remove()方法，避免线程销毁了，但是因为ThreadLocal还存在，所以被销毁线程对应的数据没有被回收。
- 使用ThreadLocal的时候最后声明为就静态的，因为是所有线程共享一个ThreadLocal的变量，声明为静态的只需要第一次加载类的时候分配一块存储空间即可。