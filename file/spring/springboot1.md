## SpringBoot启动类run()底层原理


### 启动类

  一般```SpringBoot```项目的启动类就是一个```@SpringBootApplication```注解，加上一个```main()```方法，调用```SpringApplication```的```run()```方法，把启动类的类文件当做参数传递进去。

  ```
  @SpringBootApplication
  public class SpringMybatisApplication {

      public static void main(String[] args) {
          SpringApplication.run(SpringMybatisApplication.class, args);
      }
  }
  ```

---

### SpringApplication类初始化

  - 构造方法和初始化方法

    ```
    public SpringApplication(ResourceLoader resourceLoader, Object... sources) {
        this.bannerMode = Mode.CONSOLE;
        this.logStartupInfo = true;
        this.addCommandLineProperties = true;
        this.headless = true;
        this.registerShutdownHook = true;
        this.additionalProfiles = new HashSet();
        this.resourceLoader = resourceLoader;
        // 调用初始化方法
        this.initialize(sources);
    }
    ```

    ```
    private void initialize(Object[] sources) {
        if (sources != null && sources.length > 0) {
            this.sources.addAll(Arrays.asList(sources));
        }

        // 判断是否是web程序，是个boolean变量
        // 依据是判断类加载器中是否包含javax.servlet.Servlet和 org.springframework.web.context.ConfigurableWebApplicationContext
        this.webEnvironment = this.deduceWebEnvironment();
        // 找出所有的应用程序初始化器
        this.setInitializers(this.getSpringFactoriesInstances(ApplicationContextInitializer.class));
        // 找出所有的应用程序事件监听器
        this.setListeners(this.getSpringFactoriesInstances(ApplicationListener.class));
        // 寻找main()方法的类
        this.mainApplicationClass = this.deduceMainApplicationClass();
    }
    ```

  - 应用程序初始化器和应用程序监听器

    所有的应用程序初始化器和应用程序监听器都会继承下面这两个接口

    ```
    // 应用程序初始化器
    public interface ApplicationListener<E extends ApplicationEvent> extends EventListener {
        void onApplicationEvent(E var1);
    }

    // 应用程序监听器
    public interface ApplicationContextInitializer<C extends ConfigurableApplicationContext> {
        void initialize(C var1);
    }

    ```

    一般情况下会初始化出下列5个应用程序初始化器和9个应用程序监听器
    - 应用程序初始化器
    - org.springframework.boot.context.config.DelegatingApplicationContextInitializer
    - org.springframework.boot.context.ContextIdApplicationContextInitializer
    - org.springframework.boot.context.ConfigurationWarningsApplicationContextInitializer
    - org.springframework.boot.context.web.ServerPortInfoApplicationContextInitializer
    - org.springframework.boot.autoconfigure.logging.AutoConfigurationReportLoggingInitializer

    - 应用程序监听器
    - org.springframework.boot.context.config.ConfigFileApplicationListener
    - org.springframework.boot.context.config.AnsiOutputApplicationListener
    - org.springframework.boot.logging.LoggingApplicationListener
    - org.springframework.boot.logging.ClasspathLoggingApplicationListener
    - org.springframework.boot.autoconfigure.BackgroundPreinitializer
    - org.springframework.boot.context.config.DelegatingApplicationListener
    - org.springframework.boot.builder.ParentContextCloserApplicationListener
    - org.springframework.boot.context.FileEncodingApplicationListener
    - org.springframework.boot.liquibase.LiquibaseServiceLocatorApplicationListener

---

### SpringApplication类的run方法

  启动类是要调用```SpringApplication.run()```方法才会启动```SpringBoot```。

  ```
    public ConfigurableApplicationContext run(String... args) {
        // 任务执行观察器，可以记录运行时间等
        StopWatch stopWatch = new StopWatch();
        // 开始计时
        stopWatch.start();
        ConfigurableApplicationContext context = null;
        FailureAnalyzers analyzers = null;
        this.configureHeadlessProperty();
        // 获取SpringApplicationRunListeners，是一个包含SpringApplicationRunListener集合和日志类的类
        // 用于监听SpringApplication.run()方法
        SpringApplicationRunListeners listeners = this.getRunListeners(args);
        // 循环启动所有SpringApplicationRunListener
        listeners.starting();

        try {
            // 应用参数持有类，里面放的都是run()方法传递进来的参数
            ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
            // 记录环境变量的类
            ConfigurableEnvironment environment = this.prepareEnvironment(listeners, applicationArguments);
            Banner printedBanner = this.printBanner(environment);
            // 创建Spring容器
            context = this.createApplicationContext();
            // 异常分析的类
            new FailureAnalyzers(context);
            // 做context的准备工作
            this.prepareContext(context, environment, listeners, applicationArguments, printedBanner);
            // 根据新的绑定参数重新对context进行更新
            this.refreshContext(context);
            // 创建完容器后额外操作
            this.afterRefresh(context, applicationArguments);
            // 广播事件
            listeners.finished(context, (Throwable)null);
            // 初始化完成，停止记录
            stopWatch.stop();
            if (this.logStartupInfo) {
                (new StartupInfoLogger(this.mainApplicationClass)).logStarted(this.getApplicationLog(), stopWatch);
            }
            // 返回Spring容器
            return context;
        } catch (Throwable var9) {
            this.handleRunFailure(context, listeners, (FailureAnalyzers)analyzers, var9);
            throw new IllegalStateException(var9);
        }
    }

  ```

---

### prepareContext()和refreshContext()方法

  这两个方法是创建容器和刷新容器的方法

  ```
    private void prepareContext(ConfigurableApplicationContext context, ConfigurableEnvironment environment, SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments, Banner printedBanner) {
        //设置容器环境
        context.setEnvironment(environment);
        this.postProcessApplicationContext(context);
        //执行容器中的 ApplicationContextInitializer 包括spring.factories和通过三种方式自定义的
        this.applyInitializers(context);

        // 向各个监听器广播容器已经准备好的事件
        listeners.contextPrepared(context);
        if (this.logStartupInfo) {
            this.logStartupInfo(context.getParent() == null);
            this.logStartupProfileInfo(context);
        }

        // 将main函数中的args参数封装成单例Bean，注册进容器
        context.getBeanFactory().registerSingleton("springApplicationArguments", applicationArguments);
        if (printedBanner != null) {
            context.getBeanFactory().registerSingleton("springBootBanner", printedBanner);
        }

        Set<Object> sources = this.getSources();
        Assert.notEmpty(sources, "Sources must not be empty");
        //将启动类注入容器
        this.load(context, sources.toArray(new Object[sources.size()]));

        //广播容器已加载事件
        listeners.contextLoaded(context);
    }

    private void refreshContext(ConfigurableApplicationContext context) {
        // 刷新容器
        this.refresh(context);
        if (this.registerShutdownHook) {
            try {
                context.registerShutdownHook();
            } catch (AccessControlException var3) {
                ;
            }
        }

    }
  ```

---

### afterRefresh()方法

  这个方法是创建完容器后调用的。

  ```
    protected void afterRefresh(ConfigurableApplicationContext context, ApplicationArguments args) {
        // 调用Spring容器中的ApplicationRunner和CommandLineRunner接口的实现类
        this.callRunners(context, args);
    }

    private void callRunners(ApplicationContext context, ApplicationArguments args) {
        List<Object> runners = new ArrayList();
        // 找出Spring容器中ApplicationRunner接口的实现类
        runners.addAll(context.getBeansOfType(ApplicationRunner.class).values());
        // 找出Spring容器中CommandLineRunner接口的实现类
        runners.addAll(context.getBeansOfType(CommandLineRunner.class).values());
        // 对runners进行排序，根据设置的优先级进行排序
        AnnotationAwareOrderComparator.sort(runners);
        Iterator var4 = (new LinkedHashSet(runners)).iterator();
        // 遍历runners依次执行
        while(var4.hasNext()) {
            Object runner = var4.next();
            if (runner instanceof ApplicationRunner) {
                this.callRunner((ApplicationRunner)runner, args);
            }

            if (runner instanceof CommandLineRunner) {
                this.callRunner((CommandLineRunner)runner, args);
            }
        }

    }
  ```

---

### 总结
```SpringBoot```启动会做两件事，第一件事就是初始化```SpringApplication```，第二步则是调用```SpringApplication.run()```方法。

- 初始化

  - 在```run()```方法中接受的参数添加到```SpringApplication```的中的一个```LinkedHashSet```集合中
  - 判断是否是```web```程序，并设置到```webEnvironment```这个```boolean```属性中
  - 找出所有的初始化器，默认有5个，设置到```initializers```属性中
  - 找出所有的应用程序监听器，默认有9个，设置到```listeners```属性中
  - 找出运行```main()```方法的启动类

- run()方法

  - 构造一个```StopWatch```，记录```SpringApplication```的执行
  - 找出所有监听```run()```方法的```SpringApplicationRunListener```类来开启监听
  - 初始化并刷新```Spring```容器
  - 在```Spring```容器中找出```ApplicationRunner```和```CommandLineRunner```接口的实现类排序再依次执行。
