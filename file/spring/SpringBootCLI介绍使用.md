### 1. Spring Boot CLI介绍

Spring Boot CLI是Spring官方推出的一个快速运行Spring Boot项目的命令行工具，众所周知，Spring Boot提供了很多依赖提供默认配置，比如数据库，Redis等等，但是这些依赖也是相对来说比较繁琐的，还需要配合Maven来构建打包，而Spring Boot CLI的作用就是更加简化这些操作，不需要配置依赖，不需要构建，甚至内部代码都是使用Groovy脚本来编写，相比Java而言Groovy脚本更加简便，语法和Java类似，甚至不需要导包，没有修饰符，Spring Boot CLI使用内嵌的Groovy编译器来编译Groovy代码，因此我们可以更加快速专心的使用Spring Boot。

其实Spring Boot CLI机制还是利用Spring Boot的自动配置，它会根据代码来分析需要引入什么依赖包，例如检测到脚本其实是一个Web应用，就会自动添加tomcat容器依赖使脚本能够正常运行。


---


### 2. 安装配置

Spring Boot CLI提供了不同操作系统下的版本，可以到```https://docs.spring.io/spring-boot/docs/current/reference/html/getting-started.html#getting-started-installing-the-cli```上下载，Windows系统需要下载```spring-boot-cli-2.4.0-bin.zip```压缩包，需要注意的是Spring Boot CLI依赖与Java和Maven，所以要提前安装它们，然后把压缩包中的bin目录添加到环境变量Path中，例如```E:\spring-2.4.0\bin```，可以使用spring version命令查看版本号。

```
C:\Users\Half>spring version
Spring CLI v2.4.0
```

前面说过了Spring Boot CLI本质是自动解析下载依赖包，所以要配置Spring Boot CLI的依赖下载仓库，它默认会读取C盘下```~/.m2/settings.xml```文件，这其实就是Maven的配置文件，所以可以直接把Maven的配置文件复制过来，这样它就会使用本地的Maven配置的仓库来下载依赖包了。

---


### 3. 运行脚本

因为Spring Boot CLI使用Groovy脚本，所以可以创建一个Groovy脚本，添加一个最简单的Controller，可以直接使用IDEA来编辑Groovy，可以看到没有任何导包的代码。

```
@RestController
class Test {

    @RequestMapping("/")
    String home() {
        "This is Spring Boot Cli Test!"
    }

}
```

可以看到code目录下放着上面的Groovy脚本，执行```spring run test.groovy```命令就可以运行该脚本，第一次执行的时候会提示解析下载依赖包，如果没有按上面配置使用国内仓库镜像会很慢，可以看到程序运行了，打印了日志，默认端口8080，使用```http://localhost:8080/```地址请求可以成功获取到值。

```
E:\spring-2.4.0\code>dir
 驱动器 E 中的卷是 开发辅助
 卷的序列号是 6E0F-553A

E:\spring-2.4.0\code 的目录

2020/11/27  23:16    <DIR>          .
2020/11/27  23:16    <DIR>          ..
2020/11/27  23:14               131 test.groovy
               1 个文件            131 字节
               2 个目录 198,673,166,336 可用字节

E:\spring-2.4.0\code>spring run test.groovy

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v2.4.0)

2020-11-27 23:16:45.077  INFO 21164 --- [       runner-0] o.s.boot.SpringApplication               : Starting application using Java 1.8.0_221 on Half-PC with PID 21164 (started by Half in E:\spring-2.4.0\code)
2020-11-27 23:16:45.084  INFO 21164 --- [       runner-0] o.s.boot.SpringApplication               : No active profile set, falling back to default profiles: default
2020-11-27 23:16:46.180  INFO 21164 --- [       runner-0] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8080 (http)
2020-11-27 23:16:46.195  INFO 21164 --- [       runner-0] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2020-11-27 23:16:46.195  INFO 21164 --- [       runner-0] org.apache.catalina.core.StandardEngine  : Starting Servlet engine: [Apache Tomcat/9.0.39]
2020-11-27 23:16:46.243  INFO 21164 --- [       runner-0] org.apache.catalina.loader.WebappLoader  : Unknown class loader [org.springframework.boot.cli.compiler.ExtendedGroovyClassLoader$DefaultScopeParentClassLoader@776389f7] of class [class org.springframework.boot.cli.compiler.ExtendedGroovyClassLoader$DefaultScopeParentClassLoader]
2020-11-27 23:16:46.278  INFO 21164 --- [       runner-0] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2020-11-27 23:16:46.278  INFO 21164 --- [       runner-0] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 1015 ms
2020-11-27 23:16:46.499  INFO 21164 --- [       runner-0] o.s.s.concurrent.ThreadPoolTaskExecutor  : Initializing ExecutorService 'applicationTaskExecutor'
2020-11-27 23:16:47.018  INFO 21164 --- [       runner-0] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
2020-11-27 23:16:47.037  INFO 21164 --- [       runner-0] o.s.boot.SpringApplication               : Started application in 2.451 seconds (JVM running for 4.52)
```

---

### 4. 高级用法

上面我们可以发现，Spring Boot CLI的作用就是让我们不去考虑任何和Spring Boot无关的东西，依赖，构建和编译都不需要我们考虑，专注与Spring Boot本身，但是如果功能复杂的话，还是需要自己来修改例如指定依赖等配置。

```@Grab```注解是用于添加依赖的，添加到任意类上都可以，它有以下四种使用方法。
```
// 指定依赖
@Grab(group="com.h2database", module="h2", version="1.4.190")
// 使用冒号分割指定依赖
@Grab(group="com.h2database:h2:1.4.190")
// 指定依赖，使用默认版本，不同版本的Spring Boot CLI关联不同版本的依赖包
@Grab(group="com.h2database:h2")
// 只使用moudle名字指定依赖，使用默认版本
@Grab(group="h2")
```


```@GrabMetadata```注解是配合配置文件使用的，例如下面这个写法表示从Maven仓库中的com.myorg目录读取名为custom-versions.properties文件。

```
@GrabMetadata ("com.myorg:custom-versions:1.0.0")
```

文件的格式名字应该是下面这样，GroupId和ModuleId为键名，版本号是值。

```
com.h2database:h2=1.4.186
```

### 5. 构建部署

Spring Boot CLI虽然支持直接读取Groovy脚本来运行，但是它也支持打包操作，可以达成jar包或者war包来进行部署，执行```spring jar```命令即可打出常规jar包。

```
E:\spring-2.4.0\code>spring jar test.jar \
E:\spring-2.4.0\code>dir
 驱动器 E 中的卷是 开发辅助
 卷的序列号是 6E0F-553A

 E:\spring-2.4.0\code 的目录

2020/11/27  23:34    <DIR>          .
2020/11/27  23:34    <DIR>          ..
2020/11/27  23:14               131 test.groovy
2020/11/27  23:34        22,915,378 test.jar
2020/11/27  23:34             8,730 test.jar.original
               3 个文件     22,924,239 字节
               2 个目录 198,627,307,520 可用字节
```


---


### 6. 命令参数

使用```spring --help```命令就可以看到所有支持的命令参数了，grab是从指定仓库下载依赖包，install是安装依赖包到lib/ext下，unistall是卸载依赖包，init则是初始化Spring Boot项目，有了这些可以更加高效的开发，还可以使用```spring help jar```这样的命令来查看单个命令的使用说明文档。

```
E:\spring-2.4.0\code>spring --help
usage: spring [--help] [--version]
       <command> [<args>]

Available commands are:

  run [options] <files> [--] [args]
    Run a spring groovy script

  grab
    Download a spring groovy script's dependencies to ./repository

  jar [options] <jar-name> <files>
    Create a self-contained executable jar file from a Spring Groovy script

  war [options] <war-name> <files>
    Create a self-contained executable war file from a Spring Groovy script

  install [options] <coordinates>
    Install dependencies to the lib/ext directory

  uninstall [options] <coordinates>
    Uninstall dependencies from the lib/ext directory

  init [options] [location]
    Initialize a new project using Spring Initializr (start.spring.io)

  encodepassword [options] <password to encode>
    Encode a password for use with Spring Security

  shell
    Start a nested shell

Common options:

  --debug Verbose mode
    Print additional status information for the command you are running


See 'spring help <command>' for more information on a specific command.
```