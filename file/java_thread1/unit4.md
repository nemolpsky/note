
## 目录
<!-- TOC -->
- [原子操作类](#原子操作类)
- [LongAddr类](#LongAddr类)
- [LongAccumulator](#LongAccumulator)
<!-- /TOC -->

---

## 原子操作类
在```java.util.concurrent.atomic```包下有各种各样的原子操作类，能够确保线程安全的更新变量值或者是进行自增、自减，其大致底层基本都是基于前面第二章到的```Unsafe```类实现的。

```
static AtomicLong longA = new AtomicLong(1);
	
public static void main(String[] args) {
    System.out.println(longA);
    // 第一个参数是期望值，当这个参数和当前值相同才会更新成功
    System.out.println(longA.compareAndSet(0, 10));
    System.out.println(longA);
    System.out.println(longA.compareAndSet(1, 10));
    System.out.println(longA);

    longA.getAndIncrement();
    System.out.println(longA);
}
```

---
## LongAddr类
虽然原子操作类已经可以保证线程安全了，但是当很多个线程一起操作一个原子类变量时会有很大的性能问题，因为只有一个线程能够成功，其他的线程都会失败，而这些失败的线程都会进行自旋操作，对性能造成很大的损耗，所以就有```LongAddr```类。
- 使用方式，可以看到使用起来非常简单，```add()```方法添加，```decrement()```方法自减，```increment()```方法自增，还可以转换为```int```或```double```等类型。
  ```
  static LongAdder longA = new LongAdder();
	
  public static void main(String[] args) {
	  longA.add(1);
	  System.out.println(longA);
	  longA.add(1);
	  System.out.println(longA);
	  longA.decrement();
      System.out.println(longA);
	  longA.increment();
	  System.out.println(longA);
  }
  ```
- 底层原理
  1. 并不像原子操作类是多个线程竞争一个变量，底层除了有一个基本的变量值之外，还有一个```Cell```数组来存放所有```Cell```变量，```Cell```类的底层其实对一个普通的```long```类型变量进行CAS操作。所以多个线程下是竞争多个```Cell```变量，最后再把所有的变量值加起来汇总，减少了自旋操作带来的压力，也减少了竞争压力。
  2. 还使用了```@sun.misc.Contended```注解来解决伪共享的问题。
  3. 一开始是不会初始化```Cell```类的，只有到一定的条件才会创建并且进行后续扩容。
  
--- 
## LongAccumulator类
这个类其实就是```LongAddr```类的加强版，不同的是第一个接受的参数是自定义的运算器，可以进行自定义的计算，比如相乘，第二参数则是默认值。
```
static LongAccumulator longA = new LongAccumulator(new LongBinaryOperator() {
	public long applyAsLong(long arg0, long arg1) {
		return arg0 * arg1;
	}
}, 10);
	
public static void main(String[] args) {
	System.out.println(longA);
	longA.accumulate(2);
	System.out.println(longA);
	longA.accumulate(3);
	System.out.println(longA);
}
```