## 线程池介绍

在实际开发中是极不推荐每次都手动去创建一个线程执行任务，因为每次都创建一个新的线程会造成很大的开销。所以线程池的作用就是把创建好的线程存起来进行复用，每次任务都由线程池中的这些线程就行调用，这样不仅避免了重复创建带来的开销，也避免了创建太多的线程直接造成系统卡死。

---

## 线程池使用
```
// 线程池初始化需要的参数
Integer corePoolSize = 1;
Integer maximumPoolSize = 2;
Integer keepAliveTime = 1;
TimeUnit unit = TimeUnit.SECONDS;
ArrayBlockingQueue<Runnable> workQueue = new ArrayBlockingQueue<>(1);
ThreadFactory factory = Executors.defaultThreadFactory();
// 拒绝策略
RejectedExecutionHandler abortHandler = new ThreadPoolExecutor.AbortPolicy();
RejectedExecutionHandler CallerRunsHandler = new ThreadPoolExecutor.CallerRunsPolicy();
RejectedExecutionHandler DiscardOldestHandler = new ThreadPoolExecutor.DiscardOldestPolicy();
RejectedExecutionHandler DiscardHandler = new ThreadPoolExecutor.DiscardPolicy();

// 创建线程池
ThreadPoolExecutor executor = new ThreadPoolExecutor(corePoolSize, maximumPoolSize, keepAliveTime, unit,workQueue, factory, abortHandler);
```
1. 参数
   
   - corePoolSize(核心工作线程数)
     
     这个参数表示线程池中一定会维持存在的工作线程数量，如果设置了10就表示可以一定会有10个线程存在，假设当前已有5个线程，只有1个在执行任务，新进来一个任务时因为工作线程数没达到10个，这个任务不会选用空闲的那4个线程，依然会创建一个新的线程，直到工作线程数有10个为止。
     
   - maximumPoolSize(最大线程数)

     这个参数是表示线程池中最大线程数量，当线程池工作线程占满了，队列也占满了，这个参数就会起作用，如果设置为10，这个时候就会再创建10个线程。
     
   - keepAliveTime(多余线程存活时间)
     
     这个参数则是指定了上面说的除工作线程之外多余等待的线程的存活时间，可以设置过了多长时间就把这些等待的线程清理掉。
     
   - unit(时间单位)

     接受一个TimeUnit类型的参数，表示上面keepAliveTime参数的时间单位是多少。
     
   - workQueue(队列)

     所有的排队等待空闲线程都放在这个队列中。

   - factory(线程创建工厂)

     线程池使用这个工厂来创建线程。
     
   - abortHandler(拒绝策略)

     上面讲了工作线程数和最大线程数，但是实际中任务是可能比最大线程数还要大的，这个时候线程池就会根据你的拒绝策略来处理这些多余的任务了，总共有四种策略，后续会介绍。

2. 代码示例
   - 创建一个线程池

     这里创建了一个工作线程数为1，最大线程数为2，空闲线程存活时间10秒，队列长度为1，拒绝策略是ThreadPoolExecutor.AbortPolicy()，就是直接放弃任务并抛出个异常
    
      ```
      // 工作线程数为1
      Integer corePoolSize = 1;
      // 最大线程数为2
      Integer maximumPoolSize = 2;
      // 空闲线程存活时间10秒
      Integer keepAliveTime = 10;
      TimeUnit unit = TimeUnit.SECONDS;
      // 队列容量为1
      ArrayBlockingQueue<Runnable> workQueue = new ArrayBlockingQueue<>(1);
      ThreadFactory factory = Executors.defaultThreadFactory();
        
      // 使用AbortPolicy策略，该策略将会直接抛出RejectedExecutionException异常
      RejectedExecutionHandler abortHandler = new ThreadPoolExecutor.AbortPolicy();

      // 创建线程池
      ThreadPoolExecutor executor = new ThreadPoolExecutor(corePoolSize, maximumPoolSize, keepAliveTime, unit,workQueue, factory, abortHandler);   
      ```

    - 执行线程池
      
      这里用实现Runnable的方法创建了一个简单的任务类，就打印任务名，再等待10秒。

      ```
      class MyThread implements Runnable {
      
          private final String taskName;
      
          public MyThread(String taskName) {
              this.taskName = taskName;
          }
      
          @Override
          public void run() {
              System.out.println("taskName:" + taskName);
          try {
              TimeUnit.SECONDS.sleep(10);
          } catch (InterruptedException e) {
              e.printStackTrace();
          }
      
      }
      
      MyThread thread1 = new MyThread("1");
      executor.execute(thread1);
      getExecutorsInfo(executor);
      
      MyThread thread2 = new MyThread("2");
      executor.execute(thread2);
      getExecutorsInfo(executor);
      
      MyThread thread3 = new MyThread("3");
      executor.execute(thread3);
      getExecutorsInfo(executor);
      
      //        MyThread thread4 = new MyThread("4");
      //        executor.execute(thread4);
      //        getExecutorsInfo(executor);
      
      public static void getExecutorsInfo(ThreadPoolExecutor executorService){
      // 获取线程池运行时的参数信息
      String info = String.format("线程池中当前的线程数量：%d，正在执行任务的线程数量：%d，曾经创建过的最大线程数量：%d，线程池队列中当前的任务数量：%d",
        executorService.getPoolSize(),
        executorService.getActiveCount(),
        executorService.getLargestPoolSize(),
        executorService.getQueue().size());
      
      // 打印线程池运行时的参数信息
      System.out.println(info);
     ```
   - 执行结果

     注意看结果，第1个任务进来创建了一个工作线程，第2个任务进来进队列等待，第3个任务进来又创建了一个线程。如果这个时候第4个任务也进来就会触发拒绝策略。
     
     ```
    taskName:1
    线程池中当前的线程数量：1，正在执行任务的线程数量：1，曾经创建过的最大线程数量：1，线程池队列中当前的任务数量：0
    线程池中当前的线程数量：1，正在执行任务的线程数量：1，曾经创建过的最大线程数量：1，线程池队列中当前的任务数量：1
    线程池中当前的线程数量：2，正在执行任务的线程数量：2，曾经创建过的最大线程数量：2，线程池队列中当前的任务数量：1
    taskName:3
    taskName:2
     ```

3. 拒绝策略
   线程池一般先检查工作线程数量，超出则放入队列，队列满了则检查最大线程数量，如果3者都超出设定值就会执行拒绝策略。如果设置工作线程数为1，最大线程数为1，队列为1000，这时来500个任务也不会执行拒绝操作，因为都放到队列中了，只有来1001个任务，前1000个任务把队列塞满，第1001个任务就会触发策略。。

   - ThreadPoolExecutor.AbortPolicy()

     直接放弃任务然后抛出一个RejectedExecutionException异常
     
   - ThreadPoolExecutor.CallerRunsPolicy()
     
     会在调用线程池execute()方法的线程中执行被拒绝的任务，比如在main()方法中调用的execute()那么就会在主线程中执行被拒绝的任务
     
   - ThreadPoolExecutor.DiscardOldestPolicy()

     会将队列中最先等待的任务抛弃掉，也就是队列的头部元素，然后再尝试提交任务，如果使用的是优先级别的队列则会把优先级最高的元素丢掉
     
   - ThreadPoolExecutor.DiscardPolicy()
     
     直接放弃任务，但是不抛出任何错误，也不做任何操作

4. 执行流程
- 提交一个任务到线程池
- 检查核心线程，没满就执行任务，满了则放入队列
- 队列也满了则会增加等待线程数
- 超出设置的等待线程数就会触发拒绝策略。


5. tomcat的线程池
Tomcat也是有自己的线程池来保持连接数，不过Tomcat的线程池是在原生ThreadPoolExecutor基础上进行了扩展，除了限制了核心线程数和队列长度之后，其实也就是最后一步有一点点的改动，改成下面的流程。
- 超出设置的等待线程不会立即触发拒绝策略，而是再次尝试放入队列，如果还是失败就执行拒绝策略。
  ```
  public class ThreadPoolExecutor extends java.util.concurrent.ThreadPoolExecutor {
    
    ...
    
    public void execute(Runnable command, long timeout, TimeUnit unit) {
        submittedCount.incrementAndGet();
        try {
            // 调用 Java 原生线程池的 execute 去执行任务
            super.execute(command);
        } catch (RejectedExecutionException rx) {
          // 如果总线程数达到 maximumPoolSize，Java 原生线程池执行拒绝策略
            if (super.getQueue() instanceof TaskQueue) {
                final TaskQueue queue = (TaskQueue)super.getQueue();
                try {
                    // 继续尝试把任务放到任务队列中去
                    if (!queue.force(command, timeout, unit)) {
                        submittedCount.decrementAndGet();
                        // 如果缓冲队列也满了，插入失败，执行拒绝策略。
                        throw new RejectedExecutionException("...");
                    }
                }
            }
        }
  }
  ```

大致的执行流程是下面这样，其本质原因就是避免原生线程池先放入队列，而队列又是无限长度所造成的线程数维持在核心线程数的问题。这样实际线程数量会限制在最大线程数。
- 如果当前运行的线程，少于corePoolSize，则创建一个新的线程来执行任务。
- 如果线程数大于 corePoolSize了，Tomcat 的线程不会直接把线程加入到无界的阻塞队列中，而是去判断，submittedCount（已经提交线程数）是否等于 maximumPoolSize。
- 如果等于，表示线程池已经满负荷运行，不能再创建线程了，直接把线程提交到队列，
- 如果不等于，则需要判断，是否有空闲线程可以消费。
- 如果有空闲线程则加入到阻塞队列中，等待空闲线程消费。
- 如果没有空闲线程，尝试创建新的线程。（这一步保证了使用无界队列，仍然可以利用线程的 maximumPoolSize）。
- 如果总线程数达到 maximumPoolSize，则继续尝试把线程加入 BlockingQueue 中。
- 如果 BlockingQueue 达到上限（假如设置了上限），被默认线程池启动拒绝策略，tomcat 线程池会 catch 住拒绝策略抛出的异常，再次把尝试任务加入中 BlockingQueue 中。
- 再次加入失败，启动拒绝策略。

## 总结
- 线程池中最重要的就是工作线程数、最大线程数、队列大小和拒绝策略四个参数，决定了对系统性能的影响
- 合理选用拒绝策略，如果选用ThreadPoolExecutor.AbortPolicy()策略，记得try住代码，否则会影响后续代码执行
- 线程池用完后要调用shutdown()方法来关闭线程池，如果不调用这个方法会发现main方法执行完后也没有关闭JVM虚拟机，调用这个方法后线程池会拒绝接受新的任务并等待现有的任务执行完之后再关闭线程池
     
