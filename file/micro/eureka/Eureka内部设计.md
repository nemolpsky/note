### Eureka设计介绍
Eureka是专门作为微服务架构中的注册中心来进行设计的，它和Zookeeper不太一样，后者是在CAP中保证CP，所以有时候会面临Zookeeper集群宕机过半选举过程不能使用的情况，或者是把Zookeeper作为分布式锁来使用。而Eureka则是保证AP，所以这种设计最适合作为注册中心，毕竟对于注册中心来说短时间的数据不一致最好过整个集群不可用。


---


#### 1. 服务注册、续约和移除
其实Eureka可以理解为管理了整个微服务群的元数据，ip地址、端口、服务名和服务状态等，这样在上千个微服务运行在不同的服务器上时就可以直接从Eureka中获取这些服务的地址再使用通讯协议进行交互，所以本质上Eureka就是一张注册表。


所以在配合Spring Cloud使用Eureka中在给每个微服务启动类上使用注解再配置好Eureka服务的地址就可以把当前这个微服务变成一个Eureka客户端，这个时候Eureka才会维护这个客户端的元数据，也就是所谓的注册。

```
@EnableDiscoveryClient
public class ClientApplication {
    ...
}

eureka:
  client:
    service-url:
      defaultZone: http://localhost:10001/eureka/
```

既然能够注册那肯定也会有移除操作，比如客户端下线或者宕机，这个时候就需要该客户端的元数据从注册表中移除。所以就需要定时监测每个客户端的状态，在Eureka中会使用心跳机制来实现，也就是客户端定期请求Eureka服务，默认是30秒，Eureka会维护每个客户端的最后请求时间，每次请求，也就是所谓的续约，都会刷新这个时间，超过一定时间就会认为这个客户端不可用，也就会把它从注册表移除。


此外客户端正常停止也会从注册表中移除，Eureka也是使用定时任务执行移除操作，值得借鉴的是Eureka会记录每次执行任务的时间，这样可以计算出是否多消耗时间，这种情况是可能发生的，比如中途JVM的GC操作，下一次判断客户端实例是否过期需要加上这部分多消耗的时间，这样就可以保证定时任务的精确性。

```
// 当前时间
long currNanos = getCurrentTimeNano();
// 上次执行任务时间
long lastNanos = lastExecutionNanosRef.getAndSet(currNanos);
// 第一次执行直接返回0
if (lastNanos == 0L) {
    return 0L;
}
// 两次任务实际执行的间隔时间
long elapsedMs = TimeUnit.NANOSECONDS.toMillis(currNanos - lastNanos);
// 实际间隔时间 - 配置间隔时间 = 多余时间
long compensationTime = elapsedMs - serverConfig.getEvictionIntervalTimerInMs();

// 如果，当前时间 > 实例最后更新时间 + 续约周期(90秒) + 补偿时间，表示该客户端需要被清理
```

整个移除操作是会把要移除的客户端放入列表，然后分批随机移除，避免一次性移除多个客户端。除此之外还要注意的是Eureka会有保护机制，简单理解的话可以认为是短时间内15%的客户端都失效时有可能是自身网络出现问题，而不是这些客户端实例宕机了。

```
// 整个注册表实例数量
int registrySize = (int) getLocalRegistrySize();
// 注册表数量保留的阈值，已注册的实例数 * 续约百分比阈值（默认0.85）
int registrySizeThreshold = (int) (registrySize * serverConfig.getRenewalPercentThreshold());
// 剔除数量，最多只能剔除15%的实例，因为默认保留阀值是85%
int evictionLimit = registrySize - registrySizeThreshold;
// 最小剔除数量
int toEvict = Math.min(expiredLeases.size(), evictionLimit);

```



---

#### 2. 拉取注册表

上面说了Eureka会维护一个注册表，这个注册表本质上就是为了给每个客户端使用，让它们直接获取其它客户端的信息，这样每个客户端之间的调用就很简单，配合服务名和负载均衡策略就可以实现。

所以每个客户端启动时不仅仅注册自己到Eureka，还会从Eureka那里获取一份注册表信息存放在客户端本地缓存中，这也是Eureka保证AP的一种设计，如果客户端本地的缓存里已经有数据了，Eureka服务不可用也不会影响到客户端之间的调用，最多就是注册表数据不能够保证是最新的。

Eureka为了性能在注册表数据结构这块做了一些优化，使用[多级缓存](https://github.com/nemolpsky/note/blob/master/file/micro/eureka/Eureka%E5%86%85%E9%83%A8%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84.md)的方式来实现。

