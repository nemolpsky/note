# 目录


## Java基础
- 基础
	- [Java底层基础](file/java_base/java_base.md)
	- [异常体系概述](file/java_base/exception.md)
	- [四种字符串拼接](file/java_base/string_append_performance.md)
	- [Java队列介绍使用](file/java_base/queue.md)
	- [Java IO stream](file/java_base/Java_IO流.md)
- Java集合系列
    - [常用集合概述](file/java_base/collections.md)
    - [ArrayList底层原理](file/java_plus/collections/arraylist.md)
	- [LinkedList底层原理](file/java_plus/collections/linkedlist.md)
	- [HashMap底层原理](file/java_plus/collections/hashmap.md)
	- [ConcurrentHashMap底层原理](file/java_plus/collections/concurrenthashmap.md)
	- [LinkedHashMap底层原理](file/java_plus/collections/linkedhashmap.md)
	- [TreeMap底层原理](file/java_plus/collections/treemap.md)
	- [HashSet底层原理](file/java_plus/collections/hashset.md)
	- [LinkedHashSet底层原理](file/java_plus/collections/linkedhashset.md)
	- [TreeSet底层原理](file/java_plus/collections/treeset.md)
- 函数式编程
	- [Java函数式编程](file/function/Java函数式编程.md)
	- [Lambda表达式使用](file/function/Lambda表达式使用.md)
	
---

## JAVA并发
- Java并发编程之美
    - [Java并发编程之美 第一章 笔记](file/java_thread1/unit1.md)
		- [创建线程的三种方式/等待和唤醒/join()/守护线程/ThreadLocal和InheritableThreadLocal/[ThreadLocal底层原理](file/java_thread1/ThreadLocal.md)]
	- [Java并发编程之美 第二章 笔记](file/java_thread1/unit2.md)
		- [线程安全问题/共享变量内存可见性问题/synchronized/volatile/Unsafe类/Java指令重排序/伪共享/锁的类型]
	- [Java并发编程之美 第三章 笔记](file/java_thread1/unit3.md)
		- [Random类/ThreadLocalRandom类]
	- [Java并发编程之美 第四章 笔记](file/java_thread1/unit4.md)
		- [原子操作类/LongAddr类/LongAccumulator]
	- [Java并发编程之美 第五章 笔记](file/java_thread1/unit5.md)
		- [CopyOnWriteArrayList类]
	- [Java并发编程之美 第六章 笔记](file/java_thread1/unit6.md)
		- [LockSupport类/[AQS原理](file/java_thread1/AQS.md)/[ReentrantLock原理](file/java_thread1/ReentrantLock.md)/[CountDownLatch原理](file/java_thread1/CountDownLatch.md)]
- 线程池
	- [Executors提供的常用线程池介绍](file/java_thread1/ThreadPool_base.md)
	- [自定义线程池详解](file/java_thread1/ThreadPoolExecutor.md)
- Java锁
	- [Synchronized关键字和锁升级](file/java_thread1/synchronized.md)
	- [ReentrantLock配合Condition](file/java_thread1/ReentrantLock2.md)
	- [volatile的介绍和使用](file/java_thread1/volatile的介绍和使用.md)
	- [并发概念以及Java中的并发实现](file/java_thread1/并发概念以及Java中的并发实现.md)
---

## JVM
- [JVM内存区域](file/jvm/jvm_memory.md)
- [Class文件介绍](file/jvm/jvm_class.md)
- [JVM即时编译器](file/jvm/JVM即时编译器.md)
- [JVM垃圾回收](file/jvm/JVM垃圾回收.md)

---

## IO
- [bio和nio介绍](file/io/io.md)
- [bio详解](file/io/bio.md)
- [nio详解](file/io/nio.md)

---

## Spring
- [Spring基本概念](file/spring/spring2.md)
- [Spring IoC容器](file/spring/spring1.md)
- [Spring常用注解](file/spring/spring3.md)
- [Spring常用功能实现](file/spring/spring4.md)
- [Spring AOP配置自定义注解](file/spring/Spring_AOP.md)

---

## Spring Boot
- [Spring Boot CLI介绍使用](file/spring/SpringBootCLI介绍使用.md)
- [Spring Boot自定义配置](file/spring/SpringBoot自定义配置.md)
- [Spring Boot自动配置机制](file/spring/SpringBoot自动配置.md)
- [SpringBoot的starter机制和自动配置原理](file/spring/springboot2.md)
- [SpringBoot启动类run()底层原理](file/spring/springboot1.md)

---

## 分布式
- [分布式锁实现](file/distributed/distributed_lock.md)
- [分布式架构概述](file/distributed/分布式架构概述.md)
- [分布式一致性协议](file/distributed/分布式一致性协议.md)

---

## 微服务组件
- [Hystrix的作用和设计](file/micro/hystrix/Hystrix的作用和设计.md)
- [Hystrix使用](file/micro/hystrix/Hystrix使用.md)

---

## 中间件
- Kakfa
	- [安装配置Zookeeper和Kafka](file/middleware/kafka/安装配置Zookeeper和Kafka.md)
	- [SpringBoot集成Kafka](file/middleware/kafka/SpringBoot集成Kafka.md)
	- [Kafka生产者](file/middleware/kafka/Kafka生产者.md)
	- [Kafka消费者](file/middleware/kafka/Kafka消费者.md)
	- [主题与分区](file/middleware/kafka/主题与分区.md)
	- [Kafka日志存储原理](file/middleware/kafka/Kafka日志存储原理.md)
- Zookeeper
	- [Zookeeper系统模型介绍](file/middleware/zookeeper/Zookeeper系统模型介绍.md)

- Elasticsearch
	- [安装Elasticsearch和Kibana](file/middleware/elasticsearch/安装Elasticsearch和Kibana.md)
---

## Mysql
- [常用命令](file/mysql/mysql_usual_command.md)
- [查询MySQL元数据](file/mysql/查询MySQL元数据.md)
- [数据库范式](file/mysql/数据库范式.md)
- [Mysql锁问题](file/mysql/mysql_lock.md)
- [Mysql explain参数解析](file/mysql/mysql_explain.md)
- [Mysql索引基础](file/mysql/mysql_index1.md)
- [Mysql索引进阶](file/mysql/mysql_index2.md)
- [Mysql优化](file/mysql/mysql_optimize.md)
- [Mysql底层原理](file/mysql/mysql_tree.md)
- [Mysql锁问题](file/mysql/mysql_lock.md)

---

## 容器

- [Docker Desktop的使用](file/container/docker/Docker_Desktop的使用.md)
- [运行Docker容器](file/container/docker/运行Docker容器.md)
- [制作Java Docker镜像](file/container/docker/制作Java_Docker镜像.md)
- [常用Docker镜像配置使用](file/container/docker/常用Docker镜像使用.md)
- [Docker Compose使用](file/container/docker/Docker_Compose使用.md)
- [Docker DeskTop设置宿主机与容器之间的通信](file/container/docker/DockerDeskTop设置宿主机与容器之间的通信.md)

---

## Nginx
- [Nginx的基本配置和使用](file/mysql/mysql_usual_command.md)

---

## Git
- [IEDA配合Git的基本使用](file/git/Git基本使用.md)
- [Git回溯历史版本](file/git/Git回溯历史版本.md)
---

## Linux
- [Linux用户体系](file/linux/Linux用户体系.md) 
- [Linux权限体系](file/linux/Linux权限体系.md)
- [systemctl命令](file/linux/systemctl.md)

---

## HTTP
- [HTTP请求和TCP握手](file/http/HTTP请求和TCP握手1.md)
- [网络基础知识(IP、子网掩码和DNS解析)](file/http/网络基础知识2.md)

---

## 杂项
- [代码整洁之道](file/java_clean/clean.md)