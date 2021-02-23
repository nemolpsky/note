### 定位Java程序CPU使用过高

很多时候代码考虑的不够周全就会在一些特殊情况下耗尽服务器资源，比如一些判断不到位，导致死循环，又或者是对使用场景的预测失误而因为大批量数据导致的大量循环。JVM本身是提供了一下检测工具，当然也有一些集成的比较好的工具，不过最基本的排除方法还是要了解。


---


#### 1. 死循环例子

这里运行一个最简单的例子，死循环打印，跑起来的时候你会看到很明显的CPU增高。
```
public class MainClass {

    public static void main(String[] args) {
        whileTrue();
    }

    public static void whileTrue(){
        while (true){
            System.out.println(1);
        }
    }
}

```

首先肯定是使用最常用的top命令，因为这是Win 10的WSL，所以除了第一个是运行的Java程序，其它的都是系统的进程。
```
top - 16:12:59 up 50 min,  0 users,  load average: 0.52, 0.58, 0.59
Tasks:  15 total,   1 running,  14 sleeping,   0 stopped,   0 zombie
%Cpu(s): 16.7 us, 19.8 sy,  0.0 ni, 63.2 id,  0.0 wa,  0.3 hi,  0.0 si,  0.0 st
MiB Mem :   8156.7 total,   1727.8 free,   6205.0 used,    224.0 buff/cache
MiB Swap:  15158.5 total,  14969.7 free,    188.8 used.   1821.1 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
  485 root      20   0 4455868  56700  12076 S  27.8   0.7   0:09.79 java
    1 root      20   0    8892    308    268 S   0.0   0.0   0:00.09 init
    8 root      20   0    8900    216    168 S   0.0   0.0   0:00.00 init
    9 half      20   0   18080   3644   3556 S   0.0   0.0   0:00.13 bash
   66 root      20   0   18388   2344   2320 S   0.0   0.0   0:00.03 su
   67 root      20   0   17140   2608   2516 S   0.0   0.0   0:00.07 bash
  110 root      20   0    8900    216    168 S   0.0   0.0   0:00.00 init
  111 half      20   0   18080   3632   3544 S   0.0   0.0   0:00.05 bash
  142 root      20   0   18388   2348   2324 S   0.0   0.0   0:00.04 su
  143 root      20   0   17140   2548   2444 S   0.0   0.0   0:00.11 bash
  366 root      20   0   18028   2120   2096 S   0.0   0.0   0:00.01 su
  367 half      20   0   18308   3924   3820 S   0.0   0.0   0:00.10 bash
  407 root      20   0   18388   2376   2348 S   0.0   0.0   0:00.04 su
  408 root      20   0   17140   2580   2472 S   0.0   0.0   0:00.04 bash
  500 root      20   0   18944   2180   1528 R   0.0   0.0   0:00.04 top
```

再使用```top -H -p 485```命令，```-H```是要显示出指定进程下的所有线程。可以看到下面会有很多Java的线程，这很正常，JVM自己GC收集器都会开启多线程，如果是真实项目还有会一些框架的线程池之类的。注意第一个，耗费CPU最高，所以需要查看这个线程对应了哪段代码。

```
top - 16:15:23 up 52 min,  0 users,  load average: 0.52, 0.58, 0.59
Threads:  15 total,   1 running,  14 sleeping,   0 stopped,   0 zombie
%Cpu(s): 15.9 us, 17.4 sy,  0.0 ni, 64.8 id,  0.0 wa,  1.9 hi,  0.0 si,  0.0 st
MiB Mem :   8156.7 total,   1645.3 free,   6287.4 used,    224.0 buff/cache
MiB Swap:  15158.5 total,  14978.1 free,    180.4 used.   1738.7 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
  486 root      20   0 4455868  57044  12044 R  35.3   0.7   1:02.77 java                                                                                      485 root      20   0 4455868  57044  12044 S   0.0   0.7   0:00.00 java
  487 root      20   0 4455868  57044  12044 S   0.0   0.7   0:00.00 java
  488 root      20   0 4455868  57044  12044 S   0.0   0.7   0:00.00 java
  489 root      20   0 4455868  57044  12044 S   0.0   0.7   0:00.00 java
  490 root      20   0 4455868  57044  12044 S   0.0   0.7   0:00.00 java
  491 root      20   0 4455868  57044  12044 S   0.0   0.7   0:00.00 java
  492 root      20   0 4455868  57044  12044 S   0.0   0.7   0:00.00 java
  493 root      20   0 4455868  57044  12044 S   0.0   0.7   0:00.00 java
  494 root      20   0 4455868  57044  12044 S   0.0   0.7   0:00.00 java
  495 root      20   0 4455868  57044  12044 S   0.0   0.7   0:00.01 java
  496 root      20   0 4455868  57044  12044 S   0.0   0.7   0:00.00 java
  497 root      20   0 4455868  57044  12044 S   0.0   0.7   0:00.03 java
  498 root      20   0 4455868  57044  12044 S   0.0   0.7   0:00.00 java
  499 root      20   0 4455868  57044  12044 S   0.0   0.7   0:00.00 java
```

接来下这部比较重要，就是把刚才那个线程的PID转为16进制的，也就是1e6，因为JVM堆栈信息对应的是十六进制的线程id。
```
root@Half-PC:/opt/jar# printf %x 486
1e6
```

最后就是执行```jstack 485 | grep -A 50 1e6```命令，使用JVM自带的工具，打印进程的堆栈信息，再从堆栈信息中查找对应的线程信息，可以看到最下面已经展示的很清楚了，```main```方法的第4行，调用了```whileTrue```方法，后面就是不同的执行IO操作，也就是打印，这样就可以定位到出问题的代码了。
```
root@Half-PC:/opt/jar# jstack 485 | grep -A 50 1e6
"main" #1 prio=5 os_prio=0 tid=0x00007f2334009800 nid=0x1e6 runnable [0x00007f233980f000]
   java.lang.Thread.State: RUNNABLE
        at java.io.FileOutputStream.writeBytes(Native Method)
        at java.io.FileOutputStream.write(FileOutputStream.java:326)
        at java.io.BufferedOutputStream.flushBuffer(BufferedOutputStream.java:82)
        at java.io.BufferedOutputStream.flush(BufferedOutputStream.java:140)
        - locked <0x0000000080811b40> (a java.io.BufferedOutputStream)
        at java.io.PrintStream.write(PrintStream.java:482)
        - locked <0x0000000080808178> (a java.io.PrintStream)
        at sun.nio.cs.StreamEncoder.writeBytes(StreamEncoder.java:221)
        at sun.nio.cs.StreamEncoder.implFlushBuffer(StreamEncoder.java:291)
        at sun.nio.cs.StreamEncoder.flushBuffer(StreamEncoder.java:104)
        - locked <0x0000000080811a40> (a java.io.OutputStreamWriter)
        at java.io.OutputStreamWriter.flushBuffer(OutputStreamWriter.java:185)
        at java.io.PrintStream.write(PrintStream.java:527)
        - eliminated <0x0000000080808178> (a java.io.PrintStream)
        at java.io.PrintStream.print(PrintStream.java:597)
        at java.io.PrintStream.println(PrintStream.java:736)
        - locked <0x0000000080808178> (a java.io.PrintStream)
        at MainClass.whileTrue(MainClass.java:9)
        at MainClass.main(MainClass.java:4)

"VM Thread" os_prio=0 tid=0x00007f233413f000 nid=0x1eb runnable

"GC task thread#0 (ParallelGC)" os_prio=0 tid=0x00007f233401f800 nid=0x1e7 runnable

"GC task thread#1 (ParallelGC)" os_prio=0 tid=0x00007f2334021000 nid=0x1e8 runnable

"GC task thread#2 (ParallelGC)" os_prio=0 tid=0x00007f2334023000 nid=0x1e9 runnable

"GC task thread#3 (ParallelGC)" os_prio=0 tid=0x00007f2334024800 nid=0x1ea runnable

"VM Periodic Task Thread" os_prio=0 tid=0x00007f2334194800 nid=0x1f3 waiting on condition

JNI global references: 5
```