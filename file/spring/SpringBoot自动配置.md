### 1.启动引导

所有的Spring Boot应用都有一个启动引导类，通过这个类可以启动整个应用程序，使用```@SpringBootApplication```注解，其实这个注解是由```@Configuration```，```@ComponentScan```和```@EnableAutoConfiguration```三个注解组合起来的，分别作用是指定该类是个配置类，开启组件扫描和允许自动配置，尤其是最后的允许自动配置让Spring Boot可以自动装载默认配置。

```
@SpringBootApplication
public class UserApplication {

    public static void main(String[] args) {
        SpringApplication.run(UserApplication.class, args);
    }
}
```

---

### 2.测试引导

Spring Boot进行测试时，需要使用```@SpringApplicationConfiguration```注解来指定上下文，也就是上面的引导类。

```
@RunWith(SpringJUnit4ClassRunner.class)
@SpringApplicationConfiguration(classes = UserApplication.class)
public class UserApplicationTests {

}
```


---


### 3.起步依赖

所谓的起步依赖也就是指下面这样的Spring Boot提供的依赖包，只需要按需求添加就可以使用，在Spring时代如果想要搭建一个最基本的Web应用，除了要配置很多模板代码之外，还需要添加各种各样的类库，而Spring Boot则只需要添加一个依赖包即可，因为这个依赖包内部就是就包含了所有需要的类库。

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

正如上面所说每个起步依赖都包含了很多类库，所以本质上它就是一个Maven的POM，所以可以使用```mvn dependency:tree```命令来查看，会发现一个起步依赖下会有很多类库。比如```spring-boot-starter-web```下面会有这么多类库，如果使用Spring自己配置都需要我们自己去添加这些依赖，而且最重要的是，Spring Boot会自动搭配这些类库的版本，根据Spring Boot的版本来配置，如果自己配置很有可能造成某些类库因版本导致的兼容问题。

```
[INFO] +- org.springframework.boot:spring-boot-starter-web:jar:2.3.5.RELEASE:compile
[INFO] |  +- org.springframework.boot:spring-boot-starter-json:jar:2.3.5.RELEASE:compile
[INFO] |  |  +- com.fasterxml.jackson.core:jackson-databind:jar:2.11.3:compile
[INFO] |  |  |  +- com.fasterxml.jackson.core:jackson-annotations:jar:2.11.3:compile
[INFO] |  |  |  \- com.fasterxml.jackson.core:jackson-core:jar:2.11.3:compile
[INFO] |  |  +- com.fasterxml.jackson.datatype:jackson-datatype-jdk8:jar:2.11.3:compile
[INFO] |  |  +- com.fasterxml.jackson.datatype:jackson-datatype-jsr310:jar:2.11.3:compile
[INFO] |  |  \- com.fasterxml.jackson.module:jackson-module-parameter-names:jar:2.11.3:compile
[INFO] |  +- org.springframework.boot:spring-boot-starter-tomcat:jar:2.3.5.RELEASE:compile
[INFO] |  |  +- org.apache.tomcat.embed:tomcat-embed-core:jar:9.0.39:compile
[INFO] |  |  +- org.glassfish:jakarta.el:jar:3.0.3:compile
[INFO] |  |  \- org.apache.tomcat.embed:tomcat-embed-websocket:jar:9.0.39:compile
[INFO] |  +- org.springframework:spring-web:jar:5.2.10.RELEASE:compile
[INFO] |  |  \- org.springframework:spring-beans:jar:5.2.10.RELEASE:compile
[INFO] |  \- org.springframework:spring-webmvc:jar:5.2.10.RELEASE:compile
[INFO] |     +- org.springframework:spring-context:jar:5.2.10.RELEASE:compile
[INFO] |     \- org.springframework:spring-expression:jar:5.2.10.RELEASE:compile
```


所以这其实是一个传递依赖的机制，而如果有时候不需要其中某些依赖包，可以使用```<exclusions>```来排除依赖，比如下面的写法就是排除```spring-boot-starter-web```依赖中的```com.fasterxml.jackson.core```依赖。

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>com.fasterxml.jackson.core</groupId>
        </exclusion>
    </exclusions>
</dependency>
```

或者说是想要指定的版本，只需要在pom文件里指定版本号的依赖即可，Maven会用最近的依赖进行覆盖。

```
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.4.3</version>
</dependency>
```

--- 

### 4.自动装配
上面讲了Spring Boot是如何解决众多的依赖问题，最后一个也是最重要的问题就是Spring Boot如何进行自动装配，或者说它是怎么知道我们需要是使用数据库或者Redis并且自动帮我们配置这些属性，其实这种机制是依赖于Spring4.0引入的条件化配置，下面这些都是相关的注解。

```
@ConditionalOnBean 配置了某个特定Bean
@ConditionalOnMissingBean 没有配置特定的Bean
@ConditionalOnClass Classpath里有指定的类
@ConditionalOnMissingClass Classpath里缺少指定的类
@ConditionalOnExpression 给定的Spring Expression Language（ SpEL）表达式计算结果为true
@ConditionalOnJava Java的版本匹配特定值或者一个范围值
@ConditionalOnJndi 参数中给定的JNDI位置必须存在一个，如果没有给参数，则要有JNDIInitialContext
@ConditionalOnProperty 指定的配置属性要有一个明确的值
@ConditionalOnResource Classpath里有指定的资源
@ConditionalOnWebApplication 这是一个Web应用程序
@ConditionalOnNotWebApplication 这不是一个Web应用程序
```

这些注解是需要配合```Condition```接口使用，例如下面在```matches```方法中的代码是用来确认在当前JVM的classpath下是否包含该类，如果包含则会创建```MyService```的Bean对象。

```
public class JdbcTemplateCondition implements Condition {

    @Override
    public boolean matches(ConditionContext context,AnnotatedTypeMetadata metadata) {
        try {
            context.getClassLoader().loadClass("org.springframework.jdbc.core.JdbcTemplate");
            return true;
        } catch (Exception e) {
            return false;
        }
    }
}

@Conditional(JdbcTemplateCondition.class)
public MyService myService() {
}
```

例如下面的```DataSourceAutoConfiguration```类就是基于这样来配置的，如果不满足条件，这个类就不会加载到Bean容器中。

```
@Configuration
@ConditionalOnClass({ DataSource.class, EmbeddedDatabaseType.class })
@EnableConfigurationProperties(DataSourceProperties.class)
@Import({ Registrar.class, DataSourcePoolMetadataProvidersConfiguration.class })
public class DataSourceAutoConfiguration {

}

```

实际上Spring Boot应用会有一个```spring-boot-autoconfigure```jar包，这里面包含了很多配置类，它都会放在应用程序的classpath下，再配合上面的条件注解，Spring Boot就可以精确的自动添加需要的配置了。