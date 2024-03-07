### 一、ApplicationContext容器
ApplicationContext是BeanFactory的子类接口，都是用于加载对象的容器，只不过BeanFactory更偏向于自定义配置，而ApplicationContext则是直接继承了更多完善的功能，所以如果没有特别的需求下，最好使用ApplicationContext来加载对象。

ApplicationContext有好几个实现类，比如ClassPathXmlApplicationContext和FileSystemXmlApplicationContext，既可以在代码中显示的调用，也可以直接在配置文件中配置。大致流程就是从元数据中读取配置生成对象，再注入实例到代码中。
  
- web.xml配置

  监听器会检查contextConfigLocation配置的参数值，也就是配置文件的路径。如果没有配置则使用缺省路径```/WEB-INF/applicationContext.xml```，支持通配符格式配置。```/WEB-INF/*Context.xml```表示名称以Context.xml结尾并驻留在WEB-INF目录中的所有文件， ```/WEB-INF/**/*Context.xml```表示适用于WEB-INF的任何子目录中的所有此类文件
  ```
  <context-param>
      <param-name>contextConfigLocation</param-name>
      <param-value>/WEB-INF/daoContext.xml /WEB-INF/applicationContext.xml</param-value>
  </context-param>
  
  <listener>
      <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
  </listener>
  ```

---

### 二、配置元数据

元数据其实就是注解或者XML配置文件里的配置，将这些元数据生成对象放入容器中，这里主要介绍注解配置。

- @Bean

  方法级别的注解，通常和@Configuration或@Component配合使用，可以用来创建一个对象，默认情况下对象名就是方法名。
  ```
  @Configuration
  public class TestClass1 {

      @Bean // 默认情况下对象名是test1
      public Test test1(){
          Test test1 = new Test();
          return test1;
      }

  //  相同效果的XML配置
  //  <beans>
  //    <bean id="test1" class="com.acme.services.MyServiceImpl"/>
  //  </beans>
  }
  ```

  可以修改对象名
  ```
  //默认对象名就是方法名
  @Bean
  //可以显示指定方法名
  @Bean(name = "test5")
  ```

  返回类型可以是接口
  ```
  // 假设Test类实现了TestInterface接口，这样也可以成功生成对象
  @Bean
  public TestInterface test1(){
      Test test1 = new Test();
      return test1;
  }
  ```
  
  生成的对象可以起多个别名
  ```
  @Bean({"test2", "test3", "test4"})
  ```

- @Scope

  用来修改Bean的作用域，默认是单例模式singleton，可以使用@Scope("prototype")来修改。


- @Configuration

  类级别的注解，当一个类使用这个注解时，Spring才会把这个类当作元数据去扫描，否则里面的@Bean注解都不会被扫描到，也就不会生效，再配合@ComponentScan设置包的扫描路径，可以确保生成的Bean内在依赖也已经注入好了。
  ```
  @Configuration
  @ComponentScan(basePackages = "com.example")
  public class TestClass1 {

      @Bean 
      public Test test1(){
          Test test1 = new Test();
          return test1;
      }

      @Bean
      public Test test2(){
          Test test2 = new Test();
          System.out.println("m2-t1-" + test1());
          System.out.println("m2-t2-" + test2);
          return test2;
      }
  }
  ```
  
- @Import

  被@Configuration注解的类还可以直接引用另一个被@Configuration注解的类，只要使用@Import，这样时候只需要操作一个类就可以获取到所有类中注入的对象了。
  ```
  @Configuration
  public class ConfigA {

      @Bean
      public A a() {
          return new A();
      }
  }

  @Configuration
  @Import(ConfigA.class)
  // 支持多个
  //@Import({ConfigA.class,ConfigC.class,ConfigD.class})
  public class ConfigB {

      @Bean
      public B b() {
          return new B();
      }
  }

  public static void main(String[] args) {
      ApplicationContext ctx = new AnnotationConfigApplicationContext(ConfigB.class);

      // 两个类中的对象都可以直接通过一个类获取到
      A a = ctx.getBean(A.class);
      B b = ctx.getBean(B.class);
  }
  ```

- @ImportResource

  注解和XML混合使用的情况下，可以使用该注解引入XML中配置的类。
  ```
  @Configuration
  @ImportResource("classpath:/com/acme/properties-config.xml")
  public class AppConfig {
      @Value("${jdbc.url}")
      private String url;

      @Value("${jdbc.username}")
      private String username;

      @Value("${jdbc.password}")
      private String password;
      ...
  }

  // jdbc.properties
  // jdbc.url=jdbc:hsqldb:hsql://localhost/xdb
  // jdbc.username=sa
  // jdbc.password=
  <beans>
      <context:property-placeholder location="classpath:/com/acme/jdbc.properties"/>
  </beans>
  ```

---

