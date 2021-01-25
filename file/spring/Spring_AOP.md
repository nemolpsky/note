### Spring AOP

Spring AOP其实就是Spring实现的一个面向切面编程的特性，其实最正统的AOP技术是AspectJ，它的功能最强大，不过强大也是有代价的，它使用起来可比Spring AOP要复杂的多了，正常的情况下Spring AOP其实已经满足开发需求了。

AspectJ支持编译时和类加载时织入，而Spring AOP则是支持运行时织入，AspectJ支持任意的切点，Spring AOP只支持方法作为切点。


---


#### 1. AOP概念
本质上面向切面变成不复杂，就是利用代理类动态的增强代码，但是AOP几个概念名词特别生涩，很容易让人莫名其妙。但是搞懂这些概念是非常重要的，死记硬背根本没有用。

- Aspect，切面

    切面其实可以这么理解，就是利用面向切面编程实现的一个模块，比如Spring中的事务就是利用AOP实现的，你可以说事务功能就是一个切面。

- Joinpoint，连接点

    这个其实就是指在哪里会应用到AOP特性，比如执行一个A方法时，AOP就会强化这个A方法，那A方法就是连接点，Spring AOP只支持方法作为连接点。

- Advice，增强

    这个比较好理解，就是通过AOP增加的代码，比如事务切面的Advice就是增加的那些控制事务的方法。

- Pointcut，切点

    这个可以理解为定义好的需要执行的方法规则，比如A，B，C三个方法，A和B需要被增强，那就需要定义一个规则来匹配A和B，排除C，说白了就是在Spring中用正则去匹配你需要增强的方法。

- Introduction，引介

    意思就是往执行的代码中添加新的属性。

- Target，目标类

    因为AOP的本质是生成代理类来实现，因为原来的类代码已经写好了，你想要增加新功能肯定要添加代码，添加过代码的类是代理类，代理类只有增加的代码，本质上还是会调用目标类去执行原有的逻辑，目标类就是你需要增加的类。

- Weaving，织入

    这个概念最生涩，但其实也最空泛，它就是说程序在使用AOP特性生成代理类来增强功能的整个过程，仅此而已。


---

#### 2. 增强类型

增强类型就是说需要在方法的什么位置来执行新增的逻辑，就这么5种类型。

- 前置通知，方法执行之前，执行通知。
- 后置通知，方法执行之后，不考虑其结果，执行通知。
- 返回后通知，方法执行成功完成之后才能执行通知。
- 抛出异常后通知，只有在方法抛出异常时才执行通知。
- 环绕通知，在方法调用之前和之后执行通知。


---


#### 3. AOP配合自定义注解

首先定义增强类的接口和实现子类，这样是为了便于以后扩展。

```
public interface AdviceHandler<T> {

     Logger logger = LoggerFactory.getLogger(AdviceHandler.class);

    /**
     * 目标方法完成时，执行的动作
     *
     * @param point     目标方法的连接点
     * @param startTime 执行的开始时间
     * @param result    执行获得的结果
     */
    default void onComplete(ProceedingJoinPoint point, long startTime,Object result) {
    }
}

@Component
public class LogAdviceHandler implements AdviceHandler<Object> {

    @Override
    public void onComplete(ProceedingJoinPoint point, long startTime, Object result) {
        // 获得被代理的类
        Object target = point.getTarget();
        String className = target.getClass().getSimpleName();
        Signature signature = point.getSignature();
        String methodName = signature.getName();
        String methodDesc = className + "." + methodName;
        Object[] args = point.getArgs();
        long costTime = System.currentTimeMillis() - startTime;

        logger.warn("{} 执行结束，耗时={}ms，入参={}, 出参={}",
                methodDesc, costTime,
                JSONUtil.toJsonStr(args),
                JSONUtil.toJsonStr(result));
    }
}
```

再定义一个切面基类和实现子类，之所以这样子设计是因为抽象基类可以写共性逻辑，而那些不确定的，需要扩展的逻辑则可以交由子类实现，在子类中定义切点和增强类，也就是说以后新增逻辑都只需要新增AdviceHandler的子类和BaseAspect的子类就可以，不修改以前的代码。

```
public abstract class BaseAspect implements ApplicationContextAware {

    Logger logger = LoggerFactory.getLogger(BaseAspect.class);

    /**
     * 切点，通过 @Pointcut 指定相关的注解
     */
    protected abstract void pointcut();

    private ApplicationContext appContext;

    /**
     * 获得切面绑定的方法增强处理器的类型
     */
    protected abstract Class<? extends AdviceHandler<?>> getAdviceHandlerType();

    @Around("pointcut()")
    public Object advice(ProceedingJoinPoint point) {
        // 获得切面绑定的方法增强处理器的类型
        Class<? extends AdviceHandler<?>> handlerType = getAdviceHandlerType();
        // 从 Spring 上下文中获得方法增强处理器的实现 Bean
        AdviceHandler<?> adviceHandler = appContext.getBean(handlerType);
        // 使用方法增强处理器对目标方法进行增强处理
        return advice(point, adviceHandler);
    }

    /**
     * 使用方法增强处理器增强被注解的方法
     *
     * @param point   连接点
     * @param handler 切面处理器
     * @return 方法执行返回值
     */
    private Object advice(ProceedingJoinPoint point, AdviceHandler<?> handler) {
        // 方法返回值
        Object result = null;
        // 开始执行的时间
        long startTime = System.currentTimeMillis();
        try {
            // 执行目标方法
            result = point.proceed();
        } catch (Throwable e) {
            // 抛出异常
            logger.error("error:", e);
        }
        // 结束
        handler.onComplete(point, startTime, result);
        return result;
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        appContext = applicationContext;
    }
}


@Aspect
@Order(1)
@Component
public class LogAspect extends BaseAspect {

    /**
     * 指定切点（处理打上 com.lp.springdemo.aops.LogAnnotation 的方法）
     */
    @Override
    @Pointcut("@annotation(com.lp.springdemo.aops.LogAnnotation)")
    protected void pointcut() {

    }

    /**
     * 指定该切面绑定的方法切面处理器为 InvokeRecordHandler
     */
    @Override
    protected Class<? extends AdviceHandler<?>> getAdviceHandlerType() {
        return LogAdviceHandler.class;
    }

}
```

最后就是定义一个注解，加在要使用的方法上。调用这个方法的时候就会进行织入操作了，这种方式适合那种有一定量但是又只针对特定代码的处理逻辑。

```
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface LogAnnotation {

    /**
     * 调用说明
     */
    String value() default "";
}


@LogAnnotation
@RequestMapping("/testModel")
public JsonModel testModel(@RequestBody JsonModel model){
    return model;
}
```