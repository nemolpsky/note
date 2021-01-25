### Netty实例1-丢弃服务器

Netty作为一个异步事件驱动的网络框架，性能非常好，被应用到的地方也很多，比如dubbo，Hadoop，Zookeeper和RocketMQ这些很出名的第三方框架和中间件都使用了Netty来实现通信。

其实本质上Netty是基于Java的NIO进行了封装和优化，再加上定位于网络框架，使用了合理的设计模式，所以在使用Netty的时候会很简单，耦合性也会很低。不过需要注意的是，网络通信本质上就是网络请求，也就是基于各种不同的协议对数据进行传输，所以如果还原到Java最初的本质，实际上就是使用Socket和IO流利用网络进行数据传输，不过它们在Java中其实都位于比较底层的位置，使用都很繁琐，Netty就是提供自己的封装和设计，让你可以简单高效的进行网络通信。

不过虽然这样，但是使用Netty还是有必要打牢基础，至少要熟悉IO流和原生socket这些东西，Netty都是在这些原有概念上的一种优化和升级，如果没有这些基础，很难学会Netty。


下面这个例子是官网上最简单的一个例子，丢弃服务器，就是接到消息之后立刻把消息丢弃掉，不会做出任何响应。启动之后，使用```telnet localhost 21001```连接，再输入字母就可以看到服务器接收到这些字母了，所谓的网络通信就是这个意思，如果你把这个服务放在一台服务器上，然后通过域名远程访问也可以访问到。

```
/**
 * 丢弃服务器，接收到消息之后不做任何响应，使用telnet命令可以看到输入的字母
 * 使用浏览器请求则只会接收到一个大写的G，也就是HTTP协议第一个字母
 */
public class DiscardServer {
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
                                    // ByteBuf是Netty实现的一个高性能的传输缓存，打印ByteBuf对象，读取的字节，字节转换后的字母
                                    ByteBuf byteBuf = (ByteBuf) msg;
                                    try {
                                        System.out.println(byteBuf.toString());
                                        byte b = byteBuf.readByte();
                                        System.out.println(b);
                                        System.out.println((char) Integer.valueOf(b + "").intValue());
                                    }finally {
                                        // 如果上面的处理失败，则直接释放消息占用的内存
                                        ReferenceCountUtil.release(msg);
                                    }
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
                    })
                    // 这里childOption设置针对NioServerSocketChannel，option针对它的父类
                    // 因为它是基于TCP/IP协议，所以可以设置传送层的一些配置，比如队列连接超过1000拒绝，保持长连接等等
                    // https://netty.io/4.1/api/io/netty/channel/ChannelOption.html
                    // https://netty.io/4.1/api/io/netty/channel/ChannelConfig.html
                    .option(ChannelOption.SO_BACKLOG, 1000)
                    .childOption(ChannelOption.SO_KEEPALIVE, true);

            // 绑定端口
            ChannelFuture future = bootstrap.bind(21001).sync();
            // 服务的套接字关闭的时候关闭服务，不过这里不会主动关闭套接字
            // 这个设置可以保证没有连接时服务器也不会自动关闭
            future.channel().closeFuture().sync();
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


其余的代码都没什么可说的，模板代码，不同的需求就配置不同的类就行。这段比较重要，因为Netty的设计模式是责任链模式，把所有的处理器连成一个责任链，然后一次对消息进行处理。```ChannelInboundHandlerAdapter```类是入站消息的处理器，实现了```ChannelInboundHandler```接口，最顶层是```ChannelHandler```。这些接口就干一件事，定义各种各样的回调方法，其中名字带```Inbound```的全是入站消息，就是接收到的消息，带```outbound```都是出站消息，就是要发送出去的消息。

这里需要读取消息然后打印，所以重写了```channelRead```回调方法，接收到消息时就会调用这个方法。

```ByteBuf```是Netty自己实现的一个数据容器，专门用来存放消息，性能也很好，所以消息的获取和存放都是放在```ByteBuf```里面，比如下面这段代码，读取成功后就会自动释放掉读取的```ByteBuf```，如果没有释放，消息会按照责任链顺序传到下一个处理器去。

```
// 初始化管道，添加各种各样的处理器
@Override
protected void initChannel(SocketChannel socketChannel) throws Exception {
    // 添加一个处理器
    socketChannel.pipeline().addLast(new ChannelInboundHandlerAdapter() {
        // 接收到消息的回调，对消息作各种处理
        @Override
        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
            // ByteBuf是Netty实现的一个高性能的传输缓存，打印ByteBuf对象，读取的字节，字节转换后的字母
            ByteBuf byteBuf = (ByteBuf) msg;
            try {
                System.out.println(byteBuf.toString());
                byte b = byteBuf.readByte();
                System.out.println(b);
                System.out.println((char) Integer.valueOf(b + "").intValue());
            }finally {
                // 如果上面的处理失败，则直接释放消息占用的内存
                ReferenceCountUtil.release(msg);
            }
        }
    }
}
```