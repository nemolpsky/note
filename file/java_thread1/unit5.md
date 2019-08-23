
## 目录
<!-- TOC -->
- [构造方法](#构造方法)
- [add()方法](#add()方法)
- [get()方法](#get()方法)
- [set()方法](#set()方法)
- [remove()方法](#remove()方法)
- [CopyOnWriteArrayList的弱一致性迭代器](#CopyOnWriteArrayList的弱一致性迭代器)

<!-- /TOC -->

---

## CopyOnWriteArrayList解析
是```Java```并发包中唯一一个并发```List```，它同一时间只允许一个线程修改，但是允许多个线程同时读取，底层的大致原理是写时复制。

---

### 构造方法
- 无参构造方法，可以看到是初始化了一个长度为0的数组，同时把这个数组指向了```array```变量，这个变量是一个使用```volatile```修饰的数组。
  ```
  public CopyOnWriteArrayList() {
      setArray(new Object[0]);
  }

  private transient volatile Object[] array;

  final void setArray(Object[] a) {
      array = a;
  }
  ```
- 接受```List```作为参数的构造方法，这里面是判断传入```List```的类型，然后进行不同的逻辑操作，但是最后都会调用```setArray()```方法，而这个方法也是把数组指向了```array```变量。
  ```
  public CopyOnWriteArrayList(Collection<? extends E> c) {
      Object[] elements;
      if (c.getClass() == CopyOnWriteArrayList.class)
          elements = ((CopyOnWriteArrayList<?>)c).getArray();
      else {
          elements = c.toArray();
          // c.toArray might (incorrectly) not return Object[] (see 6260652)
          if (elements.getClass() != Object[].class)
              elements = Arrays.copyOf(elements, elements.length, Object[].class);
      }
      setArray(elements);
  }

  final void setArray(Object[] a) {
      array = a;
  }
  ```
  
---

### add()方法
```add()```中声明了一个独占锁```lock```，整个添加操作都是被锁住的，所以同一时间只能有一个线程可以操作这个方法。首先调用了```getArray```方法，这其实就是获取那个使用```volatile```声明的```array```数组，也就是存储数值的数组。添加的时候如果超出索引会抛出异常，接着声明了一个新的数组```newElements```，然后把旧的数组里面的值都拷贝过去，然后再把新增的值添加进新数组，最后把```volatile```声明的```array```变量指向这个新数组，这就是写时复制策略。
```
public void add(int index, E element) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        if (index > len || index < 0)
            throw new IndexOutOfBoundsException("Index: "+index+
                                                    ", Size: "+len);
        Object[] newElements;
        int numMoved = len - index;
        if (numMoved == 0)
            newElements = Arrays.copyOf(elements, len + 1);
        else {
            newElements = new Object[len + 1];
            System.arraycopy(elements, 0, newElements, 0, index);
            System.arraycopy(elements, index, newElements, index + 1,
                                 numMoved);
        }
        newElements[index] = element;
        setArray(newElements);
    } finally {
        lock.unlock();
    }
}

final Object[] getArray() {
    return array;
}
```

---

### get()方法
```get()```方法相对来说就是比较简单，没有加锁，获取当前```array```变量指向的那个数组，然后获取对应索引的值。但是因为```get()```方法其实是分成了两步调用，```getArray()```获取当前的数组，```get(Object[] a,int index)```获取数组中的某个值，这并不是一个原子操作，所以会有并发问题，如果A线程刚好执行完```getArray()```方法，然后B线程进来把数组里的值都清空了，接着A线程又继续执行```get(Object[] a,int index)```方法，但是因为A线程已经获取到原来旧数组，所以这个方法里面执行的还是旧数组，这就是所谓的弱一致性问题。
```
public E get(int index) {
    return get(getArray(), index);
}

final Object[] getArray() {
    return array;
}

private E get(Object[] a, int index) {
    return (E) a[index];
}
```

---

### set()方法
```set()```方法其实和add方法很像，也是加锁操作，然后复制整个数组。
```
public E set(int index, E element) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        E oldValue = get(elements, index);

        if (oldValue != element) {
            int len = elements.length;
            Object[] newElements = Arrays.copyOf(elements, len);
            newElements[index] = element;
            setArray(newElements);
        } else {
            // Not quite a no-op; ensures volatile write semantics
            setArray(elements);
        }
        return oldValue;
    } finally {
        lock.unlock();
    }
}
```

---

### remove()方法
同理，```remove()```方法也是差不多的逻辑。
```
public E remove(int index) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        E oldValue = get(elements, index);
        int numMoved = len - index - 1;
        if (numMoved == 0)
            setArray(Arrays.copyOf(elements, len - 1));
        else {
            Object[] newElements = new Object[len - 1];
            System.arraycopy(elements, 0, newElements, 0, index);
            System.arraycopy(elements, index + 1, newElements, index,
                                 numMoved);
            setArray(newElements);
        }
        return oldValue;
    } finally {
        lock.unlock();
    }
}
```

---

### CopyOnWriteArrayList的弱一致性迭代器
```CopyOnWriteArrayList```类自己实现了一个迭代器，这里只截取了一部分源码，注意看```snapshot```这个变量，这个其实就是```CopyOnWriteArrayList```那个名字叫做```array```的数组的快照，所以当获取一个迭代器后它就会获取当前数组的快照，然后就算是其他线程修改了数组的值，也不会影响到这个迭代器里获取到的数组中的值。
```
public Iterator<E> iterator() {
    return new COWIterator<E>(getArray(), 0);
}

static final class COWIterator<E> implements ListIterator<E> {
    /** Snapshot of the array */
    private final Object[] snapshot;
    /** Index of element to be returned by subsequent call to next.  */
    private int cursor;

    private COWIterator(Object[] elements, int initialCursor) {
        cursor = initialCursor;
        snapshot = elements;
    }

    public boolean hasNext() {
        return cursor < snapshot.length;
    }

    public boolean hasPrevious() {
        return cursor > 0;
    }
    ...
}
```
```
static volatile  CopyOnWriteArrayList<Integer> list = new CopyOnWriteArrayList<Integer>();

public static void main(String[] args) throws Exception {
    list.add(1);
    list.add(2);
    list.add(3);

    Thread thread = new Thread(new Runnable()  {
        @Override
        public void run() {
            try {
                TimeUnit.SECONDS.sleep(1);
            }catch (Exception e){
                e.printStackTrace();
            }

            list.set(0,1111);
            list.remove(1);
            list.remove(2);
        }
    });

    thread.start();

    // 子线程睡眠了1秒，所以在修改数据之前就获取到了迭代器，迭代器中就获取到了数组的快照数据
    Iterator<Integer> iterator = list.iterator();

    thread.join();

    // 已经执行完子线程的修改，会发现直接打印数组，和使用先前获取的迭代器打印，输出不一样。
    System.out.println(list);
    while (iterator.hasNext()){
        System.out.println(iterator.next());
    }
}
```