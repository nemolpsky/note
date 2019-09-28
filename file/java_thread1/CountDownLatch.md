## CountDownLatch底层原理

```CountDownLatch```也是一个```java.util.concurrent```包中的类，可以设置一个初始数值，在数值大于0之前让调用```await()```方法的线程堵塞住，数值为0是则会放开所有阻塞住的线程。

### 使用例子：
```
public static void main(String[] args) throws InterruptedException {

    // 设置初始数值为10
	CountDownLatch latch = new CountDownLatch(10);

    // 循环中调用countDown()减1，如果调用9次则数值为1，主线程和子线程都会阻塞，改为i<10调用10次则主线程和子线程都可以运行
	for(int i=0;i<9;i++) {
		latch.countDown();
		System.out.println(latch.getCount());
	}
	
	
	new Thread(new Runnable() {
		
		@Override
		public void run() {
			System.out.println("thread start");
			try {
                // 阻塞子线程
				latch.await();
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			System.out.println("thread end");
		}
	}).start();
	
    // 阻塞主线程
	latch.await();
	System.out.println("main end");

}
```

### 底层原理：
1. 构造方法
   
   内部也是有个```Sync```类继承了```AQS```，所以```CountDownLatch```类的构造方法就是调用```Sync```类的构造方法，然后调用```setState()```方法设置```AQS```中```state```的值。

   ```
    public CountDownLatch(int count) {
        if (count < 0) throw new IllegalArgumentException("count < 0");
        this.sync = new Sync(count);
    }

    Sync(int count) {
        setState(count);
    }    
   ```

2. await()

   该方法是使调用的线程阻塞住，直到```state```的值为0就放开所有阻塞的线程。实现会调用到```AQS```中的```acquireSharedInterruptibly()```方法，先判断下是否被中断，接着调用了```tryAcquireShared()```方法，讲```AQS```那篇文章里提到过这个方法是需要子类实现的，可以看到实现的逻辑就是判断```state```值是否为0，是就返回1，不是则返回-1。

   ```
    public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }

    public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (tryAcquireShared(arg) < 0)
            doAcquireSharedInterruptibly(arg);
    }

    protected int tryAcquireShared(int acquires) {
        return (getState() == 0) ? 1 : -1;
    }
   ```

3. countDown()

   这个方法会对```state```值减1，会调用到```AQS```中```releaseShared()```方法，目的是为了调用```doReleaseShared()```方法，这个是AQS定义好的释放资源的方法，而```tryReleaseShared()```则是子类实现的，可以看到是一个自旋```CAS```操作，每次都获取```state```值，如果为0则直接返回，否则就执行减1的操作，失败了就重试，如果减完后值为0就表示要释放所有阻塞住的线程了，也就会执行到```AQS```中的```doReleaseShared()```方法。
   

   ```
    public void countDown() {
        sync.releaseShared(1);
    }

    public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }

    protected boolean tryReleaseShared(int releases) {
        // Decrement count; signal when transition to zero
        for (;;) {
            int c = getState();
            if (c == 0)
                return false;
            int nextc = c-1;
            if (compareAndSetState(c, nextc))
                return nextc == 0;
        }
    }
   ```