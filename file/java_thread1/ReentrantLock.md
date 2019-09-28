## ReentrantLock底层原理
```ReentranLock```是一个支持重入的独占锁，在```java.util.concurrent```包中，底层就是基于```AQS```实现的。

- Sync类

  在源码中可以看到```ReentrantLock```类本身并没有继承```AQS```，而是创建了一个内部类```Sync```来继承```AQS```，而```ReentrantLock```类本身的那些方法都是调用```Sync```里面的方法来实现，而```Sync```本身自己也是一个抽象类，它还有两个子类，分别是```NonfairSync```和```FairSync```，对锁各种实际的实现其实在这两个类中实现，顾名思义，这两个类分别实现了非公平锁和公平锁，在创建```ReentrantLock```时可以进行选择。

  ```
  // 默认构造函数是创建一个非公平锁
  public ReentrantLock() {
      sync = new NonfairSync();
  }

  // 接受一个boolean参数，true是创建公平锁，false是非公平锁
  public ReentrantLock(boolean fair) {
      sync = fair ? new FairSync() : new NonfairSync();
  }
  ```

- lock()

  - 会调用Sync类中的lock()方法，所以需要看创建的是公平锁还是非公平锁
  
    ```
    public void lock() {
        sync.lock();
    }
    ```
  - 非公平锁中的```lock()```方法，先试用```CAS```的方式更新```AQS```中的```state```的状态，默认是0代表没有被获取，当前线程就可以获取锁，然后把```state```改为1，接着把当前线程标记为持有锁的线程，如果```if```中的操作失败就表示锁已经被持有了，就会调用```acquire()```方法
    
    ```
    final void lock() {
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            acquire(1);
    }    
    ```

  - ```acquire()```方法是```AQS```中的那个方法，这里面调用子类实现的```tryAcquire()```方法，最终是调用到```Sync```类中的```nonfairTryAcquire()```方法，可以看到先判断```state```是不是0，也就是能不能获取锁，如果不能则判断请求锁的线程和持有锁的是不是同一个，如果是的话就把```state```的值加1，也就是实现了重入锁
    ```
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }

    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }    

    final boolean nonfairTryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                return true;
            }
        } else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
    ```

  - 重写的```tryAcquire()```方法，上面非公平锁是调用的```Sync```里的```tryAcquire()```方法，而公平锁则在子类```NonfairSync```中重写了这个方法，注意```if(c==0)```判断中的代码，也就是线程抢夺锁的时候会调用```hasQueuedPredecessors()```方法，这个方法会判断队列中有没有已经先等待的线程了，如果有则当前线程不会抢到锁，这就实现了公平性，上面```nonfairTryAcquire()```方法则没有这种判断，所以后来的线程可能会比先等待的线程先拿到锁

    ```
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }

    public final boolean hasQueuedPredecessors() {
        // The correctness of this depends on head being initialized
        // before tail and on head.next being accurate if the current
        // thread is first in queue.
        Node t = tail; // Read fields in reverse initialization order
        Node h = head;
        Node s;
        return h != t &&
            ((s = h.next) == null || s.thread != Thread.currentThread());
    }
    ```

- tryLock()
  
  这个方法是尝试去获取锁，如果成功返回```true```，失败则返回```false```。会发现调用的就是上面```Sync```中的```nonfairTryAcquire()```方法

  ```
  public boolean tryLock() {
      return sync.nonfairTryAcquire(1);
  }
  ```

- unlock()

  这个方法是释放锁，最终会调用到```Sync```类中的```tryRelease()```方法。在这个方法里面会对```state```减1，如果减1之后为0就表示当前线程持有次数彻底清空了，需要释放锁

  ```
  public void unlock() {
      sync.release(1);
  }

  public final boolean release(int arg) {
      if (tryRelease(arg)) {
      Node h = head;
      if (h != null && h.waitStatus != 0)
          unparkSuccessor(h);
          return true;
      }
      return false;
  }  

  protected final boolean tryRelease(int releases) {
      int c = getState() - releases;
      if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
      boolean free = false;
      if (c == 0) {
          free = true;
          setExclusiveOwnerThread(null);
      }
      setState(c);
      return free;
  }  
  ```