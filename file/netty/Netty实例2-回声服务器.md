### Netty实例2-回声服务器

丢弃服务器让你学会怎么去接收读取消息，回升服务器则是教你怎么写入返回消息。

```
/**
 * 回声服务器，接收到消息之后直接回传消息
 */
public class EchoServer {
    public static void main(String[] args) {
        // Group本质上是基于事件响应的线程池
        // acceptGroup用于接收请求，注册到handlerGroup中
        EventLoopGroup acceptGroup = new NioEventLoopGroup(1);
        // handlerGroup则会真正的处理这些请求的读写操作
        EventLoopGroup handlerGroup = new NioEventLoopGroup(2);
        try {
            // 服务引导器，快速创建一个服务
            ServerBootstrap bootstrap = new ServerBootstrap();
            // 给服务设置两个group
            bootstrap.group(acceptGroup, handlerGroup)
                    // 添加处理连接的channel类，不同的channel对应不同的类型
                    // NioServerSocketChannel是基于NIO类型处理连接
                    // 它的传输层协议是基于TCP/IP进行传输，具体参考https://netty.io/3.8/guide/#architecture.7
                    .channel(NioServerSocketChannel.class)
                    // 设置管道，Netty是基于责任链模式，管道可以看作一个责任链
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        // 初始化管道，添加各种各样的处理器
                        @Override
                        protected void initChannel(SocketChannel socketChannel) throws Exception {
                            // 添加一个处理器
                            socketChannel.pipeline().addLast(new ChannelInboundHandlerAdapter() {
                                // 接收到消息的回调，对消息作各种处理
                                @Override
                                public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
                                    ctx.writeAndFlush(msg);
                                }

                                // 处理异常的回调
                                @Override
                                public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
                                    System.out.println("handler error!");
                                    cause.printStackTrace();
                                    ctx.close();
                                }
                            });
                        }
                    });
            // 绑定端口
            bootstrap.bind(21002).sync().channel().closeFuture().sync();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            // 释放资源
            acceptGroup.shutdownGracefully();
            handlerGroup.shutdownGracefully();
        }
    }
}

```

可以看到除了接收消息的回调方法里面的逻辑有变化，其它都没变，```ctx.writeAndFlush(msg);```这行就把接收到的消息原封不动返回给客户端了，所以使用```telnet```命令连接之后输入字母，会看到立刻返回一个相同的字母。所以从这两个例子来看可以总结出，数据都要放到```ByteBuf```中，处理消息依靠各种处理器，以及处理器的回调方法，按照责任链模式处理。

```
// 初始化管道，添加各种各样的处理器
@Override
protected void initChannel(SocketChannel socketChannel) throws Exception {
    // 添加一个处理器
    socketChannel.pipeline().addLast(new ChannelInboundHandlerAdapter() {
        // 接收到消息的回调，对消息作各种处理
        @Override
        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
            ctx.writeAndFlush(msg);
        }
    }
}
```

如果想要对数据处理下再返回，还可以写成这样。
```
ctx.writeAndFlush(Unpooled.wrappedBuffer("收到消息了".getBytes()));
```