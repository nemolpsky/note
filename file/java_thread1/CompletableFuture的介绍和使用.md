### 1. CompletableFuture介绍

CompletableFuture是Java 8新增的一个类库，可以看作是Future的升级版，Future是在使用子线程的时候可以获取子线程的执行结果，也可以阻塞等待子线程执行完。而CompletableFuture则是在这个基础上新增了回调接口，同时还可以合并多个线程的执行结果，当然也支持函数式编程，在复杂情况下的多线程执行时进行流程的控制交互相当方便。


---


### 2. 正常回调

下面调用了supplyAsync方法，这是一个异步回调方法，执行之后可以观察日志看到当前get1方法不会阻塞，会在默认的线程池中执行，执行完之后就会在回调方法thenAccept中打印结果。

```
@Service
@Slf4j
public class FutureService {

    @SneakyThrows
    public String execute1(){
        TimeUnit.SECONDS.sleep(5);
        log.info("futureService execute1.");
        return "execute1";
    }

    @SneakyThrows
    public String execute2(){
        TimeUnit.SECONDS.sleep(3);
        int i = 1/0;
        log.info("futureService execute2.");
        return "execute2";
    }

    @SneakyThrows
    public String execute3(){
        TimeUnit.SECONDS.sleep(3);
        log.info("futureService execute3.");
        return "execute3";
    }
}

@RestController
@RequestMapping(value = "/future")
@Slf4j
public class FutureController {

    @Autowired
    private FutureService futureService;

    @GetMapping(value = "/get1")
    public void get1() {
        log.info("get1 start.");
        CompletableFuture<String> future = CompletableFuture.supplyAsync(futureService::execute1);
        future.thenAccept(System.out::println);
        log.info("get1 over.");
    }
}

```

---

### 3. 异常回调

这个方法跟上面相比，只是多了一个exceptionally方法，这个方法是在执行子线程出错的时候才会回调。

```
    @GetMapping(value = "/get2")
    public void get2() {
        log.info("get2 start.");
        CompletableFuture<String> future = CompletableFuture.supplyAsync(futureService::execute2);
        future.thenAccept(System.out::println);
        future.exceptionally(e -> {
            System.out.println(e.getLocalizedMessage());
            return null;
        });
        log.info("get2 over.");
    }
```

---

### 4. 串行执行

这其实也算是一个多线程中比较常见的需求，多个任务串行执行，任务之间会有参数依赖，换做以前最简单的办法就是阻塞当前线程等待任务一个个执行完，如果是要多线程实现这种串行还是比较麻烦的，但是CompletableFuture提供了thenApplyAsync方法，直接将上面的任务结果作为参数执行下一个任务，也是异步的，不会阻塞当前线程，也不需要我们去实现这种多线程的串行。

```
    @GetMapping(value = "/get3")
    public void get3() {
        log.info("get3 start.");

        CompletableFuture<String> future1 = CompletableFuture.supplyAsync(futureService::execute1);
        CompletableFuture<String> future3 = future1.thenApplyAsync( result1 -> result1 + "|" + futureService.execute3());
        future3.thenAccept(System.out::println);

        log.info("get3 over.");
    }
```

---

### 5. 并行执行

anyOf方法作用是多个任务之间只要有一个任务完成，就立刻返回结果。

```
    @GetMapping(value = "/get4")
    public void get4() {
        log.info("get4 start.");

        CompletableFuture<String> future1 = CompletableFuture.supplyAsync(futureService::execute1);
        CompletableFuture<String> future3 = CompletableFuture.supplyAsync(futureService::execute3);
        CompletableFuture<Object> allResult = CompletableFuture.anyOf(future1,future3);

        allResult.thenAccept(System.out::println);
        log.info("get4 over.");
    }
```

---

### 6. 合并结果

上面的anyOf方法是只要有一个任务完成就返回结果，下面allOf则是等待所有的结果都完成，还可以继续进行这样的操作。

```
    @SneakyThrows
    @GetMapping(value = "/get5")
    public void get5() {
        log.info("get5 start.");

        CompletableFuture<String> future1 = CompletableFuture.supplyAsync(futureService::execute1);
        CompletableFuture<String> future3 = CompletableFuture.supplyAsync(futureService::execute3);
        CompletableFuture<Void> result = CompletableFuture.allOf(future1,future3);

        String r1 = result.thenApply(v -> {
            String r = Stream.of(future1, future3).map(CompletableFuture::join)
                    .collect(Collectors.joining("|"));
            log.info("result:{}", r);
            return r;
        }).get();
        log.info("result1:{}", r1);

        CompletableFuture<String> future4 = result.thenApplyAsync(str -> r1 + "|" + futureService.execute1());
        CompletableFuture<String> future5 = result.thenApplyAsync(str -> r1 + "|" + futureService.execute3());
        CompletableFuture<Void> last = CompletableFuture.allOf(future4,future5);
        last.thenAccept(v -> {
            log.info("last:{}", Stream.of(future4, future5).map(CompletableFuture::join)
                    .collect(Collectors.joining("|")));
        });
        log.info("get5 over.");
    }
```

---

### 总结：

Java 8最核心的变更就是函数式编程了，以致于很多新加的类库也是必须以函数式语法去调用，目的是因为函数式更加清晰和简介，其本质是对接口的一种抽象，所以也减少了开发的工作量，比如上面的CompletableFuture很多方法返回的都是CompletableFuture的引用，而很多方法接受的参数也是CompletableFuture的引用，这样就不需要开发者再进行中间的中转，把注意力更加专注于功能本身。

Java 8新增的很多有用的函数式的接口方法也都可以在CompletableFuture中应用。