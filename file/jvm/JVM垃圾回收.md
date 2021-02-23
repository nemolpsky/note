### 垃圾回收

Java语言和C语言最大的区别就是把内存的建立和释放由JVM来控制，在Java中也称作垃圾回收(GC)，如果对JVM的结构有了解的话，就会知道在JVM中会把堆内存大致的分为新生代和老年代，理解起来也很简单，一般来说创建的对象都会进入新生代 ，在新生代的垃圾回收也被称为Minor GC，经过一次GC后新生代中还存活着的对象年龄就会加1，达到一定阀值之后就会把这个对象放入老年代，所以可以理解为老年代存储的是长期存在的对象，老年代也会触发垃圾回收，被称为Full GC，这种回收会更加耗时，这样分代也是为了提高回收的效率。

还有一个要注意的地方就是，GC不仅仅是单纯的把不再使用的对象清理掉释放内存，还需要整理内存，就像下图所示，最开始内存是空的，但是随着使用和清理之后，整个内存的可用区域会变得不连续，如果想要在内存上连续存储一个数组那就会找不到一块连续可用的区域足够大小来存储，因此GC除了清理对象之外还需要整理内存，把这种间隔的可用内存区域整理成一块连续完整的，这种操作才是最耗时的，而且对象的地址也会跟着变化，所以整理期间这些对象对外是不可用的，这也造成了常说的GC停顿。

![3](https://github.com/nemolpsky/note/raw/master/file/jvm/images/3.png)

实际上JVM中有好几种不同的垃圾回收期，分别有着不同的特性，对应着不同的情况。


---


#### 1. 分代垃圾收集器

- Serial垃圾收集器

  这种垃圾收集器是最早也是最简单的一种，内部是单线程进行GC操作，所以在GC的时候整个程序都会停顿，在Java 8上面已经不在使用这种垃圾收集器了，默认是关闭的，通过下列参数可以看到。

  ```
  root@Half-PC:/opt# java -XX:+PrintFlagsFinal -version | grep "UseSerialGC"
       bool UseSerialGC                               = false                               {product}
  java version "1.8.0_261"
  Java(TM) SE Runtime Environment (build 1.8.0_261-b12)
  Java HotSpot(TM) 64-Bit Server VM (build 25.261-b12, mixed mode)
  ```

- Throughput垃圾收集器

  这种垃圾收集器最大的特点是使用多线程进行GC操作，所以GC的速度会相对来说快很多，停顿时间也会比较短暂，也正以为如此被称为Parallel收集器，表示并行的意思，在Java 8上默认使用这种垃圾收集器。下列两个参数都可以开启或关闭。

  ```
  root@Half-PC:/opt# java -XX:+PrintFlagsFinal -version | grep "UseParallelGC"
     bool UseParallelGC                            := true                                {product}
  java version "1.8.0_261"
  Java(TM) SE Runtime Environment (build 1.8.0_261-b12)
  Java HotSpot(TM) 64-Bit Server VM (build 25.261-b12, mixed mode)
  root@Half-PC:/opt# java -XX:+PrintFlagsFinal -version | grep "UseParallelOldGC"
       bool UseParallelOldGC                          = true                                {product}
  java version "1.8.0_261"
  ```

- CMS收集器

  全程是Concurrent Mark & Sweep GC，它在回收老年代上做了优化，它会定期扫描老年代，然后及时回收对象，虽然最终程序运行过久还是会触发Full GC，但是这种定时扫描清理的操作会大大减少Full GC的触发，当然代价就是这种定时扫描需要耗费CPU，不过总的来说定式扫描时的停顿时间会比一次Full GC快的多。可以使用下列命令开启CMS收集器。

  ```
  root@Half-PC:/opt# java -XX:+PrintFlagsFinal -version | grep "UseParNewGC"
       bool UseParNewGC                               = false                               {product}
  java version "1.8.0_261"
  Java(TM) SE Runtime Environment (build 1.8.0_261-b12)
  Java HotSpot(TM) 64-Bit Server VM (build 25.261-b12, mixed mode)
  root@Half-PC:/opt# java -XX:+PrintFlagsFinal -version | grep "UseConcMarkSweepGC"
       bool UseConcMarkSweepGC                        = false                               {product}
  java version "1.8.0_261"
  Java(TM) SE Runtime Environment (build 1.8.0_261-b12)
  Java HotSpot(TM) 64-Bit Server VM (build 25.261-b12, mixed mode)
  ```

- G1垃圾收集器
   
   全称是Garbage First GC，专门为处理超大堆内存设计的，和其他收集器不同的是，它不是将堆内存分为一块老年代和一块新生代，再把新生代分为一块Eden区域和两块survivor区域那么简单，它会把整个堆分为上百个小的区域，每个区域默认空间是2MB，这些空间有的老年代，有的是survivor，有的是Eden，每次G1垃圾收集器都只会回收一部分区域，把这些区域存活的对象移动到另一些空余的区域，然后清空原来的区域，因为这些区域都很小所以回收的速度会非常快，而且省去了整理不连续内存的操作。在Java 9中已经默认使用G1垃圾收集器了，Java 8可以使用下列参数开启。

   ```
    root@Half-PC:/opt# java -XX:+PrintFlagsFinal -version | grep "UseG1GC"
     bool UseG1GC                                   = false                               {product}
    java version "1.8.0_261"
    Java(TM) SE Runtime Environment (build 1.8.0_261-b12)
    Java HotSpot(TM) 64-Bit Server VM (build 25.261-b12, mixed mode)
   ```
   
   所以这也就是为什么G1垃圾收集器提供了参数可以控制GC停顿时间，它会根据参数设置的时间估算每次清理多少个区域，所以如果堆内存太小就体现不出它的优势，一般需要堆内存大于4G，最终就会演变成和其它垃圾收集器一样的整体GC。

   ```
   root@Half-PC:/opt# java -XX:+PrintFlagsFinal -version | grep "MaxGCPauseMillis"
    uintx MaxGCPauseMillis                          = 18446744073709551615                    {product}
    java version "1.8.0_261"
   Java(TM) SE Runtime Environment (build 1.8.0_261-b12)
   Java HotSpot(TM) 64-Bit Server VM (build 25.261-b12, mixed mode)
   ```

---

#### 2. 调整堆内存

垃圾回收其实就是对堆内存中的对象进行清理，所以堆内存的大小就比较重要了，不同的系统不同的版本下堆内存的默认初始值和最大值都不一样，可以直接使用下列命令查看默认初始大小和最大值，是以字节为单位。

```
root@Half-PC:/opt# java -XX:+PrintFlagsFinal -version | grep InitialHeapSize
    uintx InitialHeapSize                          := 134217728                           {product}
java version "1.8.0_261"
Java(TM) SE Runtime Environment (build 1.8.0_261-b12)
Java HotSpot(TM) 64-Bit Server VM (build 25.261-b12, mixed mode)
root@Half-PC:/opt# java -XX:+PrintFlagsFinal -version | grep MaxHeapSize
    uintx MaxHeapSize                              := 2139095040                          {product}
java version "1.8.0_261"
Java(TM) SE Runtime Environment (build 1.8.0_261-b12)
Java HotSpot(TM) 64-Bit Server VM (build 25.261-b12, mixed mode)
```

因为JVM的机制是，先创建一个初始大小的堆内存，然后随着多次的GC操作之后慢慢进行扩容，所以可以使用下面两个参数来设置初始大小和最大值，设置初始值为1M，最大值为5M。

```
root@Half-PC:/opt/jar# java -Xms1m -Xmx5m -jar JVMDemo-1.0-SNAPSHOT.jar
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
        at MainClass2.main(MainClass2.java:4)
```

如果堆内存设置的太小，代码就会提示OOM异常，比如下面的代码配合上面的参数就会提示上面的错误，但是也不要把堆内存建的太大，还要考虑服务器上其他软件对内存的使用，以及是否还有别的JVM在运行。
```
    public static void main(String[] args) {
        byte[] bytes = new byte[1000 * 1000 * 10];
    }
```

堆内存的自动调整是默认可以关闭的，不过要确保你非常熟悉你的应用，精准的配置好了堆内存，比如下面的命令就是关闭了自动调整和开启自动调整时打印的日志信息，可以发现关闭自动调整就没有那些很长的调整日志了。-XX:-UseAdaptiveSizePolicy是关闭自动调整，-XX:+PrintAdaptiveSizePolicy是打印调整日志。
```
root@Half-PC:/opt/jar# java -Xms1m -Xmx20m -XX:+PrintAdaptiveSizePolicy -jar JVMDemo-1.0-SNAPSHOT.jar
AdaptiveSizePolicy::update_averages:  survived: 344080  promoted: 8192  overflow: false
AdaptiveSizeStart: 0.083 collection: 1
PSAdaptiveSizePolicy::compute_eden_space_size: costs minor_time: 0.009175 major_cost: 0.000000 mutator_cost: 0.990825 throughput_goal: 0.990000 live_space: 268779520 free_space: 1048576 old_eden_size: 524288 desired_eden_size: 524288
AdaptiveSizeStop: collection: 1
AdaptiveSizePolicy::update_averages:  survived: 311312  promoted: 0  overflow: false
AdaptiveSizeStart: 0.085 collection: 2
PSAdaptiveSizePolicy::compute_eden_space_size: costs minor_time: 0.198061 major_cost: 0.000000 mutator_cost: 0.801939 throughput_goal: 0.990000 live_space: 268763136 free_space: 1048576 old_eden_size: 524288 desired_eden_size: 1048576
AdaptiveSizePolicy::survivor space sizes: collection: 2 (524288, 524288) -> (524288, 524288)
AdaptiveSizeStop: collection: 2
AdaptiveSizeStart: 0.090 collection: 3
PSAdaptiveSizePolicy::compute_eden_space_size: costs minor_time: 0.198061 major_cost: 0.040960 mutator_cost: 0.760979 throughput_goal: 0.990000 live_space: 268655008 free_space: 1572864 old_eden_size: 1048576 desired_eden_size: 1048576
PSAdaptiveSizePolicy::compute_old_gen_free_space: costs minor_time: 0.198061 major_cost: 0.040960 mutator_cost: 0.760979 throughput_goal: 0.990000 live_space: 268930464 free_space: 1572864 old_promo_size: 524288 desired_promo_size: 1048576
AdaptiveSizePolicy::old generation size: collection: 3 (14155776) -> (1572864)
AdaptiveSizeStop: collection: 3
AdaptiveSizePolicy::update_averages:  survived: 0  promoted: 0  overflow: false
AdaptiveSizeStart: 0.092 collection: 4
PSAdaptiveSizePolicy::compute_eden_space_size: costs minor_time: 0.145869 major_cost: 0.040960 mutator_cost: 0.813171 throughput_goal: 0.990000 live_space: 268875584 free_space: 2097152 old_eden_size: 1048576 desired_eden_size: 1048576
AdaptiveSizePolicy::survivor space sizes: collection: 4 (524288, 524288) -> (524288, 524288)
AdaptiveSizeStop: collection: 4
AdaptiveSizeStart: 0.096 collection: 5
PSAdaptiveSizePolicy::compute_eden_space_size: costs minor_time: 0.145869 major_cost: 0.304757 mutator_cost: 0.549374 throughput_goal: 0.990000 live_space: 268842656 free_space: 2097152 old_eden_size: 1048576 desired_eden_size: 1048576
PSAdaptiveSizePolicy::compute_old_gen_free_space: costs minor_time: 0.145869 major_cost: 0.304757 mutator_cost: 0.549374 throughput_goal: 0.990000 live_space: 268836224 free_space: 2097152 old_promo_size: 1048576 desired_promo_size: 2097152
AdaptiveSizePolicy::old generation size: collection: 5 (14155776) -> (2621440)
AdaptiveSizeStop: collection: 5
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
        at MainClass2.main(MainClass2.java:4)
root@Half-PC:/opt/jar# java -Xms1m -Xmx20m -XX:+PrintAdaptiveSizePolicy -XX:-UseAdaptiveSizePolicy -jar JVMDemo-1.0-SNAPSHOT.jar
AdaptiveSizePolicy::update_averages:  survived: 360480  promoted: 8192  overflow: false
AdaptiveSizePolicy::update_averages:  survived: 311312  promoted: 0  overflow: false
AdaptiveSizePolicy::update_averages:  survived: 0  promoted: 0  overflow: false
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
        at MainClass2.main(MainClass2.java:4)
```

使用-XX:+PrintGCDetails参数可以打印堆栈信息，新生代和老年代的大小和使用情况，默认情况下新生代和老年代的比例是1/(n+1)，n可以通过-XX:NewRatio参数来设置，默认是2.
```
root@Half-PC:/opt/jar# java -Xms15m -Xmx15m -XX:+PrintGCDetails -jar JVMDemo-1.0-SNAPSHOT.jar
Heap
 PSYoungGen      total 4608K, used 508K [0x00000000ffb00000, 0x0000000100000000, 0x0000000100000000)
  eden space 4096K, 12% used [0x00000000ffb00000,0x00000000ffb7f108,0x00000000fff00000)
  from space 512K, 0% used [0x00000000fff80000,0x00000000fff80000,0x0000000100000000)
  to   space 512K, 0% used [0x00000000fff00000,0x00000000fff00000,0x00000000fff80000)
 ParOldGen       total 11264K, used 9765K [0x00000000ff000000, 0x00000000ffb00000, 0x00000000ffb00000)
  object space 11264K, 86% used [0x00000000ff000000,0x00000000ff989690,0x00000000ffb00000)
 Metaspace       used 2666K, capacity 4486K, committed 4864K, reserved 1056768K
  class space    used 285K, capacity 386K, committed 512K, reserved 1048576K
```
---

#### 3. 调整元空间

元空间是在Java 8中才有的，是用来存放对象的元数据的，以前是对应Java 7的永久代，因为在Java 8中元空间是没有限制大小的，所以如果无限制增长可能会耗尽服务器所有的内存，下面的参数可以配置元空间的初始大小和最大值。
```
root@Half-PC:/opt# java -XX:+PrintFlagsFinal -version | grep "MetaspaceSize"
    uintx MaxMetaspaceSize                          = 18446744073709547520                    {product}
    uintx MetaspaceSize                             = 21807104                            {pd product}
```

---

#### 4. GC线程
上面说到的4种垃圾收集器里，有3种是多线程的，因此合理的设置线程数对GC的效率也很重要，一般来说是默认每个CPU启动一个线程，但是如果数量超过了8个，则会使用这个公式来计算，threads=8 +（（N - 8）* 5/8），N是CPU数量，下面是查看默认线程数。
```
root@Half-PC:/opt# java -XX:+PrintFlagsFinal -version | grep "ParallelGCThreads"
    uintx ParallelGCThreads                         = 4                                   {product}
java version "1.8.0_261"
Java(TM) SE Runtime Environment (build 1.8.0_261-b12)
Java HotSpot(TM) 64-Bit Server VM (build 25.261-b12, mixed mode)
```

所以有时候就需要使用-XX:ParallelGCThreads=1参数来配置线程数，它会对下面这些参数的线程数有影响，不过还是需要从整体服务器考虑情况，如果一个服务器上有多个JVM运行，那么就要考虑多个JVM的线程总值。
```
-XX:+UseParallelGC，收集新生代空间
-XX:+UseParallelOldGC，收集老年代空间
-XX:+UseParNewGC，收集新生代空间
-XX:+UseG1GC，收集新生代空间
CMS收集器的扫描阶段阶段(并非Full GC)
G1收集器的区域清理阶段(并非Full GC)
```


> https://www.dynatrace.com/news/blog/understanding-g1-garbage-collector-java-9/
> https://blog.openj9.org/2020/04/30/default-java-maximum-heap-size-is-changed-for-java-8/
> https://stackoverflow.com/questions/24202523/java-process-memory-check-test