### Java中的CAS

CAS其实就是Compare and Swap，也就是所谓的比较交换，是一种乐观锁，Java中JUC并发库下很多锁都是依赖于CAS来解决并发问题，不像synchronized关键字那样直接直接阻塞线程。

---

#### 1. CAS概念

比较交换其实就是每次更新都会有一个标志位，更新的时候需要检查标志位是否被更改，没有被更改就表示是第一个更新，也就是抢到锁了，后面的都算失败。

在Java中CAS会有3个参数，比如原子类中就是使用Unsafe来实现CAS操作，假设要更新V值，必须A值等于V值，才能够把V值修改成B值。
- V，要修改的内存值
- A，进行对比的值
- B，要修改的值，也就是把V修改成B

```
public class AtomicInteger extends Number implements java.io.Serializable {
    private static final long serialVersionUID = 6214790243416807050L;

    // setup to use Unsafe.compareAndSwapInt for updates
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long valueOffset;

    static {
        try {
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }

    private volatile int value;
    ...
}
```

---

#### 2. ABA、自旋和多值比较问题

ABA问题很好理解，假设V值操作流程是```A->B->A```，这个时候又一个线程拿着A进来要进行CAS操作时会成功，因为V确实等于A，但是实际上它已经被修改过了，只不过修改后的值刚好和修改前是一样的。

Java也在JDK1.5提供了解决方案，新增```AtomicStampedReference```类，新增了一个版本号字段，这样```A->B->A```就变成了```1A->2B->3A```的流程，1A和3A就不会被判断为一致了。
```
    public static void main(String[] args) {
        // 初始值为1，版本号为1
        AtomicStampedReference<Integer> reference = new AtomicStampedReference<>(1, 1);
        // A->值为2，版本号为2
        System.out.println(reference.compareAndSet(1, 2, 1, 2));
        // B->值为1，版本号为3
        System.out.println(reference.compareAndSet(2, 1, 2, 3));
        // C->修改失败，值匹配上了，但是版本号没有匹配
        System.out.println(reference.compareAndSet(1, 2, 1, 2));

        // 初始值为1
        AtomicInteger integer = new AtomicInteger(1);
        // A->修改值为2
        System.out.println(integer.compareAndSet(1,2));
        // B->修改值为1
        System.out.println(integer.compareAndSet(2,1));
        // C->修改值为2
        System.out.println(integer.compareAndSet(1,2));
    }
```

上面的CAS操作最大的问题其实就是只能比较一个值，JDK也提供了一个可以比较对象中多个字段的类```AtomicReference```，可以解决多值比较问题。
```
AtomicReference<String> str = new AtomicReference<>("hello");
boolean flag = str.compareAndSet(str.get(), str.get().concat(" world"));
```

因为CAS是自旋等待，其实就是一直循环获取锁，好处就是避免了上下文切换，但是线程如果多的话，那就会对CPU耗费严重，解决方法就是限制自旋的时间，比如JVM可以设置synchronized自旋的次数，ReentrantLock则可以设置自旋的时间，都可以解决。

