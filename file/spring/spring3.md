## Spring注解

1. @Component

   标注当前的类是需要Spring自动装配到容器中的

2. @Configuration

   标注当前的类有一个或多个@Bean声明的方法

3. @Bean

   表示当前方法需要Spring来生成一个Bean对象

4. @Repository/@Service/@Controller 

   和@Component是一个意思，只不过是命名上规范，表示该类是Dao/Servier/Controller中的一层

5. @Autowired

   自动装配Bean对象到当前引用上

6. @RequestBody

   设置请求参数的类型为JSON格式

7. @ResponseBody

   以JSON格式返回对象

8. @RestController

   相当于@Controller + @ResponseBody，省掉了@ResponseBody

9. @ComponentScan

   扫描被@Component、@Service、@Controller等注解的bean

10. @EnableAutoConfiguration
    
    启用SpringBoot的自动配置机制

9. @SpringBootApplication

   相当于@Configuration + @EnableAutoConfiguration + @ComponentScan 