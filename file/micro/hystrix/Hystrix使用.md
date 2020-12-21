### Hystrix使用

在Java中一般分布式系统会使用Spring Cloud或者dubbo等分布式框架来搭建，Spring也提供了关于Hystrix的依赖，其实就是基于原生的Hystrix的封装，能够更方便的继承到Spring中。


---


#### 1. 添加依赖

在需要使用Hystrix的项目上添加依赖，启动类上添加注解，在配置注解即可，注意因为是第三方库，所以IDE有时候会提示无法识别配置项，Spring的文档上面页面也没有参数，但是可以去Hystrix在Github上的项目下找到所有的配置解释。

```
@EnableHystrix 
```

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
```

```
hystrix:
  command:
    # default表示，全局配置，单个配置使用key名
    default:
      execution:
        timeout:
          enabled: true
        isolation:
          thread:
            timeoutInMilliseconds: 10000
```

---

#### 2. 使用@HystrixCommand注解
除了配置文件外，还可以使用@HystrixCommand注解来配置单独的方法，优先级比配置文件高，可以灵活的修改单个接口的Hystrix配置，注意groupKey和aommandKey的设置，前者可以理解为一个分组，后者理解为一个Hystrix标识，在指定配置的时候会用到。

其中熔断器的配置默认是打开的，熔断器的触发会有两个条件，```circuitBreaker.requestVolumeThreshold```表示在十秒之内连续失败的次数，如果设置50次，连续49次都不会触发。上面的条件满足后还有```circuitBreaker.errorThresholdPercentage```参数，表示失败的请求已经占到多少比例了，如果超出比例则回触发熔断器，也就是说这个时候请求进来都直接返回或者降级。

还有一个要注意的是，Hystrix的隔离是基于线程池的，而这个线程池是基于key名，如果key名相同，就会使用同一个线程池，否则就会使用全局默认的线程池，所以一定要合理配置线程池的数量，如果线程池无法执行任务时，也会走降级方法，所以线程数不能太少，以免很容易就触发降级，太多则会导致对CPU的激烈抢占。

```
    @HystrixCommand(fallbackMethod = "errorReturn",groupKey = "key2",commandKey = "key2"
            , commandProperties = {
            @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "10000")
            },threadPoolProperties = {
            @HystrixProperty(name = "coreSize", value = "1")
    }
    )
    @GetMapping("/hystrixGet2")
    public User hystrixGet2(Integer time) throws Exception {
        time = time==null?0:time;
        logger.info("hystrixGet2 step 1，thread is {}",Thread.currentThread().getName());
        if (time>0){
            logger.info("hystrixGet2 step 2");
            TimeUnit.SECONDS.sleep(time);
        }
        logger.info("hystrixGet2 step 3");
        return new User(111);
    }
```

> https://github.com/Netflix/Hystrix/wiki/Configuration
> https://github.com/nemolpsky/feign-demo