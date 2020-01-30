## ReentrantLock配合Condition

```ReentrantLock```是一个可重入的独占锁，和s```ynchronized```相比功能更加强大，```synchronized```底层是基于JVM虚拟机实现的，而```ReentrantLock```底层则是基于```AQS```(AbstractQueuedSynchronizer，Java中大量的同步API都是基于这个抽象类进行封装)实现的。

1. ReentrantLock基本使用

   初始化的时候可以设置是否为公平锁，true为公平锁，false为非公平锁，默认是非公平锁。

   ```
   ReentrantLock lock = new ReentrantLock(true);
   ```

   ```lock()```方法和```unlock()```方法分别是加锁和解锁，因为它不会自动释放锁，所以使用的时候一定要在finally语句中调用```unlock()```方法。
   
   ```
    try {
        lock.lock();
        // 逻辑代码
    } finally {
        lock.unlock();
    }
   ```

   相比```synchronized```关键字，多了尝试获取锁、设置超时时间，查看重入次数等功能。

   ```
   // 尝试获取锁，返回是否成功结果
   boolean success = lock.tryLock();
   // 尝试获取锁，返回是否成功结果，设置超时时间
   boolean success = lock.tryLock(1,TimeUnit.SECONDS);
   // 统计当前线程获取锁的次数
   int count = lock.getHoldCount();
   ```

2. Condition

   相比于上面的功能，```ReentrantLock```最值得注意的就是```Condition```功能，可以使用```ReentrantLock```锁实例获取一个```Condition```对象，```await()/signal()/signalAll()```三个是最核心的方法，分别是等待、唤醒单个线程和唤醒所有线程，这和```wait()/notify()/notifyAll()```很像，使用方法也很像，必须是获得到锁才可以调用。不过更加灵活的地方是可以new多个```Condition```对象，而每个```Condition```对象只能唤醒调用自己的```await()```方法的那个线程，比如```ConditionA```和```ConditionB```，线程A调用```ConditionA```的```await()```方法，那么必须调用```ConditionA```的```signal()/signalAll()```方法才可以唤醒，调用```ConditionB```的唤醒方法是没效果的。

   ```
   public class ConditionTest {

    public static void main(String[] args) throws Exception{

        Lock lock = new Lock();
        // 开启一个子线程
        new Thread(new MyRunnableA(lock)).start();
        TimeUnit.SECONDS.sleep(1);

        // signalB()无法唤醒子线程，signalB()可以
        // lock.signalB();
        lock.signalA();
    }

   }

   // 线程实现类
   class MyRunnableA implements Runnable{

    private Lock lock = null;

    public MyRunnableA(Lock lock) {
        this.lock = lock;
    }

    @Override
    public void run() {
        System.out.println("线程等待...");
        // 调用ConditionA的await()方法
        lock.awaitA();
        System.out.println("线程唤醒...");
    }
   }

   // 锁封装类
   class Lock {
    private ReentrantLock lock = new ReentrantLock();
    // 初始化两个Condition对象
    private Condition conditionA = lock.newCondition();
    private Condition conditionB = lock.newCondition();

    //调用ConditionA的await()方法
    public void awaitA(){
        try {
            System.out.println("awaitA");
            lock.lock();
            System.out.println("awaitA得到锁");
            conditionA.await();
        }catch (Exception e){
            e.printStackTrace();
        }finally {
            lock.unlock();
        }
    }

    //调用ConditionA的signalAll()方法
    public void singalA(){
        try {
            System.out.println("singalA");
            lock.lock();
            System.out.println("singalA得到锁");
            conditionA.signalAll();
        }catch (Exception e){
            e.printStackTrace();
        }finally {
            lock.unlock();
        }
    }

    //调用ConditionB的await()方法
    public void awaitB(){
        try {
            System.out.println("awaitB");
            lock.lock();
            System.out.println("awaitB得到锁");
            conditionA.await();
        }catch (Exception e){
            e.printStackTrace();
        }finally {
            lock.unlock();
        }
    }

    //调用ConditionB的signalAll()方法
    public void singalB(){
        try {
            System.out.println("singalB");
            lock.lock();
            System.out.println("singalB得到锁");
            conditionB.signalAll();
        }catch (Exception e){
            e.printStackTrace();
        }finally {
            lock.unlock();
        }
    }
   }
   ```

3. 基于ReentrantLock和Condition实现一个简单队列

   原理就是基于上面的```Condition```的等待和唤醒必须基于同一个```Condition```实例，消费者和生产者各有对应的```Condition```来控制等待和唤醒，消费者和生产者之间互相唤醒。

   ```
   public class Test2 {

    public static void main(String[] args) {
        

        // 两个生产者，一个消费者
        Queue queue = new Queue();
        Thread producer1 = new Thread(new Producer(queue));
        producer1.setName("Producer1");
        producer1.start();
        Thread producer2 = new Thread(new Producer(queue));
        producer2.setName("Producer2");
        producer2.start();
        Thread customer = new Thread(new Customer(queue));
        customer.setName("Customer");
        customer.start();
    }
   }
   
   // 队列封装类
   class Queue {
    private int[] arr = new int[5];
    int size = 0;

    // 初始化锁和两个Condition
    private ReentrantLock lock = new ReentrantLock();
    public Condition pCondition = lock.newCondition();
    public Condition cCondition = lock.newCondition();
    public void lock() {
        lock.lock();
    }

    public void unLock() {
        lock.unlock();
    }

    public boolean isEmpty() {
        return size==0;
    }

    public boolean isFull() {
        return size==5;
    }

    public void put(Integer value,String name) throws Exception {

        try {
            lock.lock();
            if (isFull()){
                // 队列满了让生产者等待
                pCondition.await();
            }
            arr[size % 5] = value;
            size++;
            // 生产完唤醒消费者
            cCondition.signalAll();
        } finally {
            System.out.println(name +"-put-" + Arrays.toString(arr));
            lock.unlock();
        }
    }

    public int take() throws Exception {
        try {
            lock.lock();
            // 队列空了就让生产者等待
            if (isEmpty()){
                cCondition.await();
            }
            int value = arr[size % 5];
            size--;
            // 消费完通知生产者
            pCondition.signalAll();
            return value;
        } finally {
            System.out.println("take-" + Arrays.toString(arr));
            lock.unlock();
        }
    }
   }

   class Producer implements Runnable {

    Queue queue = null;

    public Producer(Queue queue) {
        this.queue = queue;
    }

    @Override
    public void run() {
        String threadName = Thread.currentThread().getName();

        try {
            // 隔10秒轮询生产一次
            while (true) {
                System.out.println("Producer");
                TimeUnit.SECONDS.sleep(10);
                queue.put(new Random().nextInt(100),threadName);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
   }

   class Customer implements Runnable {

    Queue queue = null;

    public Customer(Queue queue) {
        this.queue = queue;
    }

    @Override
    public void run() {
        String threadName = Thread.currentThread().getName();
        try {

            // 隔3秒轮询消费一次
            while (true) {
                System.out.println("Customer");
                TimeUnit.SECONDS.sleep(3);
                System.out.println("取到的值-" + queue.take());
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
   }

   ```