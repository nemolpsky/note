## LinkedList

```LinkedList```实现了```List```接口和```Deque```接口的，底层的双端链表结构使它支持高效的插入和删除操作，也具有队列的特性，非线程安全的。

---

1. 底层结构

   如果有在```LeetCode```上刷过题的话，对这种结构一定非常熟悉，这是定义的一个```Node```节点类，有三个属性，```item```是任意类型的数值，```prev```和```next```则是前置节点和后置节点，构成了整个链条的数据结构。

   ```
    private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
   ```

2. 初始化

   可以看到两个构造方法都很简单，其实都是构造方法啥都不做，只有在添加数据的时候才开始构建链表的数据结构。

   ```
    public LinkedList() {
    }

    public LinkedList(Collection<? extends E> c) {
        this();
        addAll(c);
    }
   ```

3. add()方法

   尾部追加的逻辑很清晰，将最后一个节点设置为追加节点的前置节点，同样把追加节点设置为最后一个节点的后置节点，此时追加节点会成为最后一个节点，也就是说只需要对两个节点进行关联操作即可即可，时间复杂度是O(1)。

   ```
    // 第一个节点
    transient Node<E> first;
    // 最后一个节点
    transient Node<E> last;

    public boolean add(E e) {
        linkLast(e);
        return true;
    }

    void linkLast(E e) {
        // 获得当前最后一个节点作为前置节点，可能为空
        final Node<E> l = last;
        // 初始化当前节点
        final Node<E> newNode = new Node<>(l, e, null);
        // 把当前节点作为最后的节点
        last = newNode;
        // 第一次添加设置为第一个节点
        if (l == null)
            first = newNode;
        else
            // 把当前节点设置为前置节点的后置节点
            l.next = newNode;
        size++;
        modCount++;
    }
   ```

4. add(int index,E e)方法

   按索引插入元素，首先判断是不是第一个添加的元素，如果是的话，直接使用```add()```方法添加就可以了，如果不是则需要根据索引来遍历寻找链表上对应位置，这里用了个小技巧，判断索引是在前半段还是在后半段，从短的那头开始遍历，找到之后，新建一个节点，建立新的前置节点和后置节点的关系。时间复杂度是O(n)，n为size/2。

   ```
    transient int size = 0;

    public void add(int index, E element) {
        // 检查数组越界
        checkPositionIndex(index);

        if (index == size)
            // 插入索引0的位置时，直接添加即可
            linkLast(element);
        else
            linkBefore(element, node(index));
    }

    // 获得被插入索引上的元素
    Node<E> node(int index) {
    // assert isElementIndex(index);
    // 如果索引是在链表的前半段
    if (index < (size >> 1)) {
        // 获得第一个节点
        Node<E> x = first;
        // 往后找到插入索引位置上的节点
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    }
    // 如果索引是在链表的前半段
    else {
        // 获得最后个节点
        Node<E> x = last;
        // 往前找到插入索引位置上的节点
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }

    void linkBefore(E e, Node<E> succ) {
        // assert succ != null;
        // 获得要插入索引上的节点的前置节点
        final Node<E> pred = succ.prev;
        // 创建一个节点，前置节点是索引位置上节点的前置节点，后置节点则是索引位置上的节点
        final Node<E> newNode = new Node<>(pred, e, succ);
        // 将这个节点设置成索引位置上节点的前置节点
        succ.prev = newNode;
        // 第一次插入，将该节点设置为第一个节点
        if (pred == null)
            first = newNode;
        // 不是第一次插入，将前置节点的后置节点设置成插入的节点
        else
            pred.next = newNode;
        size++;
        modCount++;
    }


   ```

5. addAll(Collection<? extends E> c)/addAll(int index, Collection<? extends E> c)方法

   这两个方法是直接把集合添加进来，而```addAll(Collection<? extends E> c)```是直接调用```addAll(int index, Collection<? extends E> c)```方法，首先会把集合转换成数组，然后找到插入位置的节点，索引位置上的节点就是后置节点，它的前置节点也就是插入的节点的前置节点，然后循环给集合中的数值创建节点，建立前后节点的关系，最后根据最开始找到的前置节点和后置节点把新创建的链表和以前的链表合成一条链表，时间复杂度是O(n)，n为插入集合的长度。

   ```
    public boolean addAll(Collection<? extends E> c) {
        return addAll(size, c);
    }

    public boolean addAll(int index, Collection<? extends E> c) {
        // 检查数组越界
        checkPositionIndex(index);
        // 集合转换成数组
        Object[] a = c.toArray();
        int numNew = a.length;
        if (numNew == 0)
            return false;

        // 声明前置节点和后置节点
        Node<E> pred, succ;
        // 在尾部插入，前置节点就是最后一个节点，没有后置节点
        if (index == size) {
            succ = null;
            pred = last;
        }
        // 调用node()方法找到后置节点，前置节点就是后置节点的前置节点
        else {
            succ = node(index);
            pred = succ.prev;
        }

        // 循环创建节点建立关系
        for (Object o : a) {
            @SuppressWarnings("unchecked") E e = (E) o;
            Node<E> newNode = new Node<>(pred, e, null);
            if (pred == null)
                first = newNode;
            else
                pred.next = newNode;
            pred = newNode;
        }

        // 后置节点为空，则最后一个节点就是前置节点
        if (succ == null) {
            last = pred;
        } 
        // 后置节点不为空，将插入的一段链表后后面的以前的链表建立关系
        else {
            pred.next = succ;
            succ.prev = pred;
        }

        size += numNew;
        modCount++;
        return true;
    }
   ```

6. addFirst(E e)/addLast(E e)方法

   ```addFirst()```方法是直接获取第一个节点，然后创建一个新节点作为新的第一个节点，而原来的第一个节点则是这个节点的后置节点，```addLast()```方法和```add()```方法没区别，都是直接在链表尾部追加，时间复杂度是O(1)。

   ```
    public void addFirst(E e) {
        linkFirst(e);
    }

    private void linkFirst(E e) {
        final Node<E> f = first;
        final Node<E> newNode = new Node<>(null, e, f);
        first = newNode;
        if (f == null)
            last = newNode;
        else
            f.prev = newNode;
        size++;
        modCount++;
    }

    public void addLast(E e) {
        linkLast(e);
    }
   ```

7. get(int index)方法

   ```get()```方法是用的上面介绍过的```node()```方法，时间复杂度是O(n)，n为size/2。

   ```
    public E get(int index) {
        // 判断数组越界
        checkElementIndex(index);
        // 遍历寻找节点
        return node(index).item;
    }
   ```

8. getFirst()/getLast()方法

   这两个方法都是直接读取```first```和```last```两个节点变量，时间复杂度是O(1)。

   ```
    public E getFirst() {
        final Node<E> f = first;
        if (f == null)
            throw new NoSuchElementException();
        return f.item;
    }

    public E getLast() {
        final Node<E> l = last;
        if (l == null)
            throw new NoSuchElementException();
        return l.item;
    }
   ```

9. indexOf(Object o)/lastIndexOf(Object o)/contains(Object o)方法
    
    ```indexOf()```是从头查找集合中一个值的索引，如果是正常的数值则是直接遍历，如果是找```null```值，则只会返回最前面的```null```的索引，因为是可以存入多个```null值```的。```lastIndexOf()```是从尾部查找，所以```null```值是返回最后面一个，```contains()```方法是查找集合中是否包含目标值，直接调用的```indexOf()```方法时间复杂度都是是O(n)。

    ```
    public int indexOf(Object o) {
        int index = 0;
        // 查找为null的节点，遍历到的第一个就返回
        if (o == null) {
            for (Node<E> x = first; x != null; x = x.next) {
                if (x.item == null)
                    return index;
                index++;
            }
        } 
        // 查找有值的节点，遍历到匹配的就返回
        else {
            for (Node<E> x = first; x != null; x = x.next) {
                if (o.equals(x.item))
                    return index;
                index++;
            }
        }
        return -1;
    }

    public int lastIndexOf(Object o) {
        int index = size;
        if (o == null) {
            for (Node<E> x = last; x != null; x = x.prev) {
                index--;
                if (x.item == null)
                    return index;
            }
        } else {
            for (Node<E> x = last; x != null; x = x.prev) {
                index--;
                if (o.equals(x.item))
                    return index;
            }
        }
        return -1;
    }

    public boolean contains(Object o) {
        return indexOf(o) != -1;
    }

    ```

10. remove()/removeFirst()/removeLast()方法

    ```remove()```方法其实就是调用的```removeFirst()```方法，移除头部结点，```removeLast()```方法是移除尾部节点，时间复杂度都是O(1)。

    ```
    public E remove() {
        return removeFirst();
    }

    public E removeFirst() {
        final Node<E> f = first;
        if (f == null)
            throw new NoSuchElementException();
        return unlinkFirst(f);
    }

    public E removeLast() {
        final Node<E> l = last;
        if (l == null)
            throw new NoSuchElementException();
        return unlinkLast(l);
    }


    ```

11. remove(Object o)/removeFirstOccurrence(Object o)/removeLastOccurrence(Object o)/remove(int index)方法

    前三个方法都是移除目标值，因为值可以重复的缘故，每次只会移除一个，```removeFirstOccurrence()```其实就是调用的```remove(Object o)```方法，从头遍历，```removeLastOccurrence()```则是从尾开始遍历，remove```(int index)```则是移除指定索引上的节点，遍历到之后都会调用一个```unlink()```方法，这个方法是用来解除移除节点跟前置节点和后置节点的关系，然后重新建立关系，因为需要先遍历查找，时间复杂度是O(n)。

    ```
    public boolean remove(Object o) {
        if (o == null) {
            for (Node<E> x = first; x != null; x = x.next) {
                if (x.item == null) {
                    unlink(x);
                    return true;
                }
            }
        } else {
            for (Node<E> x = first; x != null; x = x.next) {
                if (o.equals(x.item)) {
                    unlink(x);
                    return true;
                }
            }
        }
        return false;
    }

    public boolean removeFirstOccurrence(Object o) {
        return remove(o);
    }

    public boolean removeLastOccurrence(Object o) {
        if (o == null) {
            for (Node<E> x = last; x != null; x = x.prev) {
                if (x.item == null) {
                    unlink(x);
                    return true;
                }
            }
        } else {
            for (Node<E> x = last; x != null; x = x.prev) {
                if (o.equals(x.item)) {
                    unlink(x);
                    return true;
                }
            }
        }
        return false;
    }

    public E remove(int index) {
        checkElementIndex(index);
        return unlink(node(index));
    }

    E unlink(Node<E> x) {
        // assert x != null;
        // 获得节点的前置节点和后置节点
        final E element = x.item;
        final Node<E> next = x.next;
        final Node<E> prev = x.prev;

        //前置节点为空，第一个节点就是后置节点
        if (prev == null) {
            first = next;
        } 
        //否则前置节点的后置节点就是被移除节点的后置节点
        else {
            prev.next = next;
            x.prev = null;
        }

        // 后置节点为空，最后一个节点就是前置节点
        if (next == null) {
            last = prev;
        } 
        // 否则后置节点的前置节点就是被移除节点的前置节点
        else {
            next.prev = prev;
            x.next = null;
        }

        x.item = null;
        size--;
        modCount++;
        return element;
    }

    ```

---

## 总结
- 非线程安全
- 相比于```ArrayList```来说，```LinkedList```其实内部原理要简单的多，就是一个双端链表，所有的逻辑操作就是对链表上某一节点的前置节点和后置节点之间的关系变更，虽然看上去很绕，但是只要熟悉链表理解起来并不难。
- 因为是实现了```Deque```接口的缘故，有很多队列的方法，很多方法只是方法名字不同而已，底层都是调用同一个逻辑。
- 链表不支持随机访问，所以随机访问是需要遍历的，头部和尾部的追加操作都很快，中间的插入活删除虽然要遍历但是一般来说也比ArrayList要快，它只是进行遍历查找，找到之后对节点的关联关系进行修改，ArrayList则是要复制移动数组，效率低的多。
- 因为是链表结构，所以整个集合也就没有大小限制了，也不会有什么初始化值和扩容了。