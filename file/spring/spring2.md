## Spring基础

1. 基本概念

   - 控制反转(IoC,Inversion of Control)
  
     是一种设计思想，使用者不再自己创建对象并获取对象的引用，而是让一个容器来控制对象的创建，使用者只需要从这个容器中获取对象即可。Spring中就有Spring容器来管理所有的对象，通过扫描注解和XML配置来生成对象放入容器。
   
   - 依赖注入(DI,Dependency Injection)

     可以理解为IoC的具体实现方式，讲容器中的对象自动注入到使用它的对象中，比如Spring中注解自动注入一个对象的实例到引用中，可以直接使用。

2. 用到的设计模式

   - 工厂模式

     使用工厂模式来创建对象

   - 单例模式
     
     Spring中bean的默认作用域就是singleton(单例)，这就应用到了单例模式，在Spring中依靠ConcurrentHashMap来实现单例模式，下面在ConcurrentHashMap调用前使用synchronized关键字同步是使用双重校验锁来确保不会有并发问题。

     ```
     // 通过 ConcurrentHashMap（线程安全） 实现单例注册表
     private final Map<String， Object> singletonObjects = new ConcurrentHashMap<String， Object>(64);

     public Object getSingleton(String beanName， ObjectFactory<?> singletonFactory) {
        Assert.notNull(beanName， "'beanName' must not be null");
        synchronized (this.singletonObjects) {
            // 检查缓存中是否存在实例  
            Object singletonObject = this.singletonObjects.get(beanName);
            if (singletonObject == null) {
                //...省略了很多代码
                try {
                    singletonObject = singletonFactory.getObject();
                }
                //...省略了很多代码
                // 如果实例对象在不存在，我们注册到单例注册表中。
                addSingleton(beanName， singletonObject);
            }
            return (singletonObject != NULL_OBJECT ? singletonObject : null);
        }
     }
    
     //将对象添加到单例注册表
     protected void addSingleton(String beanName， Object singletonObject) {
            synchronized (this.singletonObjects) {
                this.singletonObjects.put(beanName， (singletonObject != null ? singletonObject : NULL_OBJECT));
            }
        }
     }
     ```
   - 代理模式

     Spring中AOP就是使用的代理模式，还是两种动态代理的方式，如果代理对象实现了接口就使用JDK Proxy，否则会使用Cglib进行动态代理。

   - 模板方法模式

     Spring中JdbcTemplate、RedisTemplate等以Template结尾的类就使用到了模板模式。一般情况下，我们都是使用继承的方式来实现模板模式，但是Spring是使用Callback模式与模板方法模式配合，既达到了代码复用的效果，同时增加了灵活性。

   - 观察者模式

     Spring中就运用观察者模式来大量实现了事件监听和事件发布。

   - 适配器模式

     Spring MVC中就是运用到了适配器模式，DispatcherServlet根据请求信息调用HandlerMapping解析请求对应的Handler。解析到对应的Handler后由HandlerAdapter适配器处理。HandlerAdapter作为期望接口，具体的适配器实现类用于对目标类进行适配，Controller作为需要适配的类。
   
   - 装饰者模式

     需要连接多个数据库，而且不同的客户在每次访问中根据需要会去访问不同的数据库。这种模式根据客户的需求能够动态切换不同的数据源。

3. MVC模式

   SpringMVC框架是以请求为驱动，围绕Servlet设计，将请求发给控制器，然后通过模型对象，分派器来展示请求结果视图。其中核心类是DispatcherServlet，它是一个Servlet，顶层是实现的Servlet接口。

   - 客户端（浏览器）发送请求，直接请求到DispatcherServlet。
   - DispatcherServlet根据请求信息调用HandlerMapping，解析请求对应的Handler(Controller)。
   - 解析到对应的Handler（也就是我们平常说的Controller控制器）后，开始由HandlerAdapter适配器处理。
   - HandlerAdapter会根据Handler来调用真正的处理器开处理请求，并处理相应的业务逻辑。
   - 处理器处理完业务后，会返回一个ModelAndView对象，Model是返回的数据对象，View是个逻辑上的View。
   - ViewResolver会根据逻辑View查找实际的View。
   - DispaterServlet把返回的Model传给View（视图渲染）。
   - 把View返回给请求者（浏览器）
     
   SpringMVC重要组件说明
   
   - 前端控制器，DispatcherServlet

     Spring MVC的入口函数。接收请求，响应结果，相当于转发器，中央处理器。有了DispatcherServlet减少了其它组件之间的耦合度。用户请求到达前端控制器，它就相当于mvc模式中的c，DispatcherServlet是整个流程控制的中心，由它调用其它组件处理用户的请求，DispatcherServlet的存在降低了组件之间的耦合性。

   - 处理器映射器，HandlerMapping

     根据请求的url查找Handler。HandlerMapping负责根据用户请求找到Handler即处理器，最常见的就是Controller，里面各种方法处理请求。

   - 处理器适配器，HandlerAdapter

     按照特定规则去执行Handler，通过HandlerAdapter对处理器进行执行，这是适配器模式的应用，通过扩展适配器可以对更多类型的处理器进行执行。

   - 处理器，Handler

      最常见的就是Controller类，它处理请求，调用Service层再去返回数据。

   - 视图解析器，View resolver

      进行视图解析，比如早期JSP就是View，设置对应的解析器就可以把数据映射到JSP中。

   - 视图View

      JSP就是一类View，说白了就是页面，现在前后端分离已经用不到了。

4. Spring AOP和AspectJ AOP

   Spring AOP属于运行时增强，而AspectJ是编译时增强。 Spring AOP基于代理(Proxying)，而AspectJ基于字节码操作(Bytecode Manipulation)。AspectJ相比于Spring AOP 功能更加强大，但是Spring AOP相对来说更简单，如果切面比较少，那么两者性能差异不大。但是，当切面太多的话，最好选择AspectJ ，它比Spring AOP快很多。

5. Spring事务传播行为

   - 支持当前事务

     - TransactionDefinition.PROPAGATION_REQUIRED
     
       如果当前存在事务，则加入该事务；如果当前没有事务，则创建一个新的事务。
     - TransactionDefinition.PROPAGATION_SUPPORTS
     
       如果当前存在事务，则加入该事务；如果当前没有事务，则以非事务的方式继续运行。
     - TransactionDefinition.PROPAGATION_MANDATORY
     
       如果当前存在事务，则加入该事务；如果当前没有事务，则抛出异常。（mandatory：强制性）

   - 不支持当前事务

     - TransactionDefinition.PROPAGATION_REQUIRES_NEW
     
       创建一个新的事务，如果当前存在事务，则把当前事务挂起。
     - TransactionDefinition.PROPAGATION_NOT_SUPPORTED
     
       以非事务方式运行，如果当前存在事务，则把当前事务挂起。
     - TransactionDefinition.PROPAGATION_NEVER
     
       以非事务方式运行，如果当前存在事务，则抛出异常。

   - 其他情况

     - TransactionDefinition.PROPAGATION_NESTED
     
       如果当前存在事务，则创建一个事务作为当前事务的嵌套事务来运行；如果当前没有事务，则该取值等价于TransactionDefinition.PROPAGATION_REQUIRED。
    
   - @Transactional(rollbackFor = Exception.class)

     默认@Transactional()事务是只在遇到RuntimeException才会滚，需要手动设置rollbackFor=Exception.class
