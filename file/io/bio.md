## Java BIO

BIO是Blocking I/O的简称，也就是同步阻塞的I/O模式，是JDK1.4版本以前的主要操作I/O的方式。


---

### 单线程模式

- 模型图

  以Socket连接为例子，所有客户端的请求都会进入到```Acceptor```中，这是一个单线程的负责监听客户端连接的类，一般是以```while```循环的方式来判断是否有请求过来，

  ![BIO](https://github.com/nemolpsky/note/raw/master/file/io/image/bio1.png)

  下面这段代码就是一个标准的BIO例子，因为在服务端每次执行完请求都是通过轮询```serverSocket.accept()```来判断是否有请求进来，而如果恰巧这个IO流操作时间很久，那么线程就会一直堵塞在这里，直到IO操作完成之后这个线程才能做别的事情。

- 代码例子

  ```
      public class Server {

          public static void main(String[] args) throws Exception {
              Server server = new Server();
              server.server();
          }

          public void server() throws Exception {
              // 初始化服务端，设置端口
              ServerSocket serverSocket = new ServerSocket(9999);
              Socket socket = null;
              InputStream inputStream = null;
              try {
                  // 隔一秒调用 serverSocket.accept()方法检查是否有请求
                  while (true) {
                      socket = serverSocket.accept();
                      TimeUnit.SECONDS.sleep(1);

                      // 获得输入流
                      inputStream = socket.getInputStream();
                      byte[] bytes = new byte[1024];
                      int len;
                      StringBuilder sb = new StringBuilder();
                      while ((len = inputStream.read(bytes)) != -1) {
                          sb.append(new String(bytes, 0, len, "UTF-8"));
                      }

                      // 输出内容
                      System.out.println(sb.toString() + "-" + LocalTime.now());
                  }
              } finally {
                  socket.close();
                  inputStream.close();
              }
          }
      }

      public class Client {

          public static void main(String[] args) throws Exception{
              Client client = new Client();
              client.client();
          }

          public void client(){
              // 循环20个请求
              for(int i=0;i<20;i++) {
                  // 连接服务端
                  try (Socket socket = new Socket("127.0.0.1", 9999);
                  OutputStream outputStream = socket.getOutputStream();) {
                      // 传输字符串
                      outputStream.write("test message".getBytes("UTF-8"));
                      outputStream.flush();
                  } catch (Exception e) {
                      e.printStackTrace();
                  }
              }
          }
      }
  ```

---

### 多线程模式
    
- 模型图

  多线程的模式就是每次服务端接受到请求之后就把耗时操作放到一个新的子线程中去执行，这样就是并发执行了，效率就会高很多。

  ![BIO](https://github.com/nemolpsky/note/raw/master/file/io/image/bio2.png)

- 代码例子

  多线程的模式可以看到就是在上面单线程的模式上使用了线程池，这样既避免了大量新建线程的操作耗费性能，也可以充分解决单线程的阻塞问题。

  ```
      class MyThread implements Runnable {

          private InputStream inputStream;

          public MyThread(InputStream inputStream) {
              this.inputStream = inputStream;
          }

          @Override
          public void run() {
              // 获得输入流
              byte[] bytes = new byte[1024];
              int len;
              StringBuilder sb = new StringBuilder();
              try {
                  TimeUnit.SECONDS.sleep(1);
                  while ((len = inputStream.read(bytes)) != -1) {
                      sb.append(new String(bytes, 0, len, "UTF-8"));
                  }
              } catch (Exception e) {
                  e.printStackTrace();
              } finally {
                  try {
                      inputStream.close();
                  } catch (IOException e) {
                      e.printStackTrace();
                  }
              }
              // 输出内容
              System.out.println(sb.toString() + "-" + LocalTime.now() + "-" + Thread.currentThread().getName());
          }
      }
      ```

      ```
      public class Server {

          static ThreadPoolExecutor executor = null;

          public static void main(String[] args) throws Exception {

              // 工作线程数为20
              Integer corePoolSize = 20;
              // 最大线程数为30
              Integer maximumPoolSize = 30;
              // 空闲线程存活时间10秒
              Integer keepAliveTime = 10;
              TimeUnit unit = TimeUnit.SECONDS;
              // 队列
              ArrayBlockingQueue<Runnable> workQueue = new ArrayBlockingQueue<>(1);
              ThreadFactory factory = Executors.defaultThreadFactory();
              // 使用AbortPolicy策略，该策略将会直接抛出RejectedExecutionException异常
              RejectedExecutionHandler abortHandler = new ThreadPoolExecutor.AbortPolicy();
              // 创建线程池
              executor = new ThreadPoolExecutor(corePoolSize, maximumPoolSize, keepAliveTime, unit,workQueue, factory, abortHandler);

              Server server = new Server();
              server.server();
          }

          public void server() throws Exception {
              // 初始化服务端，设置端口
              ServerSocket serverSocket = new ServerSocket(9999);
              Socket socket = null;
              try {
                  // 隔一秒调用 serverSocket.accept()方法检查是否有请求
                  while (true) {
                      socket = serverSocket.accept();
                      async(socket.getInputStream());
                  }
              }catch (Exception e){
                  e.printStackTrace();
              }
          }

          // 将耗时操作放入线程池
          public static void async(InputStream inputStream){
              executor.execute(new MyThread(inputStream));
          }
      }
  ```



---

## 总结

BIO模式其实就是说所有的操作都是链式的，就是某个环节是阻塞的，如果是单线程的情况下有一个请求在这个环节进行了耗时操作，那么当前线程就会把所有的请求都会堵死在那里，所以可以使用线程池来优化，但是使用多线程也不是没有问题，因为在实际应用中过多的线程带来的频繁切换会很耗费性能，使整体速度运行更慢，而且大量的线程对系统的内存负担也大，所以后面会介绍NIO模型来解决这些问题。