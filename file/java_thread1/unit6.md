
## 目录
<!-- TOC -->
- [LockSupport类](#LockSupport类)
  - [park()方法](#park()方法)
  - [unPark()方法](#unPark()方法)
  - [其他一些重载的方法](#其他一些重载的方法)

<!-- /TOC -->

---
## LockSupport类
是专门用来挂起和唤醒线程的类，它可以与使用它的线程关联起来，但是要调用特定方法才会关联，否则默认是不关联的。

---

### park()方法
如果```LockSupport```和线程还没有建立关联的时候，该方法是使当前调用的线程阻塞住，只有当调用```unpark()``` 方法和```interrupt()```方法才会唤醒阻塞的线程，要注意的是因为```park()```方法而阻塞的线程被```interrupt()```中断时不会抛出异常。如果已经有关联了则不会阻塞，而是立刻返回。
```
public static void test1() throws InterruptedException {
    System.out.println("m-start");
    // 调用会阻塞
    LockSupport.park();
    System.out.println("m-end");
}

public static void test2() throws InterruptedException {
    new Thread(new Runnable() {
        @Override
        public void run() {
            System.out.println("c-start");
            // 直接阻塞
            LockSupport.park();
            System.out.println("c-end");
        }
    }).start();;

    TimeUnit.SECONDS.sleep(1);
    System.out.println("m-start");
    System.out.println("m-end");
}


public static void test3() throws InterruptedException {
    Thread thread = new MyThread();
    thread.start();

    System.out.println("m-start");
    System.out.println("m-end");
    thread.interrupt();
}

class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println("c-start");
        try {
            // 如果是睡眠阻塞，调用interrupt方法中断，会报错
            TimeUnit.SECONDS.sleep(2);
            // 如果是park阻塞，调用interrupt方法中断，不会报错
//			LockSupport.park();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("c-end");
    }
}
```
### unPark()方法
最开始介绍了说默认情况下```LockSupport```类是不会和线程产生关联的，而调用了```unPark()```方法则会建立这种关联。并且还会让调用```park()```方法阻塞的线程立刻放回。
```
public static void test1() throws InterruptedException {
    // 可以让线程获得关联，否则默认是没有关联的
    LockSupport.unpark(Thread.currentThread());
    System.out.println("m-start");
    // 不会阻塞，因为已经有关联了
    LockSupport.park();
    System.out.println("m-end");
}
```

### 其他一些重载的方法
- park(Object blocker)
  设置了阻塞对象，线程阻塞查看堆栈信息的时候可以看到阻塞对象是传入的```blocker```
- parkNanos(long nanos)
  带参数的```park()```方法，可以设置超时时间
- parkNanos(Object blocker, long nanos)
  在设置超时时间的基础上，还可以设置了阻塞对象。
- parkUntil(Object blocker, long deadline)
  和```parkNanos()```很相似，不同之处在于它将指定的时间点减去1970年这个时间点的毫秒数来作为超时时间