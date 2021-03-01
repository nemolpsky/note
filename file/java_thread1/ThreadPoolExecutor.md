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
     
     这个参数表示线程池中工作线程数量，如果设置了10就表示可以同时有10个线程工作，即使10个线程中只有5个工作，其他5个是空闲的，线程池也会维持这个数量的工作线程。
   - maximumPoolSize(最大线程数)

     这个参数是表示线程池中最大线程数量，因为实际中不可能你设置了10个工作线程刚好就是有10个任务进来，可能是20个任务，这时候就会创建20个线程，有10个是工作线程，另外10个线程则等待执行。
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
     // 队列 
 	 ArrayBlockingQueue<Runnable> workQueue = new ArrayBlockingQueue<>(1);
 	 ThreadFactory factory = Executors.defaultThreadFactory();
		
     // 使用AbortPolicy策略，该策略将会直接抛出RejectedExecutionException异常
 	 RejectedExecutionHandler abortHandler = new ThreadPoolExecutor.AbortPolicy();

     // 创建线程池
 	 ThreadPoolExecutor executor = new ThreadPoolExecutor(corePoolSize, maximumPoolSize, keepAliveTime, unit,workQueue, factory, abortHandler);   
     ```

   - 执行线程池
     
     这里创建了一个HashMap，长度是30，里面存了30个false。然后创建了一个线程类，
     接收这个map参数和一个int参数修改对应key的值为true，最后调用execute()方法接收一个Thread对象来执行任务，也就是有30个任务修改map中30个key的value。

     ```
     
	 // 一个map值都是false
	 HashMap<Integer,Boolean> map = new HashMap<Integer,Boolean>();
	 for (int l = 0; l < 30; l++) {
		 map.put(l,false);
	 }
		
	 int count = 0;
	 int i = 0;

	 for (i = 0; i < 30; i++) {
	     try {
		     // 不能返回线程执行的结果
			 executor.execute(new MyThread1(i,map));
			 // 调用get方法就会阻塞当前线程直到完成
		 } catch (Exception e) {
			 System.out.println("被拒绝的任务的key: " + i);
			 e.printStackTrace();
			 count ++;
		 }
	 }

     // 在所有任务执行完后关闭线程池
     executor.shutdown();

     主线程睡眠一段时间，避免上面的for循环中线程池中的任务还没有执行完就开始打印结果
     //try {
	 //    TimeUnit.SECONDS.sleep(10);
	 //} catch (InterruptedException e) {
	 //	 e.printStackTrace();
	 //}

	 // 被拒绝执行的任务数量
	 System.out.println("被拒绝的任务的数量: " + count);
		
	 // 被拒绝执行的任务对应的map里的值
	 for(Entry<Integer,Boolean> entry:map.entrySet()) {
		 if (entry.getValue() == false) {
			 System.out.println("被拒绝的任务的key: " + entry.getKey());
		 }
	 }
     ```
     
     ```
     class MyThread1 extends Thread {
         private int i;
         private HashMap<Integer,Boolean> map;

         public MyThread1(int i,HashMap<Integer,Boolean> map) {
	         this.map = map;
	         this.i = i;
	     }

	     @Override
	     public void run() {

             //try {
			 //    TimeUnit.SECONDS.sleep(1);
		     //} catch (InterruptedException e) {
			 //    e.printStackTrace();
		     //}

		     // 修改map中对应的值
		     map.put(i, true);
	     }

     }


     ```
   - 执行结果

     上面的代码中有三个地方打印了信息，第一个是catch块中，因为使用的是ThreadPoolExecutor.AbortPolicy()策略，而工作线程数只有1，最大线程数是2，30个循环有很大的可能性会有线程被拒绝，被拒绝之后就会抛出一个异常。这里一定要catch住，不然整个线程池都会停止运行，第二个是打印被拒绝的任务的数量，第三个则是遍历打印map看看value还是false没有被修改的key是不是和前面打印的对的上，验证是否任务真的被拒绝了。注意这个结果是随机的，可以在线程类里面睡眠一段时间，会发现很有很多的任务被拒绝，因为执行一个任务的时间长了，线程池的工作线程更加无法处理这么多任务，但是要注意让主线程打印结果前等待下，不然可能线程池的任务还没执行就开始打印结果了，数据会不准。
     
     ```
     被拒绝的任务的key: 7
     java.util.concurrent.RejectedExecutionException: Task Thread[Thread-7,5,main] rejected from java.util.concurrent.ThreadPoolExecutor@33909752[Running, pool size = 2, active threads = 2, queued tasks = 5, completed tasks = 0]
	 at java.util.concurrent.ThreadPoolExecutor$AbortPolicy.rejectedExecution(ThreadPoolExecutor.java:2063)
	 at java.util.concurrent.ThreadPoolExecutor.reject(ThreadPoolExecutor.java:830)
	 at java.util.concurrent.ThreadPoolExecutor.execute(ThreadPoolExecutor.java:1379)
	 at com.lp.test.ThreadPool.main(ThreadPool.java:44)
     i error 15
     java.util.concurrent.RejectedExecutionException: Task Thread[Thread-15,5,main] rejected from java.util.concurrent.ThreadPoolExecutor@33909752[Running, pool size = 2, active threads = 2, queued tasks = 1, completed tasks = 8]
	 at java.util.concurrent.ThreadPoolExecutor$AbortPolicy.rejectedExecution(ThreadPoolExecutor.java:2063)
	 at java.util.concurrent.ThreadPoolExecutor.reject(ThreadPoolExecutor.java:830)
	 at java.util.concurrent.ThreadPoolExecutor.execute(ThreadPoolExecutor.java:1379)
	 at com.lp.test.ThreadPool.main(ThreadPool.java:44)
     被拒绝的任务的数量: 1
     被拒绝的任务的key: 7
     ```

3. 拒绝策略

   上面说过了线程池中设置了最大线程数，这个数比工作线程数要大，如果执行的任务数量超过了最大线程数就会根据拒绝策略来拒绝任务，但是有需要注意的地方，就是当执行的任务超出工作线程的数量时会把空闲的任务放入队列中，只有当工作线程和队列都满了之后才会去判断数量是否超过了最大线程数。也就是说，如果设置工作线程数为1，最大线程数为1，队列为1000，这时来500个任务也不会执行拒绝操作，因为都放到队列中了。

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

## 总结
- 线程池中最重要的就是工作线程数、最大线程数、队列大小和拒绝策略四个参数，决定了对系统性能的影响
- 合理选用拒绝策略，如果选用ThreadPoolExecutor.AbortPolicy()策略，记得try住代码，否则会影响后续代码执行
- 线程池用完后要调用shutdown()方法来关闭线程池，如果不调用这个方法会发现main方法执行完后也没有关闭JVM虚拟机，调用这个方法后线程池会拒绝接受新的任务并等待现有的任务执行完之后再关闭线程池
     