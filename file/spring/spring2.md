## Spring基础

1. 基本概念

   - 控制反转(IoC)

     原来在A对象中的一个方法里有使用到B对象，使用的是new一个B对象，也就是它自己创建需要使用的对象，有了IoC之后就完全不一样了，首先IoC会负责所有的对象创建，它会先创建好A和B对象，而A对象如果需要使用B对象就不再是自己创建了，而是IoC注入进来，所以创建B对象的权利就由A对象自己交给了IoC容器，所以被称为控制反转。

     过程是从XML或注解中解析，然后由Spring的对象工厂去创建。
   
   - 依赖注入(DI)

     依赖注入就是实际的对IoC的一种实现，将IoC容器中的对象注入到需要使用的对象中。

2. 用到的设计模式

   - 工厂模式

     使用工厂模式来创建对象

   - 单例模式
     
     Spring中bean的默认作用域就是singleton(单例)，这就应用到了单例模式，在Spring中依靠ConcurrentHashMap来实现单例模式，下面在ConcurrentHashMap调用前使用synchronized关键字同步是使用双重校验锁来确保不会有并发问题。

     ```
     // 通过 ConcurrentHashMap（线程安全） 实现单例注册表
     private final Map<String, Object> singletonObjects = new ConcurrentHashMap<String, Object>(64);

     public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
        Assert.notNull(beanName, "'beanName' must not be null");
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
                addSingleton(beanName, singletonObject);
            }
            return (singletonObject != NULL_OBJECT ? singletonObject : null);
        }
     }
    
     //将对象添加到单例注册表
     protected void addSingleton(String beanName, Object singletonObject) {
            synchronized (this.singletonObjects) {
                this.singletonObjects.put(beanName, (singletonObject != null ? singletonObject : NULL_OBJECT));
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
   
   - 前端控制器DispatcherServlet（不需要工程师开发）,由框架提供（重要）

     Spring MVC 的入口函数。接收请求，响应结果，相当于转发器，中央处理器。有了 DispatcherServlet 减少了其它组件之间的耦合度。用户请求到达前端控制器，它就相当于mvc模式中的c，DispatcherServlet是整个流程控制的中心，由它调用其它组件处理用户的请求，DispatcherServlet的存在降低了组件之间的耦合性。

   - 处理器映射器HandlerMapping(不需要工程师开发),由框架提供

     根据请求的url查找Handler。HandlerMapping负责根据用户请求找到Handler即处理器（Controller），SpringMVC提供了不同的映射器实现不同的映射方式，例如：配置文件方式，实现接口方式，注解方式等。

   - 处理器适配器HandlerAdapter

     按照特定规则（HandlerAdapter要求的规则）去执行Handler，通过HandlerAdapter对处理器进行执行，这是适配器模式的应用，通过扩展适配器可以对更多类型的处理器进行执行。

   - 处理器Handler(Controller，需要工程师开发)

     编写Handler时按照HandlerAdapter的要求去做，这样适配器才可以去正确执行Handler Handler 是继DispatcherServlet前端控制器的后端控制器，在DispatcherServlet的控制下Handler对具体的用户请求进行处理。 由于Handler涉及到具体的用户业务请求，所以一般情况需要工程师根据业务需求开发Handler。

   - 视图解析器View resolver(不需要工程师开发),由框架提供

     进行视图解析，根据逻辑视图名解析成真正的视图（view） View Resolver负责将处理结果生成View视图，View Resolver首先根据逻辑视图名解析成物理视图名即具体的页面地址，再生成View视图对象，最后对View进行渲染将处理结果通过页面展示给用户。 springmvc框架提供了很多的View视图类型，包括：jstlView、freemarkerView、pdfView等。 一般情况下需要通过页面标签或页面模版技术将模型数据通过页面展示给用户，需要由工程师根据业务需求开发具体的页面。

   - 视图View(需要工程师开发)

     View是一个接口，实现类支持不同的View类型（jsp、freemarker、pdf...）

     注意：处理器Handler（也就是我们平常说的Controller控制器）以及视图层view都是需要我们自己手动开发的。其他的一些组件比如：前端控制器DispatcherServlet、处理器映射器HandlerMapping、处理器适配器HandlerAdapter等等都是框架提供给我们的，不需要自己手动开发。

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