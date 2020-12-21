### Kafka生产者
在Kafka的架构体系中，对生产者最简单最直观的理解就是用来发送消息的，它将各种外部写入的信息发送到各个节点中供消费者使用，但实际上整个过程还有很多操作，比如分区操作，序列化和拦截操作等等，因为spring-kafka是基于kafka-clients.jar实现的，所以这些内部的机制其实都是Spring在其基础上进行包装。

---

#### 1. 发送方式

第一种最简单，发后即忘，不关心发送成功没有，性能最好。
```
kafkaTemplate.send(topic,user);
```

第二种是同步方式，ListenableFuture实现了Future接口，发送完会一直阻塞知道返回一个成功的请求数据，性能最差。
```
ListenableFuture<SendResult> listenableFuture = kafkaTemplate.send(topic,user);
SendResult sendResult = listenableFuture.get();
logger.info(sendResult.toString());
```

第三种是异步方式，成功或失败会调用对应的回调接口，下面两种方式都可以。
```
listenableFuture.addCallback(new SuccessCallback<SendResult>() {
                @Override
                public void onSuccess(SendResult sendResult) {
                    logger.info("send success");
                }
            }, new FailureCallback() {
                @Override
                public void onFailure(Throwable throwable) {
                    logger.info("send failure");
                }
});

listenableFuture.addCallback(new ListenableFutureCallback<SendResult>() {
                @Override
                public void onFailure(Throwable throwable) {
                    logger.info("send failure");
                }

                @Override
                public void onSuccess(SendResult sendResult) {
                    logger.info("send success");
                }
});
```

可以配置重试机制，发送失败了就重试指定次数，注意的是Kafka里面会有两类异常，第一类是网络异常之类的，这个时候会进行重试，因为可能下一次发送网络已经恢复，第二次是例如发送消息过大的异常，这种不会重试，重试也没有用。
```
spring:
    kafka:
        producer:
            retries: 10
```

---

#### 2. 序列化

发送的时候需要把key和value都进行序列化成字节数组才能发送，可以在配置文件配置，也可以在代码中配置。

```
spring:
    kafka:
        producer:
            value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
```
```
@Configuration
public class SerializerConfig {

    @Bean
    public DefaultKafkaProducerFactory pf(KafkaProperties properties) {
        Map<String, Object> props = properties.buildProducerProperties();
        DefaultKafkaProducerFactory pf = new DefaultKafkaProducerFactory(props,new JsonSerializer(),new JsonSerializer());
        return pf;
    }
}
```
---

#### 3. 分区

在发送消息的时候，如果指定了分区，那就会发送给指定的分区，但是如果没有指定分区，并且key也不为空，则会根据key进行哈希计算，确定发送到哪个分区。如果key为null则会以轮询的方式发送给主题内各个可用分区。

Kafka默认的分区策略是，尽量均匀的分布在所有分区，还有其他策略可以选择，或者自定义分区策略。

---

#### 4. 拦截器

Kafka也支持设置拦截器，不过安装Spring的官方文档介绍，拦截器没法使用Spring注入，只能换个思路，自己创建生产者的工厂覆盖默认的，然后添加拦截器。比如修改下上面序列换配置的代码，添加拦截器。

```
@Configuration
public class SerializerConfig {

    @Bean
    public DefaultKafkaProducerFactory factory(KafkaProperties properties) {
        Map<String, Object> producerProperties = properties.buildProducerProperties();
        // 添加拦截器
        producerProperties.put(ProducerConfig.INTERCEPTOR_CLASSES_CONFIG, CustomProducerInterceptor.class.getName());
        DefaultKafkaProducerFactory<?, ?> factory = new DefaultKafkaProducerFactory<>(producerProperties,new JsonSerializer(),new JsonSerializer());
        String transactionIdPrefix = properties.getProducer().getTransactionIdPrefix();
        if (transactionIdPrefix != null) {
            factory.setTransactionIdPrefix(transactionIdPrefix);
        }
        return factory;
    }
}   
```

实现ProducerInterceptor接口，在onSend方法中执行拦截操作。
```
public class CustomProducerInterceptor implements ProducerInterceptor<String, String> {

    private final Logger logger = LoggerFactory.getLogger(KafkaReceiver.class);

    @Override
    public ProducerRecord<String, String> onSend(ProducerRecord<String, String> producerRecord) {
        logger.info("CustomProducerInterceptor onSend!");
        return producerRecord;
    }

    @Override
    public void onAcknowledgement(RecordMetadata recordMetadata, Exception e) {
    }

    @Override
    public void close() {
    }

    @Override
    public void configure(Map<String, ?> map) {
    }
}
```

---

#### 5. 架构设计

前面说了spring-kafka是基于kafka-clients.jar实现的，所以架构设计其实也是指kafka生产者的实现，也是使用Java编写的。设计架构如同下图，整个生产者是由两个线程协调运行，主线程用于前面的创建消息，序列化，分区计算和拦截操作，而真正发消息则会有另一个专门的线程执行。

![BIO](https://github.com/nemolpsky/note/raw/master/file/kafka/images/1.png)

需要注意的是，为了提高效率，不是每创建一条消息就立刻发送，它内部会有一个消息累加器，相当于一个缓存一样，来存放消息，这样每次发送消息都是批量发送。可以通过下面的配置来设置累加器的缓存大小，默认32M。

```
spring:
  kafka:
    producer:
      buffer-memory: 1024
```
消息累加器其实就是一个双端队列，每个分区对应一个双端队列，源码中的声明是```Deque<ProducerBatch>```，注意看ProducerBatch，ProducerRecord才是对应的每条消息，而ProducerBatch则是一批消息，这样做的目的也是为了实现批量发送，而且因为每次发送都要先把一个批次的消息进行序列化转成字节进行传输，这些字节是需要占用内存来进行存储的，而累加器内部会建立BufferPool专门存储这些字节，这样可以复用同一块内存，效率高，但是如果有一个批次的消息超过了缓存池的大小，就会使用另一块内存来存储转换后的字节，这个BufferPool的大小由下面的参数进行配置，默认是16KB。

```
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      batch-size: 1024
```

---

### 6. 获取元数据

最后的一步，就是把这些消息发送到对应的节点上，但是有个问题，如果是集群配置，Kafka怎么知道整个集群的元数据呢？也就是所有节点的地址，然后分别都是什么节点，Leader还是Follow，诸如此类的，Kafka会定期请求集群节点来获取这些元数据，可以通过下列配置来修改间隔时间，默认是5分钟，获取到后存入本地缓存中，注意如果当前发送消息指定的topic在缓存中找不到，也会触发这个请求。

会请求负载最小的节点执行，Kafka内部有记录谁是负载最小的，根据节点当前等待请求响应的数量为准，注意有些配置Spring没有提供配置文件提示，需要在代码中配置，或者在节点上config目录下的producer.properties文件里进行设置。

```
@Bean
public DefaultKafkaProducerFactory factory(KafkaProperties properties) {
    Map<String, Object> producerProperties = properties.buildProducerProperties();
    producerProperties.put(ProducerConfig.METADATA_MAX_AGE_CONFIG,1000);
    ....
}

```