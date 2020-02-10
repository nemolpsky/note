## Java NIO

NIO是non-blocking I/O的简称，是对于BIO的一种改进，是同步不阻塞的，基于Reactor模式，也就是事件驱动的。

---

### 结构

跟BIO不同的地方就是它可以配合一个多路复用器来轮询判断是否有IO操作要看开始了，在Jdk1.4中引入了Channel(通道)和Selector(选择器)的概念，就是由Selector来轮询判断有哪个Channel需要开始准备进行数据传输了，然后再开启线程进行IO操作，这也就是IO多路复用。

![NIO](https://github.com/nemolpsky/note/raw/master/file/io/image/io2.png)


### 例子

可以看到就是通过Selector来轮询判断哪个Channel已经准备好就绪了，然后把它修改为读取状态再开始读取，整个过程并没有开启多线程。

- Server代码
      
  ```
      public class Server4 {

          public static void main(String[] args) throws IOException {
              Selector selector = Selector.open();

              ServerSocketChannel server = ServerSocketChannel.open();
              server.socket().bind(new InetSocketAddress(9999));

              // 将其注册到 Selector 中，监听 OP_ACCEPT 事件
              server.configureBlocking(false);
              server.register(selector, SelectionKey.OP_ACCEPT);

              while (true) {
                  int readyChannels = selector.select();
                  if (readyChannels == 0) {
                      continue;
                  }
                  Set<SelectionKey> readyKeys = selector.selectedKeys();
                  // 遍历
                  Iterator<SelectionKey> iterator = readyKeys.iterator();
                  while (iterator.hasNext()) {
                      SelectionKey key = iterator.next();
                      iterator.remove();

                      if (key.isAcceptable()) {
                          // 有已经接受的新的到服务端的连接
                          SocketChannel socketChannel = server.accept();

                          // 有新的连接并不代表这个通道就有数据，
                          // 这里将这个新的 SocketChannel 注册到 Selector，监听 OP_READ 事件，等待数据
                          socketChannel.configureBlocking(false);
                          socketChannel.register(selector, SelectionKey.OP_READ);
                      } else if (key.isReadable()) {
                          // 有数据可读
                          // 上面一个 if 分支中注册了监听 OP_READ 事件的 SocketChannel
                          SocketChannel socketChannel = (SocketChannel) key.channel();
                          ByteBuffer readBuffer = ByteBuffer.allocate(1024);
                          int num = socketChannel.read(readBuffer);
                          if (num > 0) {
                              // 处理进来的数据...
                              System.out.println("收到数据：" + new String(readBuffer.array()).trim());
                              ByteBuffer buffer = ByteBuffer.wrap("返回给客户端的数据...".getBytes());
                              socketChannel.write(buffer);
                          } else if (num == -1) {
                              // -1 代表连接已经关闭
                              socketChannel.close();
                          }
                      }
                  }
              }
          }
      }
  ```

- Client代码

  ```

      public class Client3 {

          public static void main(String[] args) throws Exception {
              SocketChannel socketChannel = SocketChannel.open();
              socketChannel.connect(new InetSocketAddress("localhost", 9999));

              new Thread(new Runnable() {
                  @Override
                  public void run() {
                      try {
                          writeAndRead(socketChannel,true,"message1");
                      } catch (Exception e) {
                          e.printStackTrace();
                      }
                  }
              }).start();

              writeAndRead(socketChannel,false,"message2");
          }

          public static void writeAndRead(SocketChannel socketChannel,boolean isSleep,String message) throws Exception {
              // 写操作
              ByteBuffer buffer = ByteBuffer.wrap((message).getBytes());
              socketChannel.write(buffer);

              if (isSleep){
                  TimeUnit.SECONDS.sleep(5);
              }

              // 读操作
              ByteBuffer readBuffer = ByteBuffer.allocate(1024);
              int num;
              if ((num = socketChannel.read(readBuffer)) > 0) {
                  readBuffer.flip();

                  byte[] re = new byte[num];
                  readBuffer.get(re);

                  String result = new String(re, "UTF-8");
                  System.out.println("返回值: " + result);
              }
          }
      }

  ```

---


## 总结

NIO模式本身是不堵塞的，所以服务端会有一个循环来判断是否有请求过来，而多路复用则是会有一个专门的线程来操作所谓的多路复用器来轮询，判断是否需要开始进行IO操作了，然后就会真正的执行IO操作，而多路复用器再没有操作过来的时候也是阻塞住的，不会想简单的while循环那样空转，造成CPU性能浪费。而不是像以前BIO模式一样，无论是不是有IO操作，只要请求过来连接上了就一直阻塞在那里，极大的解决了BIO下大量线程的高性能消耗，但是如果并发量太高也是会造成阻塞，因为虽然有专门的的多路复用器来轮询，不会堵塞当前线程，但是大量IO操作如果经过了多路复用器到达后面开始操作IO的线程中，数量多了一样会耗尽线程资源。
