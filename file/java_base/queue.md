## 队列

队列是一种先进先出的数据结构，所以才被称为队列，因为就像排队一样，但是也有一些队列实现了双端队列，可以分别从两头出队。最底层的接口Queue也是和Set、List接口一样继承Collection接口，下面分别有三个子接口，Dequeue、BlockingQueue和AbstarctQueue，下面的实现类分别对应了双端队列、阻塞队列和非阻塞队列。

---

### 阻塞队列

  阻塞队列顾名思义是阻塞性质的，在入队的时候发现队列满了就会一直阻塞在那里，直到队列里面有元素出队空出位置位置，出队的时候也是一个原理，如果发现队列空了，就会一直阻塞直到有元素入队了，就会把这个入队的元素进行出队操作，总而言之就是出队和入队操作如果无法执行，就会一直阻塞在那里直到可以执行为止。

  - ArrayBlockingQueue

    底层使用数组实现的，默认不是一个公平队列(根据阻塞的先后顺序来访问队列)，不过可以在初始化的时候设置为公平队列，是一个有界队列，可以设置队列大小，例如```ArrayBlockingQueue<Integer> Queue = new ArrayBlockingQueue<Integer>(3,true)```。

    下面这个小例子就是阻塞队列的一个很好的应用，初始化一个长度为3的队列，在main方法里面启动一个子线程，子线程里面调用一个入队操作的方法，间隔时间是2秒一次，而又调用了一个arrayBlockingQueue()方法直接填满了队列，然后循环出队，时间间隔3秒。会发现一次打印1、2、3之后就开始打印后面入队的随机数，因为出队的间隔时间是3秒，而入队的间隔时间是2秒，而一开始塞满了队列，但是做入队操作也没有报错，这就是阻塞队列的特性，如果满了就一直阻塞住，整个过程就像排队一样，依次入队，依次出队。

    ```
    static ArrayBlockingQueue<Integer> arrayBlockingQueue = new ArrayBlockingQueue<Integer>(3);

    public static void main(String[] args) throws Exception {

        System.out.println(arrayBlockingQueue.size());

        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    addQueue();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }).start();

        arrayBlockingQueue();

    }


    public static void arrayBlockingQueue() throws Exception {
        arrayBlockingQueue.put(1);
        arrayBlockingQueue.put(2);
        arrayBlockingQueue.put(3);

        while (true) {
            TimeUnit.SECONDS.sleep(3);
            System.out.println(arrayBlockingQueue.size() + "-" + arrayBlockingQueue.take());
        }
    }

    public static void addQueue() throws Exception {
        while (true) {
            TimeUnit.SECONDS.sleep(2);
            arrayBlockingQueue.put(new Random().nextInt(100));
        }
    }
    ```

  - LinkedBlockingQueue

    这个阻塞队列是基于链表实现的，也是一个有界队列，不过有默认大小，默认大小和最大长度都是Integer.MAX_VALUE。和ArrayBlockingQueue很相似，只不过底层控制并发阻塞的原理稍稍不同

  - PriorityBlockingQueue

    这个队列也是基于数组实现，但是数据结构是一颗完全二叉树，存在数组中，值得注意的是这个队列是无界队列，虽然初始化的时候可以设置初始值，但是一旦队列的长度超过的时候就会自动扩容，最重要的是这个队列是一个优先级队列，所以并不是先进先出(FIFO)的顺序，而是按照元素的优先级来进行出队，所以初始化的时候可以设置一个比较器，确保存入的元素是可以比较的，这样就可以根据元素的优先级进行出队了。

    ```
    public static void priorityBlockingQueue() throws Exception{
        PriorityBlockingQueue<Integer> queue = new PriorityBlockingQueue<Integer>(5,new Comparator<Integer>() {
            @Override
            public int compare(Integer o1, Integer o2) {
                return o2-o1;
            }
        });


        queue.add(10);
        queue.add(2);
        queue.add(5);

        while(true){
            System.out.println(queue.take());
        }
    }
    ```
  
  - DelayQueue

    这个队列也是一个优先级队列，但是它的优先级是以时间为准，放入队列中的元素一定要是Delayed类型，里面有两个方法，分别是compareTo()和getDelay()方法，第一个是设置比较规则，第二个则是判断时间是否已经到达，只有按照getDelay()方法中的代码判断时间已经到达了，这个时候获取队列尾部的元素才可以成功获取到，否则即使有元素也获取不到，也是一个无界队列，底层也是基于数据实现。

    下面这个小例子模仿了订单超时的操作，超时时间设置的各有不同，可以发现，根据对比规则先把最早超时的那个订单出队了，依次出队，直到最晚超时的那个订单。

    ```
    public static void delayQueue() throws Exception{
        DelayQueue<TestDelayed> queue = new DelayQueue<TestDelayed>();

        long time = System.currentTimeMillis()+10000;

        queue.put(new TestDelayed(1,time + 500));
        queue.put(new TestDelayed(2,time + 300));
        queue.put(new TestDelayed(3,time + 100));
        queue.put(new TestDelayed(4,time + 200));
        queue.put(new TestDelayed(5,time + 400));


        while (true){
            TestDelayed delayed = (TestDelayed)queue.take();
            System.out.println(delayed.orderNumber);
        }
    }

    // 自定义Delayed类
    class TestDelayed implements Delayed{
        // 订单号
        int orderNumber;
        // 超时时间
        long expire;

        public TestDelayed(int orderNumber, long expire) {
            this.orderNumber = orderNumber;
            this.expire = expire;
        }

        // 超时时间减去当前时间，如果小于0就表示时间到了
        @Override
        public long getDelay(TimeUnit unit) {
            return expire-System.currentTimeMillis();
        }

        // 设置对比规则
        @Override
        public int compareTo(Delayed o) {
            long t1 = this.getDelay(TimeUnit.MILLISECONDS);
            long t2 = o.getDelay(TimeUnit.MILLISECONDS);
            return Long.compare(t1,t2);
        }
    }
    ```
  
  - SynchronousQueue

    这个队列非常有意思，它自身并没有任何容量，所以每次入队操作之后就必须进行一次出队操作，否则就会阻塞，底层是使用CAS来保证并发问题，适合调度派发任务的场景，生产者源源不断的做入队操作，多个消费者则进行出队操作，它也支持公平队列的设置，在初始化的时候可以进行设置，好处就是在一些调度任务频繁但是又不像使用无界队列就可以使用SynchronousQueue，而且性能更加好。

    ```
    public static void synchronousQueue() throws Exception{
        SynchronousQueue<Integer> queue = new SynchronousQueue<Integer>();

        new Thread(new Runnable() {
            @Override
            public void run() {
                while (true) {
                    try {
                        TimeUnit.SECONDS.sleep(1);
                        queue.put(new Random().nextInt(100));
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }
            }
        }).start();


        while (true){
            TimeUnit.SECONDS.sleep(1);
            System.out.println(queue.take());
        }
    }
    ```

- LinkedTransferQueue

  这是一个很特殊的无界阻塞队列，底层基于链表实现，在JDK7的时候新增的，相比于上面的阻塞队列，它使用CAS和自旋这些无锁算法来保证并发安全问题，所以性能更好，默认就是一个公平队列，而且它更加灵活，可以控制是阻塞线程还是直接把元素传递给等待的消费的线程。

  而它的新特性都是因为新实现的一个接口，TransferQueue接口，所以新特性都是在这个接口里面的方法来实现。

  ```
  // 尝试直接把元素传递给等待的消费者，成功返回true，失败返回false，非阻塞
  boolean tryTransfer(E e);
  // 和上面的方法类似，但是会阻塞
  void transfer(E e) throws InterruptedException;
  // 和第一个方法一样，增加了超时等待时间
  boolean tryTransfer(E e, long timeout, TimeUnit unit)
        throws InterruptedException;
  // 判断是否有等待的消费者，即调用了take()方法的线程
  boolean hasWaitingConsumer();
  // 获取等待的消费者的数量
  int getWaitingConsumerCount();
  ```

  已经有很专业权威的性能测试对比过LinkedTransferQueue和SynchronousQueue的性能了，分别是3倍(非公平模式)和14倍(公平模式)，所以底层的原理和实现也是相当复杂。这里只做简单介绍。


  ```
    public static void linkedTransferQueue() throws Exception{
        LinkedTransferQueue<Integer> queue = new LinkedTransferQueue<Integer>();

        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    while (true) {
                        System.out.println("thread");
                        TimeUnit.SECONDS.sleep(1);
    //                  queue.transfer(3);
    //                  System.out.println(queue.tryTransfer(3));

                        System.out.println(queue.hasWaitingConsumer());
                        System.out.println(queue.getWaitingConsumerCount());
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }).start();

        // 开启5个子线程获取元素
        for(int i=0;i<5;i++){
            new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        queue.take();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }).start();
        }
    }

  ```

---

### 非阻塞队列

与阻塞队列对应的就是非阻塞队列，就是出队和入队如果执行失败也不会阻塞队列，会直接抛错或者返回空值。

- ConcurrentLinkedQueue

  底层基于链表的无界队列，使用CAS来保证并发安全，因为没有使用锁，所以size()方法可能不准确，因为可能同时做了入队和出队操作。



---

### 双端队列

顾名思义就是不一定是先进先出(FIFO)，也可能是先进后出(FILO)，出队操作在首尾都可以进行。

- LinkedBlockingDeque

  这个队列就是一个双端阻塞队列，底层是由双向链表实现的，可以指定队列大小，默认长度是Integer.MAX_VALUE。



---


### 出队和入队操作
在队列中出队和入队基本都有三个方法，这三个方法是有不同的效果。

- put()/add()/offer()区别
  
  - put()

    如果队列满了则会阻塞当前线程，直到队列有空间能够放入为止。

  - add()

    如果入队操作成功则返回true，否则抛出异常。

  - offer()

    入队操作成功返回true，否则返回false。

- poll()/remove()/take()区别

  - take()
    
    如果队列为空则会阻塞，直到有元素可以进行出队操作为止。

  - remove()

    如果出队操作失败则抛出异常。

  - poll()
    
    队列为空则返回null，可以设置等待超时时间。
