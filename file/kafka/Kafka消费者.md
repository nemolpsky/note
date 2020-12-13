### Kafka消费者

Kafka消费者就是专门用来消费消息的，对应的是KafkaConsumer类。

---

#### 1. 消费者与消费组

消费者很容易理解，就是设置消费者订阅对应的主题，这样该主题有消息的时候，消费者就会拉取消息，拉取的方式确保不会因为消息积压问题直接把消费者压垮。

而消费组是Kafka一个特有的逻辑概念，顾名思义就是将消费者分组，这样的目的是为了灵活的设置一份消息是否被多个消费者消费，比如将消费者A和消费者B分到第一组中，消费者C分到第二组。如果第一组和第二组都订阅了Test主题，那么发送一条消息到Test主题时，第一组和第二组都会收到这条消息，但是第一组中有两个消费者，A和B，它们两个之中只会有一个收到这条消息。

```
    @KafkaListener(id = "testGroup1", topics = "testTopic1",groupId = "t1")
    public void listen1(ConsumerRecord<String, Object> record) {
        logger.info("listen1 received: " + ((User) record.value()).toString());
    }

    @KafkaListener(id = "testGroup3", topics = "testTopic1",groupId = "t2")
    public void listen3(ConsumerRecord<String, Object> record) {
        logger.info("listen3 received: " + ((User) record.value()).toString());
    }

    @KafkaListener(id = "testGroup4", topics = "testTopic1",groupId = "t2")
    public void listen4(ConsumerRecord<String, Object> record) {
        logger.info("listen4 received: " + ((User) record.value()).toString());
    }
```

还有一个很重要的问题，就是消费者与分区之间的关系，Kafka的主题是允许有多个分区，分区可以暂时理解为把同一个主题下的消息存放在不同的地方，然后分区各自发送消息出去，这也是为什么主题下的消息不保证有序性，因为无法在多个分区的情况下保证整体的有序性，只能保证分区的有序性。如果一个消费组订阅了一个主题，而这个主题又有多个分区，那么这些分区会分配给消费组中不同的消费者，但是如果主题的分区超过了消费组中消费者的数量，比如有32个分区，只有16个消费者，那么会有16个消费者收不到消息，这点一点要注意。

---

#### 2.监听主题和分区

使用@KafkaListener注解可以很方便的设置要监听的主题和分区，不过要注意，如果确保监听的分区是存在的。

```
    @KafkaListener(id = "testGroup1", topics = "testTopic1")
    public void listen1(ConsumerRecord<String, Object> record) {
        logger.info("listen1 received: " + ((User) record.value()).toString());
    }

    // testTopic2中的分区0和1，testTopic3中的分区2
    @KafkaListener(id = "testGroup2", topics = {"testTopic2", "testTopic3"},
            topicPartitions = {
            @TopicPartition(topic = "testTopic2", partitions = {"0", "1"}), @TopicPartition(topic = "testTopic3", partitions = "2")
            })
    public void listen2(ConsumerRecord<String, Object> record) {
        logger.info("listen2 received: " + ((User) record.value()).toString());
    }
```

---

#### 3.反序列化

前面生产者是支持序列化操作的，消费者自然也支持反序列化，这样就可以直接转换成对应的对象类型。

```
    // 设置反序列化方式，反序列化要添加对应的类包名到信任名单中
    @Bean
    public DefaultKafkaConsumerFactory consumerFactory(KafkaProperties properties) {
        Map<String, Object> consumerProperties = properties.buildConsumerProperties();
        JsonDeserializer<Object> valueDeserializer = new JsonDeserializer<>();
        valueDeserializer.addTrustedPackages("com.lp.jms");
        DefaultKafkaConsumerFactory<?, ?> factory = new DefaultKafkaConsumerFactory<String,Object>(consumerProperties,
                new StringDeserializer(),
                valueDeserializer);
        return factory;
    }

    // 直接将 value转换成对象，不需要再进行JSON解析
    @KafkaListener(id = "testGroup1", topics = "testTopic1")
    public void listen1(ConsumerRecord<String, Object> record) {
        logger.info("listen1 received: " + ((User) record.value()).toString());
    }
```

---


#### 4.位移

在Kafka中有个概念叫offset，可以理解为消息所在的偏移量，因为Kafka的消息是会持久化的，因此也支持重复消费，所以需要有类似于索引一样的偏移量概念，记录每条消息的位置，以及消费到了哪个地方。消费完后消费者会提交更新当前消费的偏移量，默认配置是自动提交，而且不是每条消息消费完就立刻提交，而是搁一段时间提交一次，下面两个参数可以配置这两项。

```
spring:
  kafka:
    consumer:
      enable-auto-commit: true  // 是否开启自动提交
      auto-commit-interval: 10000 // 自动提交的间隔时间
```

既然Kafka支持关闭自动提交，那肯定也是支持手动提交的，有同步方式和异步方式，更加灵活。还可以指定消费的位置，下面的配置是指在找不到位移记录的时候，从哪里开始读取消息，earliest是从头全部读，latest是从尾部读取一条，none是找不到位移记录就抛出异常。当然也可以在代码内部实现更细粒度化的配置，指定从哪开始消费。

```
spring:
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      enable-auto-commit: true
      auto-commit-interval: 10000
      auto-offset-reset: earliest // exception，latest，none
```

---

#### 5.再均衡

再均衡是指Kafka主题分区和消费组中的消费者的对应关系进行重新分配，这期间消费者是无法读取消息的，另外重新分配后的消费者也会丢失原来的位移记录，会根据配置重新读取消息，尤其是下面两个参数要注意，第一个是拉取时间，默认是5分钟，第二个是拉取的消息数，默认500，所以如果消息处理的逻辑比较耗时，然后消息里面又放的集合数据，循环处理，很容易超时造成再均衡，就会全部从头消费，造成重复消费。

```
spring:
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      properties:
        {max.poll.interval.ms: 300000, max.poll.records: 100}
```


---


#### 6.多线程

Kafka的消费者是可以配合多线程提高处理速度的，比如接受到消息之后把逻辑处理都放到线程池中处理，spring-kafka也提供了这种机制，只要配置@KafkaListener注解的concurrency字段即可，或者配置文件，注解配置会覆盖配置文件，不过还有个要注意的就是，主题的分区必须和线程数相同才有效，因为上面说过了，分区会对应消费者，这里一个线程就相当于一个消费者，分区少了，会有消费者收不到消息，加入你只有一个分区，即使增加线程数也是串行，因为这个分区会固定使用一个消费者。

```
@KafkaListener(id = "testGroup7", topics = "testTopic7",concurrency = "5")

spring:
  kafka:
    listener:
      concurrency: 
```