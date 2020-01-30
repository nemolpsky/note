## 常用线程池介绍


1. FixedThreadPool

    FixedThreadPool的特点是工作线程数和最大线程数都是相同的，因为使用了无界队列LinkedBlockingQueue队列，所以任务容量可以达到Intger.MAX_VALUE，极端情况下会导致OOM问题。

    ```
    Executors.newFixedThreadPool(3);
    Executors.newFixedThreadPool(3,Executors.defaultThreadFactory());

    public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,0L, TimeUnit.MILLISECONDS,new LinkedBlockingQueue<Runnable>());
    }
    ```

2. SingleThreadExecutor

    SingleThreadExecutor是线程池内部只有一个工作线程，配合上队列这个线程池可以包含任务的执行顺序，但是它和FixedThreadPool有一样的问题，也是用了无界队列，可能会导致OOM问题。

    ```
    Executors.newSingleThreadExecutor();
    Executors.newSingleThreadExecutor(Executors.defaultThreadFactory());

    public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,0L, TimeUnit.MILLISECONDS,new LinkedBlockingQueue<Runnable>()));
    }
    ```

3. CachedThreadPool

    CachedThreadPool队列设置了最大线程数是Integer.MAX_VALUE，而核心线程数是0，所以这就表示每次任务进来都会开启一个新的线程，所以极端情况下会耗尽整个服务器的资源。

    ```
    Executors.newCachedThreadPool();
    Executors.newCachedThreadPool(Executors.defaultThreadFactory());

    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,60L, TimeUnit.SECONDS,new SynchronousQueue<Runnable>());
    }
    ```

4. ScheduledThreadPool
   
    ScheduledThreadPool最大的不同之处在于使用了DelayedWorkQueue队列，这是一个时间优先级队列，它会根据时间将队列里的任务进行排序，时间短的将会先被执行。
   
    ```
    Executors.newScheduledThreadPool(5);
    Executors.newScheduledThreadPool(5, Executors.defaultThreadFactory());

    public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
    }

    public ScheduledThreadPoolExecutor(int corePoolSize) {
        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
              new DelayedWorkQueue());
    }
    ```

5. SingleThreadScheduledExecutor

    SingleThreadScheduledExecutor队列只是在ScheduledThreadPool队列的基础上封装了一下，线程池中只有一个线程在运行。

    ```
    Executors.newSingleThreadScheduledExecutor();
    Executors.newSingleThreadScheduledExecutor(Executors.defaultThreadFactory());

    public static ScheduledExecutorService newSingleThreadScheduledExecutor() {
        return new DelegatedScheduledExecutorService
            (new ScheduledThreadPoolExecutor(1));
    }
    ```

---

## 总结
Executors工具类中提供了5种线程池的封装，后两种基本很少使用，而前三种的配置在极端情况下都会有使服务器宕机的风险，所以使用线程池的时候最好自定义配置参数。