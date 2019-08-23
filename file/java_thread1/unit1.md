
## 目录
<!-- TOC -->
- [创建线程的三种方式](#创建线程的三种方式)
- [wait、notify、notifyAll](#wait、notify、notifyAll)
- [join](#join
- [ThreadLocal和InheritableThreadLocal](#ThreadLocal和InheritableThreadLocal)
<!-- /TOC -->

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

---


## wait、notify、notifyAll
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

---


## join
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

---


## 守护线程
调用setDaemon(true)可以把线程设置为守护线程。线程默认是用户线程，JVM会检测有没有用户线程，如果有的话会等待用户线程执行完成。如果没有用户线程只有守护线程，JVM会直接结束而不等待守护线程执行完毕。

---

## ThreadLocal和InheritableThreadLocal
ThreadLocal内部是一个HashMap，以线程为key，线程中的值为value，所以用ThreadLocal来存放一个共享变量是不会用并发的问题的。InheritableThreadLocal和ThreadLocal差不多，但是InheritableThreadLocal可以在子线程的变量值为空的情况下获取到主线程中共享变量的值。

