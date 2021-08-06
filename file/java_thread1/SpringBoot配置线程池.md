### SpringBoot配置线程池

```Spring```家族里面的框架，从```SpringBoot```到```Spring Cloud```，它们的本质其实都是减少对象的控制，减少模板配置的书写，减少工作量，有时候很多人会觉得配置```Spring```会比较复杂，但是只要理解好```Spring```注入和配置的本质是什么就不会觉得复杂了。其本质就是```Spring```自己提供了多种方式来创建对象，比如```@Configuration```注解用代码创建对象，或者配置xml文件来生成对象，然后放到```Spring```容器对象中，使用的时候在注入即可。弄懂这个步骤的大致原理，基本上就能搞定```Spring```的自定义配置了。

#### 1. 线程池配置

下面这种方式是使用代码注解来生成Bean对象，```@Configuration```表示当前是个配置类，```@Bean()```则表示这个方法会生成一个Bean对象，默认名字就是方法名。所以这其实就是创建了一个最普通不过的线程池对象。

```
@Configuration
public class ThreadPoolConfig {

    @Bean()
    public Executor executor1() {
        // 工作线程数为1
        Integer corePoolSize = 2;
        // 最大线程数为2
        Integer maximumPoolSize = 3;
        // 空闲线程存活时间10秒
        Integer keepAliveTime = 120;
        TimeUnit unit = TimeUnit.SECONDS;
        // 队列
        ArrayBlockingQueue<Runnable> workQueue = new ArrayBlockingQueue<>(1);
        ThreadFactory factory = Executors.defaultThreadFactory();
        // 使用AbortPolicy策略，该策略将会直接抛出RejectedExecutionException异常
        RejectedExecutionHandler abortHandler = new ThreadPoolExecutor.AbortPolicy();
        // 创建线程池
        ThreadPoolExecutor executor = new ThreadPoolExecutor(corePoolSize, maximumPoolSize, keepAliveTime, unit
                , workQueue, factory, abortHandler);
        return executor;
    }
}
```

下面可以看到注入线程池对象，```Spring```会根据对象名或者类型来匹配，只要确保能匹配到就会注入上面创建的那个线程池对象，调用的时候可以成功看到线程池中线程打印的运行日志。

```
@RestController
@RequestMapping(value = "/thread")
public class ThreadController {
    @Autowired
    Executor executor1;

    @GetMapping(value = "/delete")
    public void delete() {
        executor1.execute(new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println(Thread.currentThread().getName() + ":delete");
            }
        }));
    }
}

```

---

#### 2. Spring异步注解

```Spring```自己提供了异步注解，其实就是给方法上打一个注解，这个方法就会自动放到线程池中执行。下面这个方法使用了```@Async()```，```Spring```会默认创建一个线程池，把这个方法放入线程池中执行。

```
@Service
public class AsyncService {

    private final Logger logger = LoggerFactory.getLogger(this.getClass());

    @Async()
    public void execute1(){
        try {
            String threadName = Thread.currentThread().getName();
            logger.info("execute1 : {}",threadName);
            TimeUnit.SECONDS.sleep(10);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

此外异步注解也可以灵活配置，比如使用自己的线程池。这个时候需要在启动类上添加```@EnableAsync```注解，才能保证配置自己的线程池生效。

```
@EnableAsync
@SpringBootApplication
public class UploadApplication {

    public static void main(String[] args) {
        SpringApplication.run(UploadApplication.class, args);
    }

}
```

因为最开始的时候有配置一个线程池对象，所以这里可以直接使用，填上对象的名字即可。这里两个方法都是配置同一个线程池，如果有需要可以配置不同的线程池。

```
@Service
public class AsyncService {

    private final Logger logger = LoggerFactory.getLogger(this.getClass());

    @Async("executor1")
    public void execute1(){
        try {
            String threadName = Thread.currentThread().getName();
            logger.info("execute1 : {}",threadName);
            TimeUnit.SECONDS.sleep(10);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    @Async("executor1")
    public void execute2(){
        try {
            String threadName = Thread.currentThread().getName();
            logger.info("execute2 : {}",threadName);
            TimeUnit.SECONDS.sleep(10);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

@RestController
@RequestMapping(value = "/thread")
public class ThreadController {

    @Autowired
    private AsyncService asyncService;

    @GetMapping(value = "/get")
    public void get() {
        asyncService.execute1();
    }

    @GetMapping(value = "/post")
    public void post() {
        asyncService.execute2();
    }
}
```


---

#### 3. 配置线程池的名字

使用线程池来实现多线程虽然方便了不少，但是定位问题却麻烦得多，所以最好给线程池和线程都起上辨识度高的名字，这样出问题的时候才容易定位问题出在哪。其实默认的线程池已经有名字了，名字的定义在```DefaultThreadFactory```里面，最简单的办法就是给每个线程池实现一个自定义的线程工厂改一下名字格式即可。

```
/**
  * The default thread factory
  */
static class DefaultThreadFactory implements ThreadFactory {
    private static final AtomicInteger poolNumber = new AtomicInteger(1);
    private final ThreadGroup group;
    private final AtomicInteger threadNumber = new AtomicInteger(1);
    private final String namePrefix;

    DefaultThreadFactory() {
        SecurityManager s = System.getSecurityManager();
        group = (s != null) ? s.getThreadGroup() : Thread.currentThread().getThreadGroup();
        namePrefix = "pool-" + poolNumber.getAndIncrement() + "-thread-";
    }

    public Thread newThread(Runnable r) {
        Thread t = new Thread(group, r, namePrefix + threadNumber.getAndIncrement(), 0);
        if (t.isDaemon())
            t.setDaemon(false);
        if (t.getPriority() != Thread.NORM_PRIORITY)
            t.setPriority(Thread.NORM_PRIORITY);
        return t;
    }
}

public class NamedThreadFactory implements ThreadFactory {
    private static final AtomicInteger poolNumber = new AtomicInteger(1);
    private final ThreadGroup group;
    private final AtomicInteger threadNumber = new AtomicInteger(1);
    private final String namePrefix;

    NamedThreadFactory(String name) {
        SecurityManager s = System.getSecurityManager();
        group = (s != null) ? s.getThreadGroup() : Thread.currentThread().getThreadGroup();
        if (null == name || name.isEmpty()) {
            name = "pool";
        }
        namePrefix = name + "-" + poolNumber.getAndIncrement() + "-thread-";
    }

    @Override
    public Thread newThread(Runnable r) {
        Thread t = new Thread(group, r, namePrefix + threadNumber.getAndIncrement(), 0);
        if (t.isDaemon())
            t.setDaemon(false);
        if (t.getPriority() != Thread.NORM_PRIORITY)
            t.setPriority(Thread.NORM_PRIORITY);
        return t;
    }
}
```

当然上面的方法比较繁琐，最好的办法是有一个专门来设置名字的线程工厂。下面这段代码主要是```Thread thread = Executors.defaultThreadFactory().newThread(r);```这行，其实是一个装饰模式，线程进来之后再过一道默认的线程工厂，获得```Thread```对象，最后我们只是给这个线程设置个名字而已，再返回这个线程，我们自定义的线程工厂只是改个名字而已。
```
public class CustomThreadFactory {

    private static final AtomicLong count = new AtomicLong(0L);

    public static ThreadFactory builder(String poolName, String threadName) {
        return new ThreadFactory() {
            @Override
            public Thread newThread(Runnable r) {
                Thread thread = Executors.defaultThreadFactory().newThread(r);
                thread.setName(poolName + threadName + count.incrementAndGet());
                return thread;
            }
        };
    }
}

@Configuration
public class ThreadPoolConfig {

    @Bean()
    public Executor executor1() {
        ...
        ThreadFactory factory = CustomThreadFactory.builder("Executor1-POOL-","testThread-");
        ...
    }
```

当然了，肯定会有线程的轮子，比如```Google```的```Guava```，```ThreadFactoryBuilder```就是用来给线程池和线程设置名字的。
```
ThreadFactory factory = new ThreadFactoryBuilder().setNameFormat("Executor1-POOL-%d").build();

<dependency>
    <groupId>com.google.guava</groupId>
    <artifactId>guava</artifactId>
    <version>29.0-jre</version>
</dependency>
```