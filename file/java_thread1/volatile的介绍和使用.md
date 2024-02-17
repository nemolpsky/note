### 一、volatile的介绍和使用
```volatile```是Java中修饰变量的修饰符，它最基本的作用就是"保证一个共享变量在修改之后可以立刻被所有的线程读取到最新的值"，但是实际上```volatile```底层远远不止那么简单，使用不当还是会造成并发问题。

---

### 二、volatile解决的问题
一般底层硬件中CPU负责执行代码转换后的机器码，而数据则暂时存放在内存中，不过读写内存中的数据相对CPU执行指令的速度要慢很多，所以一般在CPU和内存之间会有寄存器来缓存少量数据，寄存器的读写要远远高于内存，也算是变相的提高了CPU的执行效率，但是这也会造成内存和寄存器之间的数据不一致。硬件层面有MESI协议来保证缓存一致。

而JVM为了统一兼容在不同的硬件上对内存的访问，也建立了一套Java内存模型，简称JMM，也就是Java Memory Model，这个模型规定所有的线程都必须拥有自己的工作内存，因为JVM虚拟机在运行的时候会有很多区域，比如堆，栈和方法区等区域，堆和方法区都是所有线程共享的，比如实例化一个对象，这个对象就会放在堆中，所有线程都可以访问堆中的对象，这里为了方便描述，把这种可以共享访问的区域都称为主内存。

那么问题来了，JMM规定线程不能直接访问主内存里的数据，每个线程自己配一个工作内存，先把主内存的值复制到自己的工作内存再进行运算，运算完成后再把最新的值修改回主内存，这就导致Java中的多线程并发问题了。

而```volatile```则可以部分解决这种问题。


### 三、volatile的内存语义
Java会把代码编译成字节码，然后JVM又把字节码转换成底层硬件能识别执行的机器码，而机器码的执行本质上就是对内存上数据的读写操作，因此内存语义就是指代码最终对硬件操作的逻辑。而volatile最终可以实现数据的可见性，也就是下面的方法，写会立刻强制刷新主内存并且让别的线程的工作内存中的数据过期，读则判断工作内存中是否国企，如果过期就只读主内存的数据，确保数据一致。

#### 1. volatile写的内存语义

```volatile```的写内存语义就是每次当线程修改了自己工作内存中变量的拷贝值后都会强制把工作内存中最新的变量值刷新到主内存中，这也就是上面说的保持可见性。


#### 2. volatile读的内存语义
```volatile```的读内存语义则是每次当有线程读取这个变量时都会把工作内存中的变量置为无效，这样就只能到主内存去读取变量值，配合写内存语义就能确保每次读取的变量一定是最新的。


### 四、禁止指令重排
指令重排这种概念如果有了解过JVM的即时编译概念就会知道其实就是JVM对代码作的一种优化，比如下面这个例子，顺序变了但是结果一致，JVM就会做出类似的重排操作，目的就是为了优化代码执行效率，```volatile```会禁止这种操作，也就是保证了先行发生原则(Happens-Before)，本质上也就是为了保证可见性。

```
// 原始代码
int a = 1;
int b = 2;
a++;
b++;

// 重排后
int a = 1;
a++;
int b = 2;
b++;
```

---

### 四、volatile的使用方式

下面这段代码如果不适用```volatile```会小概率导致死循环，子线程需要依靠主线程中的```isStop```变量来判断是否终止子线程中的循环，子线程会把变量拷贝到自己的工作内存中，但是修改操作是由最开始```isStop```是```false```，如果主线程把```isStop```变量修改后没有把自己工作内存中的值更新到主内存中，那子线程中的工作内存里还是```false```，子线程就不会终止，但是如果使用了```volatile```就不一样了，一定会读取主内存中的最新值，子线程也就会终止了。

```
public class Test {
    //标记变量，判断是否中断循环
    public boolean isStop = false;
//    public volatile boolean isStop = false;

    public static void main(String[] args) throws Exception {
        Test test = new Test();
        test.run();
    }

    public void run() throws InterruptedException {
        Thread thread1 = new Thread(() -> {
            //isStop为true时，终止循环
            while (!isStop) {
                System.out.println("子线程任务运行");
            }
            System.out.println("isStop=true，满足停止条件。" +
                    "停止时间：" + LocalDateTime.now());
        });
        thread1.start();

        //主线程睡眠100毫秒，保证在子线程启动后执行。
        TimeUnit.MILLISECONDS.sleep(100);
        isStop = true;
        System.out.println("主线程设置停止标识 isStop=true。" +
                "设置时间：" + LocalDateTime.now());
        TimeUnit.HOURS.sleep(1);
    }
}
```


下面这段代码，在实际中很常见，判断连接是否开启，及时其他的工作线程在连接开启之后再调用```connect()```方法读取```isConnected```变量来判断连接是否开启依然可能会产生问题，因为可能这个变量修改后还没有更新到主内存中，工作线程已经把变量拷贝到自己的工作内存中了，因此这个使用```volatile```修饰就不会有这种问题。


```
class WebSocketClient {
    volatile boolean isConnected = false;
    
    public void connect() {
        // ... do connect
        if (success) {
            isConnected = true;
        }
    }
    
    public void disconnect() {
        isConnected = false;
    }
}
```

下面这段代码就是错误的```volatile```使用方式，开启20个线程，并发的递增变量```num```直到10000为止，可以先使用```synchronized```关键字修饰```increase()```方法，最后的结果一定会是10000，但是如果只是用```volatile```关键字则还是会产生并发问题。

问题其实就出在```increase()```方法上，递增的过程不是原子性的，比如```num=1```，A线程执行```num++```，把```num```修改为2，然后A还没有把工作内存中的值同步到主内存，B线程抢到执行权，又拿到了主内存中的```num```值，也是1，并且自增到2，这样就会造成最后加的结果不一致，所以这也是为什么说```volatile```只能保持可见性，但是不能保持原子性的缘故。

```
public class ThreadTest {
    public static ```volatile``` int num = 0;

    public static void increase() {
        num++;
    }

    public static void main(String[] args) throws InterruptedException {
        for (int i = 0; i < 20; i++) {
            Thread thread = new Thread(new Runnable() {
                @Override
                public void run() {
                    for (int i = 0; i < 10000; i++) {
                        increase();
                    }
                }
            });
            thread.start();
        }

        TimeUnit.SECONDS.sleep(5);

        System.out.println(num);
    }
}
```

### 五、volatile的原理和实现机制
使用volatile修饰的代码，底层编译时会生成一个内存屏障，这个屏障就实现上述说的读写内存语义，也会禁止指令重排序。

---

### 总结
- ```volatile```只保证了可见性，没有保证原子性，使用的时候一定要注意场景。
- ```volatile```禁止指令重排，保证先行原则，避免因为JVM的一些即时编译优化引起并发问题。
- 因为第一条的缘故，```volatile```适合一写多读的场景，因为写没用办法保证原子性，但是可见性可以保证读一定读取到最新值。
