## 一、volatile的介绍和使用
```volatile```是Java中修饰变量的修饰符，它最基本的作用就是"保证一个共享变量在修改之后可以立刻被所有的线程读取到最新的值"，但是实际上```volatile```底层远远不止那么简单，使用不当还是会造成并发问题。

---


## 二、volatile的内存语义
因为Java是会把代码编译成字节码，然后Java虚拟机又会把字节码转换成服务机器能识别执行的机器码，而机器码的执行本质上就是对内存上数据的读写操作，因此```volatile```这种Java内置的关键字在编译成字节码的时候就会生成特定的指令，而这种特定的指令则会执行硬件层面的内存操作从而避免并发问题，这就是内存语义。

#### 1. volatile写的内存语义
在Java中所有的变量都是存放在主内存中，而每个线程都有它的工作内存，工作线程的作用就是当一个线程要使用到主线程中的变量时，它不会直接操作主内存中的变量，而是先把变量拷贝到工作内存，然后每次读写操作都是对工作内存中的拷贝值进行操作，再同步到主内存中，也正是因为这样才有会并发问题。

而```volatile```的写内存语义是每次当线程修改了工作内存中变量的拷贝值后都会强制把工作内存中最新的变量值刷新到主内存中。


#### 2. ```volatile```读的内存语义
```volatile```的读内存语义则是每次当有线程读取这个变量时都会把工作内存中的变量置为无效，这样就只能到主内存去读取变量值，配合写内存语义就能确保每次读取的变量一定是最新的。

---

## 三、volatile的正确使用方式

下面这段测试代码就很好的证明了```volatile```的作用，子线程需要依靠```isStop```变量来判断是否终止子线程中的循环，因为子线程会把这个变量拷贝到自己的工作内存中，最开始```isStop```是```false```，而主线程对```isStop```变量的修改是在子线程执行之后，也就是说主内存虽然已经改为```true```了，但是子线程中的工作内存里还是```false```，子线程就不会终止，但是如果使用了```volatile```就不一样了，一定会读取主内存中的最新值，子线程也就会终止了。

```
public class ```volatile```Test {
    //标记变量，判断是否中断循环
    public static boolean isStop = false;

    public static void main(String[] args) throws Exception {
        Thread thread1 = new Thread(() -> {
            //isStop为true时，终止循环
            while (!isStop) { 
                //do someting
            }
            System.out.println("isStop=true，满足停止条件。" +
                    "停止时间：" + DateTime.now());
        });
        thread1.start();

        //主线程睡眠100毫秒，保证在子线程启动后执行。
        Thread.sleep(100);
        isStop = true;
        System.out.println("主线程设置停止标识 isStop=true。" +
                "设置时间：" + DateTime.now());
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

下面这段代码就是错误的```volatile```使用方式，开启20个线程，并发的递增变量```num```直到2万为止，可以先使用```synchronized```关键字修饰```increase()```方法，最后的结果一定会是2万，但是如果只是用```volatile```关键字则还是会产生并发问题。

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

---

## 四、总结
- ```volatile```只保证了可见性，没有保证原子性，使用的时候一定要注意场景。
- 因为第一条的缘故，```volatile```适合一写多读的场景，因为写没用办法保证原子性，但是可见性可以保证读一定读取到最新值。
