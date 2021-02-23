### JDK 命令行工具

#### 1. jps-查看所有Java进程
jps(JVM Process Status)命令类似UNIX的ps命令，显示虚拟机执行主类名称以及这些进程的本地虚拟机唯一ID（Local Virtual Machine Identifier,LVMID）。
- jps -q：只输出进程的本地虚拟机唯一ID。
- jps -l：输出主类的全名，如果进程执行的是Jar包，输出Jar路径。
- jps -v：输出虚拟机进程启动时JVM参数。
- jps -m：输出传递给Java进程main()函数的参数。

---

#### 2.  jstat-监视虚拟机各种运行状态信息
jstat（JVM Statistics Monitoring Tool）使用于监视虚拟机各种运行状态信息的命令行工具。它可以显示本地或者远程（需要远程主机提供RMI支持）虚拟机进程中的类信息、内存、垃圾收集、JIT 编译等运行数据，在没有GUI，只提供了纯文本控制台环境的服务器上，它将是运行期间定位虚拟机性能问题的首选工具。
- jstat -class vmid：显示 ClassLoader 的相关信息
- jstat -compiler vmid：显示 JIT 编译的相关信息
- jstat -gc vmid：显示与 GC 相关的堆信息
- jstat -gccapacity vmid：显示各个代的容量及使用情况
- jstat -gcnew vmid：显示新生代信息
- jstat -gcnewcapcacity vmid：显示新生代大小与使用情况
- jstat -gcold vmid：显示老年代和永久代的信息
- jstat -gcoldcapacity vmid：显示老年代的大小
- jstat -gcpermcapacity vmid：显示永久代大小
- jstat -gcutil vmid：显示垃圾收集信息
- jstat -gc -h3 31736 1000 10：分析进程id为31736的gc情况，每隔1000ms打印一次记录，打印10次停止，每3行后打印指标头部。

---

#### 3. jinfo-实时地查看和调整虚拟机各项参数
- jinfo vmid：输出当前jvm进程的全部参数和系统属性(第一部分是系统的属性，第二部分是JVM的参数)。
- jinfo -flag name vmid：输出对应名称的参数的具体值。
- jinfo  -flag MaxHeapSize 17340：输出MaxHeapSize
- jinfo  -flag PrintGC 17340：查看当前jvm进程是否开启打印GC日志(-XX:PrintGCDetails:详细GC日志模式，这两个都是默认关闭的)。

使用jinfo可以在不重启虚拟机的情况下，可以动态的修改jvm的参数。jinfo -flag [+|-]name vmid 开启或者关闭对应名称的参数。
- jinfo  -flag  +PrintGC 17340：开启参数

---

#### 4. jmap-生成堆转储快照
jmap（Memory Map for Java）命令用于生成堆转储快照。如果不使用jmap命令，要想获取Java堆转储，可以使用“-XX:+HeapDumpOnOutOfMemoryError”参数，可以让虚拟机在OOM异常出现之后自动生成dump 文件，Linux命令下可以通过kill -3发送进程退出信号也能拿到dump文件。jmap的作用并不仅仅是为了获取dump文件，它还可以查询finalizer执行队列、Java堆和永久代的详细信息，如空间使用率、当前使用的是哪种收集器等。和jinfo一样，jmap有不少功能在Windows平台下也是受限制的。
- jmap -dump:format=b,file=C:\Users\SnailClimb\Desktop\heap.hprof 17340

---

#### 5. jhat-分析heapdump文件
jhat用于分析heapdump文件，它会建立一个HTTP/HTML服务器，让用户可以在浏览器上查看分析结果。
- jhat C:\Users\SnailClimb\Desktop\heap.hprof

---

#### 6. jstack-生成虚拟机当前时刻的线程快照
jstack（Stack Trace for Java）命令用于生成虚拟机当前时刻的线程快照。线程快照就是当前虚拟机内每一条线程正在执行的方法堆栈的集合.生成线程快照的目的主要是定位线程长时间出现停顿的原因，如线程间死锁、死循环、请求外部资源导致的长时间等待等都是导致线程长时间停顿的原因。线程出现停顿的时候通过jstack来查看各个线程的调用堆栈，就可以知道没有响应的线程到底在后台做些什么事情，或者在等待些什么资源。

---

#### 7. JConsole
JConsole是基于JMX的可视化监视、管理工具。可以很方便的监视本地及远程服务器的java进程的内存使用情况。你可以在控制台输出console命令启动或者在JDK目录下的bin目录找到jconsole.exe然后双击启动。
- 本地连接
  输入vmid就可以
- 远程连接，输入以下参数
  -Djava.rmi.server.hostname=外网访问 ip 地址 
  -Dcom.sun.management.jmxremote.port=60001   //监控的端口号
  -Dcom.sun.management.jmxremote.authenticate=false   //关闭认证
  -Dcom.sun.management.jmxremote.ssl=false
  
---

#### 8. Visual VM
更加强大的监控工具，可以安装任意的插件，直接在JDK目录下的bin目录中找到jvisualvm.exe打开即可，在1.8之后就被移除了。