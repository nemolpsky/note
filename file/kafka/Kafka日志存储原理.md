### Kafka日志存储原理

Kafka独有的特点就是使用日志持久化消息，所以这也是为什么Kafka可以回溯消费消息，同时Kafka还有分区，副本的概念，所以整个Kafka的日志持久化设计的还是相当的复杂。


---

#### 1. 日志类型

在Kafka中，每个分区都会对应一个日志，存放在log.dirs参数下配置的目录中，比如下列就是日志目录下，部分日志文件。

```
root@Half-PC:/opt/tmp/kafka-logs# ll
...
drwxr-xr-x 1 root root  512 Dec 12 11:23 testTopic7-0/
drwxr-xr-x 1 root root  512 Dec 12 19:55 testTpoic1-0/
```

因为一个分区中的消息数可能会很大，还要引入了日志分段的概念，其实就是超过一定大小，就新建一个日志来存储消息，注意每个日志还会有两个索引文件，偏移量索引和时间索引，可以看到名字比较怪，其实是以日志段中第一个消息的偏移量来命名，方便查找，注意看下面两次查询，其中log的文件大小有增加，打开看也可以看到里面的消息内容。

```
root@Half-PC:/opt/tmp/kafka-logs/testTopic1-0# ll
total 20480
drwxr-xr-x 1 root root      512 Dec 13 10:31 ./
drwxr-xr-x 1 root root      512 Dec 13 10:32 ../
-rw-r--r-- 1 root root 10485760 Dec 13 10:31 00000000000000000000.index
-rw-r--r-- 1 root root        0 Dec 13 10:31 00000000000000000000.log
-rw-r--r-- 1 root root 10485756 Dec 13 10:31 00000000000000000000.timeindex
-rw-r--r-- 1 root root        8 Dec 13 10:31 leader-epoch-checkpoint
root@Half-PC:/opt/tmp/kafka-logs/testTopic1-0# ll
total 20480
drwxr-xr-x 1 root root      512 Dec 13 10:31 ./
drwxr-xr-x 1 root root      512 Dec 13 10:32 ../
-rw-r--r-- 1 root root 10485760 Dec 13 10:31 00000000000000000000.index
-rw-r--r-- 1 root root      340 Dec 13 10:32 00000000000000000000.log
-rw-r--r-- 1 root root 10485756 Dec 13 10:31 00000000000000000000.timeindex
-rw-r--r-- 1 root root        8 Dec 13 10:31 leader-epoch-checkpoint
root@Half-PC:/opt/tmp/kafka-logs/testTopic1-0# cat 00000000000000000000.log
�U��$vY�8�vY�8�����������������{"username":"David","password":"12345","birthday":1607826749567,"age":22}__TypeId__com.lp.jms.User��6�%vY�?�vY�?�����������������{"username":"David","password":"12345","birthday":1607826751412,"age":22}__TypeId__com.lp.jms.Userroot@Half-PC:/opt/tmp/kafka-logs/testTopic1-0#
```

满足日志分段的要求有下列几点，满足任意一点就可以出发分段，也可以配置参数来设置。

```
// 每个日志段的大小，默认10737418241，也就是1GB
log.segment.bytes

// 当前时间戳和日志段中最大的时间戳只差，大于这两个时间，ms的优先级要高
log.roll.hours
log.roll.ms 

// 索引文件大小大于10M
log.index.size.max.bytes

// 追加消息的偏移量，大于日志段的基准偏移量，也就是第一条消息的偏移量
```
---

#### 2. 索引文件

索引文件并不会给每个消息都建立索引，那样数量太大了，它是依靠消息大小来触发的。

```
loq.index.interval.bytes // 默认4KB，写入超过4KB的消息，会建立一个索引
```

偏移量索引是存储了相对偏移量和在文件段中的物理位置，这样可以快速的读取消息。相对偏移量就是相对于该日志段的基准偏移量，比如总偏移量是100，有两个日志段，第一个是0-60，第二个则是61-120，那么总偏移量100的消息应该是第二个日志段上的相对偏移量为39。时间索引则是存储着当前日志分段的最大时间戳和时间戳对应的消息的相对偏移量，定位到消息的索引之后，再使用偏移量索引查询消息。

因为Kafka所有日志消息都是追加形式写的，也就是按顺序写的，递增形式写的，所以查找索引都是按照二分算法来查询，同时还会把索引文件加载到内存中来查找，效率会更快。


---


#### 3. 磁盘持久化

Kakfa使用文件读写来持久化消息，确能保证吞吐量，这是因为它的消息都是按顺序追加的，所以对磁盘的写也是按顺序写，这和内存的随机写效率其实是相当的。

在文件的I/O读写上，Kafka也有很多优化方式，比如页缓存，这是系统的一种缓存机制，可以简单的理解为把磁盘的数据缓存到内存中来读取，都效率会更快，而且相比在程序中自己缓存，整个程序都可以共享，效率更好，也更加节省内存。

最后就是经常说到的零拷贝，如果使用C语言的函数来从操作系统面上讲，调用read()读取存放到内核中的Buffer，再调用write()将Buffer中的内容写入Socket，可以分成下面四个步骤，也就是4次复制，之所以是下面的步骤，涉及到了Linux操作系统的设计，这也是为什么Kafka建议在Linux系统上使用它，所谓零拷贝就是略过内核模式到用户模式的步骤，直接让内核把读取到的磁盘数据发送给Socket。

依赖C语言中的sendfile()函数，对应Java中FileChannel. transferTo()的方法来实现。


- read()函数，将文件内容复制到内核模式下的Read Buffer中。
- CPU将内核模式数据复制到用户模式
- write()函数，将用户模式下的内容复制到内核模式的Socket Buffer中。
- 再把内核模式下的内容复制到网卡设备，进行网络传送