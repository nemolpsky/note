### Eureka集群
Eureka作为微服务架构中比较重要的注册中心来说，肯定也是可以集群部署，这个时候也才更容易了解到Eureka保证AP到底是什么样的概念。


#### 1. 配置文件

单机模拟可以将本地ip设置为三个不同的域名。
```
127.0.0.1 e1
127.0.0.1 e2
127.0.0.1 e3
```

配置文件使用域名来注册，分别有3个配置对应3个节点，分别在不同端口运行3个实例就可以在控制台界面看到其它节点的信息。
```
spring:
  application:
    name: eureka-cluster

---
spring:
  profiles: e1
server:
  port: 8001

# 使用主机名注册，便于单机模拟测试
eureka:
  instance:
    hostname: e1
  client:
    # 是否向注册中心注册自己
    register-with-eureka: false
    # 是否抓取注册表
    fetch-registry: true
    service-url:
      defaultZone: http://e1:8001/eureka,http://e2:8002/eureka,http://e3:8003/eureka


---
spring:
  profiles: e2
server:
  port: 8002

eureka:
  instance:
    hostname: e2
  client:
    register-with-eureka: false
    fetch-registry: true
    service-url:
      defaultZone: http://e1:8001/eureka,http://e2:8002/eureka,http://e3:8003/eureka

---
spring:
  profiles: e3
server:
  port: 8003

eureka:
  instance:
    hostname: e3
  client:
    register-with-eureka: false
    fetch-registry: true
    service-url:
      defaultZone: http://e1:8001/eureka,http://e2:8002/eureka,http://e3:8003/eureka

```

---

#### 2. 同步方式

Eureka集群是没有主从节点之说，每个节点都是平等的可以进行读写，所以任何对注册表的修改都是先在其中一个Eureka节点上进行操作，而后同步到其它节点上。

下面这张图大概描述了同步的过程，其实就是每次有新的客户端注册时都放入队列，等到达到一定数量再批量同步。所以说Eureka是AP，因为这期间不同节点上的注册表数据是可能不一致的，但是能够保证最终一致。

因为每个节点都会对集群其他节点作同步操作，所以Eureka内部注册表维护了对每个实例最后一次更改的时间戳，如果其他节点的时间戳比当前节点内的时间戳更新就表示更新自己的注册表，反之则是更新对方的注册表。

![2](https://github.com/nemolpsky/note/raw/master/file/micro/eureka/eureka/2.png)
