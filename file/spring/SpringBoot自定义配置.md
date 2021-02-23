### 1. 自定义配置

Spring Boot提供了自动装配，但是也提供了很多种方式可以修改配置，比如下面这些方式都可以修改配置，并且优先级从上到下降序排列，默认配置是最低优先级的。

- 命令行参数
- java:comp/env里的JNDI属性
- JVM系统属性
- 操作系统环境变量
- 随机生成的带random.*前缀的属性（在设置其他属性时，可以引用它们，比如${random.long}）
- 应用程序以外的application.properties或者appliaction.yml文件
- 打包在应用程序内的application.properties或者appliaction.yml文件
- 通过@PropertySource标注的属性源
- 默认属性

其中配置文件也可以放在不同的地方，优先级同样是从上到下降序排列。此外yml会覆盖properties文件。application.properties和application.yml文件能放在以下四个位置。
- 外置，在相对于应用程序运行目录的/config子目录里。
- 外置，在应用程序运行的目录里。
- 内置，在config包内。
- 内置，在Classpath根目录。



---

### 2. Bean注入外部配置

```@ConfigurationProperties```注解可以配合配置文件使用set属性方法注入属性值，比如下面的例子表示会从配置文件中读取```test```为前缀的属性，注入到Bean中。默认是从全局配置文件中读取，还可以配合```@PropertySource```注解从指定配置文件中读取。

```
@Component
@ConfigurationProperties(prefix ="test")
@PropertySource(value = "classpath:test.properties")
public class TestProperties {
    private String id;
    public void setId(String id) {
        this.id = id;
    }
    public String getId() {
        return id;
    }
}
```
```
test:
    id: 500
```


---


### 3. Profile配置不同环境

在配置文件中可以分别设置对应不同环境的配置文件，例如dev，test和prd，使用下面的属性来指定当前的环境。

```
spring:
    profiles:
        active: test
```

如果主配置文件是application.yml，那么可以遵循application-{profile}.yml的命名格式创建不同环境下的配置文件，如果不想用那么多配置文件，也可以放在同一个配置文件里，使用```---```来分割。

```
logging:
    level:
        root: INFO
---
spring:
    profiles: dev

logging:
    level:
        root: DEBUG
---
spring:
    profiles: prd
    
logging:
    path: /tmp/
    file: BookWorm.log
    level:
        root: WARN
```