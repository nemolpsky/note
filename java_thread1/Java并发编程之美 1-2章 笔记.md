
#Java并发编程之美
---

## 创建线程的三种方式
- 继承Thread类
```
public static void main(String[] args) {
	Thread thread = new MyThread();
	thread.start();
}

static class MyThread extends Thread {
	@Override
	public void run() {
	}
}
```
- 实现Runnable接口
```
public static void main(String[] args) {
		
	MyRunable runable = new MyRunable();
	Thread thread = new Thread(runable);
	thread.start();
}

static class MyRunable implements Runnable {
	@Override
	public void run() {
	}
}
```
- 实现FutureTask接口，可以获得返回值
```
public static void main(String[] args) {
	FutureTask<String> task = new FutureTask<String>(new CallTask());
	new Thread(task).start();
	
	try {
		String result = task.get();
		System.out.println(result);
	} catch (Exception e) {

	}
}
	
	
static class CallTask implements Callable<String>{

	@Override
	public String call() throws Exception {
		return "Test Callable";
	}
}
```

## 等待(wait方法)和唤醒(notify和notifyAll方法)
- 当一个前程调用一个对象的wait()方法时线程会被阻塞挂起，直到别的线程调用了同一个对象的notify()和notifyAll()方法才会唤醒，而调用这两个方法的线程也必须先获得这个对象的锁才行。
```
public static void main(String[] args) throws Exception {
		Object object = new Object();

	Thread thread1 = new Thread(new Runnable() {
		@Override
		public void run() {
			synchronized (object) {
				System.out.println(" t get o");
					System.out.println("t 开始等待");
					try {
						object.wait();
					} catch (InterruptedException e) {
						e.printStackTrace();
					}
					System.out.println("子线程被唤醒了");
			}
		}
	});
	thread1.setName("生产");
	thread1.start();
		
	Thread.currentThread().sleep(1000);

	System.out.println("主线程睡3秒唤醒子线程");
	Thread.currentThread().sleep(3000);
		
	synchronized (object) {
		System.out.println("主线程获取到锁了唤醒子线程");
		object.notifyAll();
	}

	Thread.currentThread().sleep(20000);
}
```
- 如果线程没有获取的到某个对象的锁而调用这个对象的wait()方法会抛出IllegalMonitorStateException异常。
```
public static void main(String[] args) throws Exception{
	Object object = new Object();

	Thread thread1 = new Thread(new Runnable() {
		@Override
		public void run() {
			try {
				// 此处没有获取到object锁，调用object.wait()方法会抛出IllegalMonitorStateException异常
				object.wait();
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
				
			synchronized (object) {
				System.out.println(" t get o");
				while(true) {
					System.out.println(1);
					try {
						// 此处获取到object锁，调用object.wait()方法不会抛出IllegalMonitorStateException异常
						object.wait();
					} catch (InterruptedException e) {
						e.printStackTrace();
					}
				}
			}
		}
	});
	thread1.setName("生产");
	thread1.start();
	Thread.currentThread().sleep(20000);
}
```
- 调用一个对象的wait()方法只会释放当前这个对象的锁，而不会释放其他对象的锁
```
private static volatile Object resourceA = new Object();
private static volatile Object resourceB = new Object();

public static void main(String[] args) throws Exception {
	Thread thread1 = new Thread(new Runnable() {
		@Override
		public void run() {
			try {
				synchronized (resourceA) {
					System.out.println("a get a");
					synchronized (resourceB) {
						System.out.println("a get b");
						resourceA.wait();
					}
				}
			} catch (Exception e) {
				
			}
		}
	});
		
	Thread thread2 = new Thread(new Runnable() {
		@Override
		public void run() {
			try {
				Thread.sleep(1000);
				synchronized (resourceA) {
					System.out.println("b get a");
					synchronized (resourceB) {
						System.out.println("b get b");
						resourceA.wait();
					}
				}
			} catch (Exception e) {
					
			}
		}
	});
		
	thread1.start();
	thread2.start();
}
```
- 如果当线程调用wait()方法挂起阻塞时，再调用该线程的interrupt()方法会抛出java.lang.InterruptedException异常
```
public static void main(String[] args) throws Exception {
	Object object = new Object();

	Thread thread1 = new Thread(new Runnable() {
		@Override
		public void run() {
			synchronized (object) {
				System.out.println(" t get o");
					try {
						System.out.println("t 开始等待");
						object.wait();
					} catch (InterruptedException e) {
						e.printStackTrace();
					}
				}
			}
		});
		thread1.setName("生产");
		thread1.start();
		
		Thread.currentThread().sleep(1000);
		System.out.println("中断子线程");
		thread1.interrupt();

		Thread.currentThread().sleep(20000);
	}
}
```
- 一个对象的锁可能阻塞住多个线程，notify()方法会随机唤醒一个线程，notifyAll()方法则是唤醒全部线程。
```
public static void main(String[] args) throws Exception {
	Object object = new Object();

	Thread thread1 = new Thread(new Runnable() {
		@Override
		public void run() {
			synchronized (object) {
				System.out.println(" t1 get o");
				System.out.println("t1 开始等待");
				try {
					object.wait();
				} catch (InterruptedException e) {
					e.printStackTrace();
				}
				System.out.println("t1子线程被唤醒了");
			}
		}
	});
	thread1.start();
		
	Thread thread2 = new Thread(new Runnable() {
		@Override
		public void run() {
			synchronized (object) {
				System.out.println(" t2 get o");
				System.out.println("t2 开始等待");
				try {
					object.wait();
				} catch (InterruptedException e) {
					e.printStackTrace();
				}
				System.out.println("t2子线程被唤醒了");
			}
		}
	});
	thread2.setName("生产");
	thread2.start();
		
	Thread.currentThread().sleep(1000);

	System.out.println("主线程睡3秒唤醒子线程");
	Thread.currentThread().sleep(3000);
		
	synchronized (object) {
		System.out.println("主线程获取到锁了唤醒子线程");
		object.notify();
//		object.notifyAll();
	}

	Thread.currentThread().sleep(20000);
}
```

## join()方法
A线程调用B线程的join()方法，A线程会阻塞住，等到执行B线程执行完之后才会执行A线程，而且如果这个时候执行A线程的interrupt()方法会抛出InterruptedException异常
```
public static void main(String[] args) throws Exception {
	final Thread mainThread = Thread.currentThread();
		
	Thread thread1 = new Thread(new Runnable() {
		@Override
		public void run() {
			System.out.println("start t1");
				
			mainThread.interrupt();
				
			for(int i=0;i<10;i++) {
				System.out.println(i);
			}
		}
	});
	thread1.setName("生产");
	thread1.start();

	Thread.currentThread().sleep(20000);
}
```

## 守护线程
调用setDaemon(true)可以把线程设置为守护线程。线程默认是用户线程，JVM会检测有没有用户线程，如果有的话会等待用户线程执行完成。如果没有用户线程只有守护线程，JVM会直接结束而不等待守护线程执行完毕。


## ThreadLocal和InheritableThreadLocal
ThreadLocal内部是一个HashMap，以线程为key，线程中的值为value，所以用ThreadLocal来存放一个共享变量是不会用并发的问题的。InheritableThreadLocal和ThreadLocal差不多，但是InheritableThreadLocal可以在子线程的变量值为空的情况下获取到主线程中共享变量的值。

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

