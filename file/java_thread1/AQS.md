### AQS底层原理
```AQS(AbstractQueuedSynchronizer)```按名字理解是抽象同步队列，其实本质上就是一个双向队列，队列里面的节点存放的就是线程，节点中会有其他的一些属性值表明当前线程的状态，阻塞，等待还是睡眠。```JUC(java.util.concurrent)```中很多同步锁都是基于```AQS```实现的。

```AQS```的基本原理就是当一个线程请求共享资源的时候会判断是否能够成功操作这个共享资源，如果可以就会把这个共享资源设置为锁定状态，如果当前共享资源已经被锁定了，那就把这个请求的线程阻塞住，也就是放到队列中等待。

---

### 一、state变量
- ```AQS```中有一个被```volatile```声明的变量用来表示同步状态
- 提供了```getState()```、```setState()```和```compareAndSetState()```方法来修改```state```状态的值
  ```
  // 返回同步状态的当前值
  protected final int getState() {  
    return state;
  }
 
  // 设置同步状态的值
  protected final void setState(int newState) { 
    state = newState;
  }

  // CAS操作修改state的值
  protected final boolean compareAndSetState(int expect, int update) {
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
  }
  ```

---

### 二、 对共享资源的操作方式：
上面说了```AQS```是```JUC```中很多同步锁的底层实现，锁也分很多种，有像```ReentrantLock```这样的独占锁，也有```ReentrantReadWriteLock```这样的共享锁，所以```AQS```中也必然是包含这两种操作方式的逻辑

1. 独占式

   - 获取资源的时候会调用```acquire()```方法，这里面会调用```tryAcquire()```方法去设置```state```变量，如果失败的话就把当前线程放入一个```Node```中存入队列
   ```
   public final void acquire(int arg) {
       if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
           selfInterrupt();
    }
   ```

   - 释放资源的时候是调用```realase()```方法，会调用```tryRelease()```方法修改```state```变量，调用成果后会去唤醒队列中```Node```里的线程，```unparkSuccessor()```方法就是判断当前```state```变量是否符合唤醒的标准，如果合适就唤醒，否则继续放回队列
   ```
   public final boolean release(int arg) {
       if (tryRelease(arg)) {
           Node h = head;
           if (h != null && h.waitStatus != 0)
               unparkSuccessor(h);
           return true;
       }
       return false;
   }
   ```
   - 注意```tryAcquire()```和```tryRelease()```方法在```AQS```中都是空的，前面说了```JUC```中很多同步锁都是基于```AQS```实现，所以加锁和释放锁的逻辑都还不确定，因此是要在这些同步锁中实现这两个方法
   ```
   protected boolean tryAcquire(int arg) {
        throw new UnsupportedOperationException();
   }

   protected boolean tryRelease(int arg) {
        throw new UnsupportedOperationException();
   }
   ```

2. 共享式
   
   - 获取资源会调用```acquireShared()```方法，会调用```tryAcquireShared()```操作```state```变量，如果成功就获取资源，失败则放入队列

     ```
     public final void acquireShared(int arg) {
         if (tryAcquireShared(arg) < 0)
             doAcquireShared(arg);
     }
     ```

   - 释放资源是调用```releaseShared()```方法，会调用```tryReleaseShared()```设置```state```变量，如果成功就唤醒队列中的一个```Node```里的线程，不满足唤醒条件则还放回队列中
     ```
     public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
     }    
     ```
   - 和独占式一样，```tryAcquireShared()```和```tryReleaseShared()```也是需要子类来提供
     ```
     protected int tryAcquireShared(int arg) {
        throw new UnsupportedOperationException();
     }

     protected boolean tryReleaseShared(int arg) {
        throw new UnsupportedOperationException();
     }
     ```

---

### 三、条件变量Condition
```Condition```中的```signal()```和```await()```方法类似与```notify()```和```wait()```方法，需要和```AQS```锁配合使用。
```
public static void main(String[] args) throws InterruptedException {

	ReentrantLock lock = new ReentrantLock();
	Condition condition = lock.newCondition();

	Thread thread1 = new Thread(new Runnable() {
		@Override
		public void run() {
			lock.lock();
			System.out.println(" t1 加锁");
			System.out.println("t1 start await");
			try {
				condition.await();
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			System.out.println("t1 end await");
			lock.unlock();
		}
	});
	thread1.start();

	Thread thread2 = new Thread(new Runnable() {
		@Override
		public void run() {
			lock.lock();
			System.out.println(" t2 加锁");
			System.out.println("t2 start signal");
			condition.signal();
			System.out.println("t2 end signal");
			lock.unlock();
		}
	});
	thread2.start();

}
```

- 在```AQS```中的原理

  上面```lock.newCondition()```其实是```new```一个```AQS```中```ConditionObject```内部类的对象出来，这个对象里面有一个队列，当调用```await()```方法的时候会存入一个```Node```节点到这个队列中，并且调用```park()```方法阻塞当前线程，释放当前线程的锁。而调用```singal()```方法则会移除内部类中的队列头部的```Node```，然后放入```AQS```中的队列中等待执行机会。

- 同样的，```AQS```并没有实现```newCondition()```方法，也是需要子类自己去实现