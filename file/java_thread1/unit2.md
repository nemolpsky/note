# 第二章
---

## 线程安全问题
比如有一个变量A进行自增，当前A变量是1，如果线程1获取到了A是1，然后还没进行自增，线程2这个时候也获取到了A也是1，然后线程1和线程2都进行了+1的操作，最后A变量的值是2，结果就是有问题的，因为做了两次操作应该是3才对。

## 共享变量内存可见性问题
从Java的内存模型来解释上面的线程安全问题，共享变量是存在主内存中，而每个线程都会有一个自己的工作内存，这里工作缓存一般对应CPU硬件中的一级缓存或者二级缓存，所以修改变量的时候先把主内存中的变量值复制到线程的工作内存中，修改完再复制回主内存中。因为从主内存复制到工作内存，修改工作内存，从工作内存复制到主内存不是原子性操作，所以就会发生线程安全问题。

## synchronized关键字
可以解决线程安全问题，原子性操作，被修饰的代码每次只允许同一个线程操作。只有获取到锁的线程才可以执行，当线程执行完成或者抛出异常才会释放锁。因为阻塞线程的时候会切换线程状态，所以会导致上下文切换，所以性能消耗很大。
- 内存语义
  把synchronized锁住的代码中所有的变量在工作线程中的复制值都清除掉，这样所有对变量的读取都是从主内存中读取，在synchronized代码结束的时候会把最新的变量值都刷新到主内存中。

## volatile
可以保证被修饰的变量的更新对其他内存是立刻可见的，但是对这个变量的操作不是原子性的，不会阻塞其他线程，性能消耗小。
- 内存语义
  和synchronized很相似，写入变量值是直接写入主内存，其他线程读取的时候也是直接读取主内存。
- 使用场景
  当对变量的操作并不依赖当前变量的值，因为读取-计算-写入不是原子性操作，volatile也不保证原子性操作。
  
## Unsafe类
```Unsafe```类中都是```native```方法，所以提供了硬件级别的原子操作。但是```Java```对这个类做了限制，不能直接获取对象来操作。不过可以使用反射来获取实例，并发包中很多类都是基于这个类来实现。
```
public class UnsafeTest {
	// Unsafe对象引用
	static Unsafe unsafe;
	// 偏移量
	long l = 0;
	
	public static void main(String[] args) throws Exception {
		// 反射获取Unsafe实例
		Field field = Unsafe.class.getDeclaredField("theUnsafe");
		// 设置为可取
		field.setAccessible(true);
		// 指向引用
		unsafe = (Unsafe) field.get(null);
		
		// 计算变量l在内存中的偏移量
		long offset = unsafe.objectFieldOffset(UnsafeTest.class.getDeclaredField("l"));
	
		// 调用compareAndSwapLong方法，第一个是操作的对象，第二个是偏移量等于当前传入值的值，第三个是期望值，第四个是更新值
		// 下面的调用，会先获取UnsafeTest中偏移量等于offset的变量值，然后拿这个值和0对比，如果相同就更新为2，更新成功返回true，失败返回false
		UnsafeTest test = new UnsafeTest();
		System.out.println(unsafe.compareAndSwapLong(test, offset, 0, 2));
		System.out.println(test.l);
	}
}
```

## Java指令重排序
Java允许编译器和处理器对执行的指令进行重排来提高性能，只要指令之间不存在依赖性。
```
// 按代码属性可能是执行 123，但是经过指令重排后可能是213
int a = 1; //1
int b = 2; //2
int c = a + b; //3
```
```
static int num = 0;
static boolean ready = false;
	
public static void main(String[] args) throws Exception {
	Thread thread1 = new Thread1();
	thread1.setPriority(1);
	Thread thread2 = new Thread2();
	thread1.setPriority(10);
		
	thread1.start();
	thread2.start();
		
	Thread.currentThread().sleep(500);
	thread1.interrupt();
	}
	
static class Thread1 extends Thread{
	@Override
	public void run() {
		System.out.println("read");
		while(!Thread.currentThread().isInterrupted()) {
			if (ready) {
				System.out.println( num + num);
			}
		}
	}
}

static class Thread2 extends Thread{
	@Override
	public void run() {
		System.out.println("write");
		num = 2;
		ready = true;
	}
}
```

## 伪共享
CPU与主内存之间会有缓存器，线程会先读取缓存器，如果没有才会读主内存。而当一个缓存器只能允许一个线程操作，但是一个缓存器里可能会有多个变量值，这个时候其他的线程又无法操作，性能就会下降。这就是伪共享。可以使用补位的方法来解决，比如一个缓存器的大小是64位，就额外声明变量了填满。但是也不一定多个变量放一个缓存内就一定会性能下降，如果是单线程反而更快，因为所有数据都存在同一个缓存器中。
```
// value占8位 6个long变量占8位  对象字节码的对象头也占8位
class FilledLong {
	public volatile long value = 0;
	public long p1, p2, p3, p4, p5, p6;
}
```

## 锁的类型
- 悲观锁和乐观锁
  - 悲观锁是指对并发问题持悲观态度，认为一定会发生并发问题，所以同一时间只允许一个线程来操作数据，是排它锁，其他的线程在这期间都会堵塞住。
  - 乐观锁则是数据会有类似版本号的字段来表示，比如当前数据版本号是1，同一时间所有线程查询到这个版本号然后更新的时候和要更新的数据进行对比，如果版本号对的上就表示可以更新，对不上则表示数据已经被修改过了，直接更新失败，而不是像悲观锁那样阻塞住。
- 公平锁和非公平锁
  - 公平锁，```ReentrantLock lock = new ReentrantLock(true)```，如果A线程在执行，B线程被阻塞住，C线程再来，当A释放锁时一定会是B获得锁，会影响性能。
  - 非公平锁，```ReentrantLock lock = new ReentrantLock(false)```，如果A线程在执行，B线程被阻塞住，C线程再来，当A释放锁时B和C都有可能获得锁。
- 独占锁和共享锁
  - 独占锁就是只有一个线程能拿到锁，例如```ReentrantLock```
  - 共享锁就是多个线程可以共同持有锁，例如```ReadWriteLock```
- 可重入锁
  - 可重入锁就是同一个线程可以多次获取一个锁，内部有个计数器来计算一个线程的获取锁的次数，每次获取都要加1，每次释放都要减1。例如下面的方法，```synchronized```就是可重入锁，所以在methodA中调用methodB不会被阻塞住。
  ```	
  public static void main(String[] args) throws InterruptedException {
	  methodA();
  }

  public static void methodA() throws InterruptedException {
	  synchronized (lockA) {
		  System.out.println("methodA");
		  methodB();
		  TimeUnit.SECONDS.sleep(5);
	      System.out.println("methodA");
	  }
  }

  public static synchronized void methodB() {
	  synchronized (lockA) {
		  System.out.println("methodB");
		  System.out.println("methodB");
	  }
  }
  ```