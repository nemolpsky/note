## SpringBoot的starter机制和自动配置原理

SpringBoot相比于Spring最大的好处就是一些固定配置模板无需配置，只需要根据所需要的功能来添加相关的starter依赖即可，而且还内置了tomcat等容器，真正的坐到了开箱即用，大大的提高了效率，只需要单独配置自己需要修改的个性化配置即可，这篇文章就来了解下SpringBoot的starter机制和自动配置的原理。


1. starter依赖

   SpringBoot官方提供了很多依赖，用于集成各种各样的框架，而SpringBoot的依赖都是以starter结尾，官方的是spring-boot-starter-xxx格式，而第三方则是xxx-spring-boot-starter格式，下面就以Mybtais的依赖作为说明例子，在pom文件中添加Mybtais的依赖。

   ```
    <dependency>
        <groupId>org.mybatis.spring.boot</groupId>
        <artifactId>mybatis-spring-boot-starter</artifactId>
        <version>1.3.2</version>
    </dependency>
   ```

   可以查看下载到本地的依赖的jar包，会发现里面的META-INF文件夹下，只有一个pom文件和一个配置文件，所以其实就是直接把所有需要的依赖都整合成一个依赖，直接引入即可。

   ```
   <?xml version="1.0" encoding="UTF-8"?>

   <project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
   <modelVersion>4.0.0</modelVersion>
    <parent>
     <groupId>org.mybatis.spring.boot</groupId>
     <artifactId>mybatis-spring-boot</artifactId>
     <version>1.3.2</version>
    </parent>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <name>mybatis-spring-boot-starter</name>
    <dependencies>
     <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter</artifactId>
     </dependency>
     <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-jdbc</artifactId>
     </dependency>
     <dependency>
      <groupId>org.mybatis.spring.boot</groupId>
      <artifactId>mybatis-spring-boot-autoconfigure</artifactId>
     </dependency>
     <dependency>
      <groupId>org.mybatis</groupId>
      <artifactId>mybatis</artifactId>
     </dependency>
     <dependency>
      <groupId>org.mybatis</groupId>
      <artifactId>mybatis-spring</artifactId>
     </dependency>
    </dependencies>
   </project>
   ```

2. 自动装配

   如果有使用过Spring来配置Mybatis的话都知道会需要配置数据源诸如此类的一大堆东西，那为什么使用starter依赖后就不会呢，其实主要关键的地方就在```mybatis-spring-boot-autoconfigure```这个依赖中，打开```MybatisAutoConfiguration这个类```，可以看到有```@Configuration```和```@Bean```注解来创建```sqlSessionFactory```等对象，所以不需要手动创建了。


   ```
   @Configuration
   @ConditionalOnClass({SqlSessionFactory.class, SqlSessionFactoryBean.class})
   @ConditionalOnBean({DataSource.class})
   @EnableConfigurationProperties({MybatisProperties.class})
   @AutoConfigureAfter({DataSourceAutoConfiguration.class})
   public class MybatisAutoConfiguration {
       private static final Logger logger = LoggerFactory.getLogger(MybatisAutoConfiguration.class);
       private final MybatisProperties properties;
       private final Interceptor[] interceptors;
       private final ResourceLoader resourceLoader;
       private final DatabaseIdProvider databaseIdProvider;
       private final List<ConfigurationCustomizer> configurationCustomizers;

    public MybatisAutoConfiguration(MybatisProperties properties, ObjectProvider<Interceptor[]> interceptorsProvider, ResourceLoader resourceLoader, ObjectProvider<DatabaseIdProvider> databaseIdProvider, ObjectProvider<List<ConfigurationCustomizer>> configurationCustomizersProvider) {
        ...
    }

    @PostConstruct
    public void checkConfigFileExists() {
        ...
    }

    @Bean
    @ConditionalOnMissingBean
    public SqlSessionFactory sqlSessionFactory(DataSource dataSource) throws Exception {
        ...
    }

    @Bean
    @ConditionalOnMissingBean
    public SqlSessionTemplate sqlSessionTemplate(SqlSessionFactory sqlSessionFactory) {
        ...
    }

    @Configuration
    @Import({MybatisAutoConfiguration.AutoConfiguredMapperScannerRegistrar.class})
    @ConditionalOnMissingBean({MapperFactoryBean.class})
    public static class MapperScannerRegistrarNotFoundConfiguration {
        ...
    }

    public static class AutoConfiguredMapperScannerRegistrar implements BeanFactoryAware, ImportBeanDefinitionRegistrar, ResourceLoaderAware {
        private BeanFactory beanFactory;
        private ResourceLoader resourceLoader;

        public AutoConfiguredMapperScannerRegistrar() {
            ...
        }

        public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
            ...
        }

        public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
            ...
        }

        public void setResourceLoader(ResourceLoader resourceLoader) {
            ...
        }

    }
   ```

   还可以看到这个类使用了很多其他的注解，来保证在加载这个类里面的Bean之前前置的Bean是否有加载。

   ```

   @Configuration
   // 指定class位于类路径上，才会实例化这个Bean。
   @ConditionalOnClass({SqlSessionFactory.class, SqlSessionFactoryBean.class})
   // 在当前上下文中存在指定bean时，才会实例化这个Bean。
   @ConditionalOnBean({DataSource.class})
   @EnableConfigurationProperties({MybatisProperties.class})
   // 在指定bean完成自动配置后实例化这个bean。
   @AutoConfigureAfter({DataSourceAutoConfiguration.class})
   public class MybatisAutoConfiguration {
      ....
   }
   ```

   再看```DataSourceAutoConfiguration```这个类，是```MybatisAutoConfiguration```加载前要先创建的一个类，它使用了一个```@EnableConfigurationProperties```注解，这个注解是配合```@ConfigurationProperties```注解使用，将yml或者properties文件转化为bean，```@ConfigurationProperties```是定义了配置文件中的各种key，所以如果添加了个性化的配置，也会生效。

   ```
   @Configuration
   @ConditionalOnClass({DataSource.class, EmbeddedDatabaseType.class})
   @EnableConfigurationProperties({DataSourceProperties.class})
   @Import({DataSourcePoolMetadataProvidersConfiguration.class, DataSourceInitializationConfiguration.class})
   public class DataSourceAutoConfiguration {
       ...
   }

   @ConfigurationProperties(prefix = "spring.datasource")
   public class DataSourceProperties implements BeanClassLoaderAware, InitializingBean {
    private ClassLoader classLoader;
    private String name;
    private boolean generateUniqueName;
    private Class<? extends DataSource> type;
    private String driverClassName;
    private String url;
    private String username;
    private String password;
    private String jndiName;
    private DataSourceInitializationMode initializationMode;
    private String platform;
    private List<String> schema;
    private String schemaUsername;
    private String schemaPassword;
    private List<String> data;
    private String dataUsername;
    private String dataPassword;
    private boolean continueOnError;
    private String separator;
    private Charset sqlScriptEncoding;
    private EmbeddedDatabaseConnection embeddedDatabaseConnection;
    private DataSourceProperties.Xa xa;
    private String uniqueName;
    ...
   }
   ```