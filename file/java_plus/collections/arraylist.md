## ArrayList底层原理

```ArrayList```是最常用的集合，底层是用数组实现的，继承```AbstractList```类，实现了```List```，```RandomAccess```，```Cloneable```和```Serializable```接口，非线程安全。

---

1. 初始化

   有两个构造方法，一个带参数的是自己指定大小，一个不带参数的则是默认大小为10，可以看到如果是使用默认构造方法的话，不会立刻初始化一个大小为10的数组，而是指向了一个空数组，这是为了避免初始化之后又没插入数据而带来的空间浪费。

   ```
    // 默认大小
    private static final int DEFAULT_CAPACITY = 10; 
    // 空数组
    private static final Object[] EMPTY_ELEMENTDATA = {};
    // 默认构造方法使用的空数组
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
     
    // 真实存放数据的数组 
    transient Object[] elementData;

    // 指定大小的构造方法
    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            // 按照参数初始化数组
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            // 设置空数组
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }

    // 默认构造方法
    public ArrayList() {
        // 初始化为空的数组
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }

   ```

2. add()方法

   这个方法是直接往list里面添加元素，可以看到它会判断数组是否初始化了，如果没有这个时候才会真正的初始化数组，然后直接把元素放到数组中，时间复杂度是O(1)。

   ```
    // 在数组最后面添加一个元素
    public boolean add(E e) {
        // 检查容量
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }

    // 检查容量是否足够的方法
    private void ensureCapacityInternal(int minCapacity) {
        // 判断是否需要初始化数组
        ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
    }

    // 判断是否需要初始化数组
    private static int calculateCapacity(Object[] elementData, int minCapacity) {
        // 如果数组是空的，创建一个长度10的数组
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            return Math.max(DEFAULT_CAPACITY, minCapacity);
        }
        // 不为空直接返回当前的长度
        return minCapacity;
    }

    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        // 添加后的长度要比数组大，需要调用grow()方法扩容
        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }

   ```

3. set(int index, E element)方法

   这个方法是覆盖一索引上的旧值，然后返回这个旧值，可以看到就是判断下索引越界问题，然后获取数组中的值返回，再把新值放进去，时间复杂度是O(1)。

   ```
    public E set(int index, E element) {
        rangeCheck(index);

        E oldValue = elementData(index);
        elementData[index] = element;
        return oldValue;
    }

    private void rangeCheck(int index) {
        if (index >= size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }
   ```

4. add(int index, E element)方法
   
   这个方法和无参的add()方法的区别就是不是在数组后面顺序添加，而是指定位置的插入，可以看到检查是否扩容的逻辑没有变，主要是多了一步copy数组的操作，比如一个数组是[1,2,3,4,5]，需要在索引1的位置插入一个10，会先从索引1的位置copy数组，把后面的数字都往后挪一位[1,2,2,3,4,5]，然后再把索引1的2改为10，就成了[1,10,2,3,4,5]，这个操作的时间复杂度就是O(n)了，因为最坏的情况就是插入到索引0，整个数组的值都要挪动一位。

   ```
    // 插入指定索引
    public void add(int index, E element) {
        // 检查索引越界
        rangeCheckForAdd(index);
        // 检查容量
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        // copy数组，在要插入的索引位置开始一直到最后一个值，整体往后挪一格，然后把新值填入要插入的索引中
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);
        elementData[index] = element;
        size++;
    }

    // 检查容量是否足够的方法
    private void ensureCapacityInternal(int minCapacity) {
        // 判断是否需要初始化数组
        ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
    }

    // 判断是否需要初始化数组
    private static int calculateCapacity(Object[] elementData, int minCapacity) {
        // 如果数组是空的，创建一个长度10的数组
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            return Math.max(DEFAULT_CAPACITY, minCapacity);
        }
        // 不为空直接返回当前的长度
        return minCapacity;
    }

    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        // 添加后的长度要比数组大，需要调用grow()方法扩容
        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }
   ```

5. addAll(Collection<? extends E> c)方法

   这个方法是直接把另一个集合里的值追加进来，判断一下容量，然后直接把要追加的里面的值copy过来，时间复杂度是O(n)，n是追加集合的长度。
   
   ```
    public boolean addAll(Collection<? extends E> c) {
        Object[] a = c.toArray();
        int numNew = a.length;
        ensureCapacityInternal(size + numNew);  // Increments modCount
        System.arraycopy(a, 0, elementData, size, numNew);
        size += numNew;
        return numNew != 0;
    }
   ```

6. addAll(int index, Collection<? extends E> c)方法

   追加集合的方法也可以指定索引，就是addAll(Collection<? extends E> c)和add(int index, E element)的结合版，计算出索引后面的值要移动的位置进行移动，再把追加的集合的值copy到指定索引，时间复杂度也是O(n)，但是n是追加集合的长度+数组移动元素的长度。

   ```
    public boolean addAll(int index, Collection<? extends E> c) {
        rangeCheckForAdd(index);

        Object[] a = c.toArray();
        int numNew = a.length;
        ensureCapacityInternal(size + numNew);  // Increments modCount

        int numMoved = size - index;
        if (numMoved > 0)
            System.arraycopy(elementData, index, elementData, index + numNew,
                             numMoved);

        System.arraycopy(a, 0, elementData, index, numNew);
        size += numNew;
        return numNew != 0;
    }
   ```

7. get()方法

   get()方法，没啥好说的了，直接就是在数组里取值，时间复杂度也是O(1)。

   ```
    public E get(int index) {
        rangeCheck(index);

        return elementData(index);
    }
   ```

8. remove()方法

   remove()方法和插入方法很类似，但是更简单，直接把值取出来返回，然后把后面的值全部都往前挪一位，最后还把空出的那个索引位置置空让虚拟机GC清理，时间复杂度也是O(n)。

   ```
    public E remove(int index) {
        rangeCheck(index);

        modCount++;
        // 获取指定索引的值
        E oldValue = elementData(index);

        // 将删除的索引右边的值往前移动
        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work

        return oldValue;
    }
   ```

9. size()方法

   获取大小也是时间复杂度为O(1)，直接读取变量值。

   ```
    private int size;

    public int size() {
        return size;
    }
   ```

10. indexOf()/contains()方法

    这两个方法是用来获取指定目标索引和判断是否包含指定目标，```contains()```方法是调用```indexOf()```方法，而```indexOf()```方法是遍历判断，所以时间复杂度是O(n)。

    ```
    public boolean contains(Object o) {
        return indexOf(o) >= 0;
    }

    public int indexOf(Object o) {
        if (o == null) {
            for (int i = 0; i < size; i++)
                if (elementData[i]==null)
                    return i;
        } else {
            for (int i = 0; i < size; i++)
                if (o.equals(elementData[i]))
                    return i;
        }
        return -1;
    }
    ```

11. 扩容

    上面看到很多方法都会调用到```grow()```方法，也就是核心的扩容方法，可以看到本质上就是先使用位运算符来扩容1.5倍，再copy到一个新的数组，替代原来那个旧的数组。

    ```
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

    private void grow(int minCapacity) {
        // oldCapacity是存放数据数组的长度，也就是当前集合的长度
        // overflow-conscious code
        int oldCapacity = elementData.length;
        // newCapacity是计算出来的新长度，>>运算符是右移一位，相当于除以2，所以扩容是原来大小的1.5倍
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        // 扩容完之后还是不够用就直接拿所需要的最小容量作为容量值
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        // 新容量大于MAX_ARRAY_SIZE，直接使用Integer.MAX_VALUE作为新容量
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        // 把原来数组中的值copy到一个新的数组中
        elementData = Arrays.copyOf(elementData, newCapacity);
    }

    private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ?Integer.MAX_VALUE : MAX_ARRAY_SIZE;
    }

    ```

12. 内部类ListItr

    这个类实现了```ListIterator```接口，而```ListIterator```又继承了```Iterator```接口，是一个功能更加强大的迭代器。

    ```
    // 将指定的元素插入列表，插入位置为迭代器当前位置之前
    add(E e); 
    // 以正向遍历列表时，如果列表迭代器后面还有元素，则返回 true，否则返回false
    hasNext();
    // 如果以逆向遍历列表，列表迭代器前面还有元素，则返回 true，否则返回false
    hasPrevious();
    // 返回列表中ListIterator指向位置后面的元素
    next();
    // 返回列表中ListIterator所需位置后面元素的索引
    nextIndex();
    // 返回列表中ListIterator指向位置前面的元素
    previous();
    // 返回列表中ListIterator所需位置前面元素的索引
    previousIndex();
    // 从列表中删除next()或previous()返回的最后一个元素
    remove();
    // 从列表中将next()或previous()返回的最后一个元素返回的最后一个元素更改为指定元素e
    set(E e);
    ```



---   



## 总结

- 初始化的时候不指定大小时，只有在添加第一个元素时才会真正的创建底层数组。
- ```add()/set(int index, E element)/get(int index)/size()```方法的时间复杂度都是O(1)，因为不需要copy移动数组。
- ```add(int index, E element)/remove(int index)/addAll(int index, Collection<? extends E> c)/addAll(Collection<? extends E> c)```的时间复杂度是O(n)，因为都需要copy移动数组，而```indexOf()/contains()```方法则是需要遍历寻找，时间复杂度也是O(n)。
- 每次扩容都会扩容到旧长度的1.5倍，再copy旧数组的值到新数组中。
- 内部实现了```ListIterator```接口，有着功能更强大的迭代器。
