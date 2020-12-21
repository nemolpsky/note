### SpringBoot集成Kafka

SpringBoot提供了对Kafka的集成使用，基本上不用什么配置就运行起来。

1. 添加依赖

    必须添加spring-kafka依赖包。

    ```
    <dependency>
        <groupId>org.springframework.kafka</groupId>
        <artifactId>spring-kafka</artifactId>
    </dependency>
    ```

2. 配置文件

    bootstrap-servers就是Kafka对外提供服务暴露的那个地址，在Kafka的配置文件里面有配置，producer-value-serializer是发送消息的时候把对象转为JSON字符串，否则会抛出转换失败的异常，这里把服务端口改成8888是因为Kafka启动也会占用到8080端口，如果是单机运行，会造成端口占用。

    ```
    spring:
        kafka:
            bootstrap-servers: localhost:9092
            producer:
                value-serializer: org.springframework.kafka.support.serializer.JsonSerializer

    server:
        port: 8888
    ```

3. 发送消息

    重点在于KafkaTemplate，属于spring-kafka包中的类，提供了封装好的对Kafka操作的方法，比如下面的send方法就会执行向kafka发送消息的操作，如果topic不存在，会自动创建一个topic，如果想要设置topic的副本和分区数最好是自己使用Kafka脚本创建。

    ```
    @Component
    public class KafkaSender {

        private final KafkaTemplate kafkaTemplate;

        @Autowired
        public KafkaSender(KafkaTemplate kafkaTemplate) {
            this.kafkaTemplate = kafkaTemplate;
        }

        public void send(String topic,User user){
            kafkaTemplate.send(topic,user);
        }
    }

    @RestController
    public class SendController {

    @Autowired
    private KafkaSender kafkaSender;

        @GetMapping("/send")
        public void sendUserMessage(){
            User user = new User("David","12345",new Date(),22);
            kafkaSender.sender("testTopic",user);
        }
    }
    ```


4. 接收消息

    接收消息也很方便，使用@KafkaListener注解就可以，声明当前消费者的组名和要监听的topic就可以，ConsumerRecord也是spring-kafka包提供的一个对象，里面包含了消息的很多内容，消息体是放在value中

    ```
    @Component
    public class KafkaReceiver {
        private final Logger logger = LoggerFactory.getLogger(KafkaReceiver.class);

        @KafkaListener(id = "testGroup", topics = "testTopic")
        public void listen(ConsumerRecord<String, String> record) {
            logger.info("Received: " + record.value());
        }
    }
    ```

### 总结

因为spring-kafka包是基于kafka-clients.jar开发的，所以不同的版本之间会有兼容问题，官方文档上有说明。

> https://spring.io/projects/spring-kafka#overview