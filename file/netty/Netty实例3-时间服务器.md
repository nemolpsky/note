### Netty实例3-时间服务器

这个例子和前面两个例子不太一样，接收到连接之后会立刻返回一个时间戳数据，然后关闭连接。

```
/**
 * 时间服务器，接收到请求之后立刻返回一个时间
 */
public class TimeServer {
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

                                // 当一个连接建立好时会回调此接口
                                @Override
                                public void channelActive(ChannelHandlerContext ctx) throws Exception {
                                    // 分配一个缓存区，占4个字节，写入32位的int值
                                    final ByteBuf byteBuf = ctx.alloc().buffer(4);
                                    byteBuf.writeInt((int) (System.currentTimeMillis() / 1000L + 2208988800L));
                                    // 写入数据并冲刷出栈，注意Netty中所有操作都是异步的
                                    // 如果此时立刻关闭连接可能还没写完，因此设置一个回调接口，消息写完了才关闭连接
                                    ChannelFuture future = ctx.writeAndFlush(byteBuf);
                                    future.addListener(ChannelFutureListener.CLOSE);
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
                    }).childOption(ChannelOption.SO_KEEPALIVE, true);
            // 绑定端口
            bootstrap.bind(21003).sync().channel().closeFuture().sync();
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


核心就是```channelActive```方法，它也是一个回调方法，不同的是它并不是接收到消息之后会调用，而是有一个新连接建立时会调用。这里创建了一个占4个字节的```ByteBuf```，然后写入一个时间戳，注意了，这里写入之后会获取一个```ChannelFuture```，在Netty中，所有的操作都是异步的，所以写入之后直接关闭连接，可能还没有写完，因此添加一个关闭的回调接口，写入完才会关闭连接。
```
// 当一个连接建立好时会回调此接口
@Override
public void channelActive(ChannelHandlerContext ctx) throws Exception {
    // 分配一个缓存区，占4个字节，写入时间戳
    final ByteBuf byteBuf = ctx.alloc().buffer(4);
    byteBuf.writeInt((int) (System.currentTimeMillis() / 1000L + 2208988800L));
    // 写入数据并冲刷出栈，注意Netty中所有操作都是异步的
    // 如果此时立刻关闭连接可能还没写完，因此设置一个回调接口，消息写完了才关闭连接
    ChannelFuture future = ctx.writeAndFlush(byteBuf);
    future.addListener(ChannelFutureListener.CLOSE);
}

```

最后还需要一个客户端，因为直接把数据写入到```ByteBuf```中其实本质上是往内存上写值，而不是把值赋予给变量，这两者之间是有区别的，所以如果直接使用```telnet```命令连接会发现是打印乱码，所以需要一个客户端读取```ByteBuf```里的数据再转换成有意义的数值。可以看到客户端其实和服务端一样的模式，只不过个别地方的配置不一样。

```
public class TimeClientV1 {
    public static void main(String[] args) {
        EventLoopGroup group = new NioEventLoopGroup();
        try {
            // 客户端引导器，快速创建一个客户端
            Bootstrap bootstrap = new Bootstrap();
            // 设置group
            bootstrap.group(group)
                    // 添加处理连接的channel类，不同的channel对应不同的类型
                    // NioSocketChannel基于NIO流
                    .channel(NioSocketChannel.class)
                    // 设置管道，Netty是基于责任链模式，管道可以看作一个责任链
                    .handler(new ChannelInitializer<SocketChannel>() {
                        // 初始化管道，添加各种各样的处理器
                        @Override
                        protected void initChannel(SocketChannel socketChannel) throws Exception {
                            // 添加一个处理器
                            socketChannel.pipeline().addLast(new ChannelInboundHandlerAdapter() {
                                @Override
                                public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
                                    // 获取传输的ByteBuf，按格式转换数据
                                    ByteBuf byteBuf = (ByteBuf) msg;
                                    try {
                                        long currentTimeMillis = (byteBuf.readUnsignedInt() - 2208988800L) * 1000L;
                                        System.out.println(new Date(currentTimeMillis));
                                        ctx.close();
                                    } finally {
                                        byteBuf.release();
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
                    }).option(ChannelOption.SO_KEEPALIVE, true);;

            // 连接远程端口和地址
            bootstrap.connect("localhost", 21003).sync().channel().closeFuture().sync();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            // 释放资源
            group.shutdownGracefully();
        }
    }
}
```

在客户端里面也是一样的责任链模式，因为本质上无论是服务端还是客户端，其实就可以理解为传输数据的两个端点，没有什么不同。这里也是在```channelRead```回调方法里处理接收到的数据，```byteBuf.readUnsignedInt()```就是把```ByteBuf```中的数据读取成一个int值，其实可以理解为一次解码，解完码之后数据就是可读的了。客户端控制台上就会显示服务端返回的时间了，而且因为服务端设置了写完数据之后立刻关闭连接，所以客户端会立刻断开。

```
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    // 获取传输的ByteBuf，按格式转换数据
    ByteBuf byteBuf = (ByteBuf) msg;
    try {
        long currentTimeMillis = (byteBuf.readUnsignedInt() - 2208988800L) * 1000L;
        System.out.println(new Date(currentTimeMillis));
        ctx.close();
    } finally {
        byteBuf.release();
    }
}
```

实际上如果使用TCP协议来传输数据的话会有一个问题，俗称粘包，其实专业术语汇总没有这个说法，本质原因是例如Nagle算法这一类算法合并多次TCP传输的数据包，而且套接字的缓冲区实际上就是一个字节队列，如果没有适当的分割，可能就会获取到错误的字节，比如服务端返回的是一个4字节的数据，那每次接收消息也应该按4个字节来截取。

最简单的方式肯定就是直接在接收到消息之后进行分配的逻辑判断，但是Netty提供了更好的方式，因为Netty是责任链模式，那么只需要在责任链上再添加一个处理器专门处理就可以，```ByteToMessageDecoder```就是专门处理字节转换成字符的处理器，其实它也继承了```ChannelInboundHandlerAdapter```，所以这就是一个又封装了一层的处理器。所以在这个处理器里面依照4字节分割，传递给下一个字节回显即可。

```
@Override
protected void initChannel(SocketChannel socketChannel) throws Exception {
    // 添加一个处理器
    socketChannel.pipeline()
            .addLast(new ByteToMessageDecoder() {
                @Override
                protected void decode(ChannelHandlerContext channelHandlerContext, ByteBuf byteBuf
                        , List<Object> list) throws Exception {
                    System.out.println("decode");
                    // 可读取的字节小于4就继续读取
                    if (byteBuf.readableBytes() < 4) {
                        return;
                    }
                    // 不小于4就表示已经读取完一条消息，传递给下一个处理器，并且丢弃这个消息占用的缓存
                    list.add(byteBuf.readBytes(4));
                }
            })
            .addLast(new ChannelInboundHandlerAdapter() {
                @Override
                public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
                    System.out.println("channelRead");
                    // 获取传输的ByteBuf，按格式转换数据
                    ByteBuf byteBuf = (ByteBuf) msg;
                    try {
                        long currentTimeMillis = (byteBuf.readUnsignedInt() - 2208988800L) * 1000L;
                        System.out.println(new Date(currentTimeMillis));
                        ctx.close();
                    } finally {
                        byteBuf.release();
                    }
                }
            });
｝
```


其实服务端还可以把消息类型改写成自己定义的对象，方便操作，就是本质就是自己添加一个处理器把```ByteBuf```中的数据转换成对应的对象即可。可以看到在```channelActive```方法里面写入的是```UnixTime```对象，这个对象核心属性就是value字段，但是如果直接这样传输客户端必定是无法解码的，因为Netty所有的传输都是依赖```ByteBuf```，所以添加了一个```MessageToByteEncoder```用来把这个对象转换成```ByteBuf```，这样的好处就是在整个处理链上都可以直接处理对象，简单方便，直到最后要传输的时候才转换成字节存放到```ByteBuf```中。

```MessageToByteEncoder```是继承的```ChannelOutboundHandlerAdapter```，也就是出站的时候会用到的处理器，写入操作就是要把数据传输给客户端。注意这个处理链的添加顺序，一直使用```addLast```添加在链尾，而出站操作实际上是从处理链的尾部开始执行，所以```MessageToByteEncoder```是最后被执行的。

```
public class UnixTime {

    private final long value;

    public UnixTime() {
        this(System.currentTimeMillis() / 1000L + 2208988800L);
    }

    public UnixTime(long value) {
        this.value = value;
    }

    public long value() {
        return value;
    }

    @Override
    public String toString() {
        return new Date((value() - 2208988800L) * 1000L).toString();
    }
}
```
```
// 添加一个处理器
socketChannel.pipeline()
        .addLast(new MessageToByteEncoder<UnixTime>() {
            @Override
            protected void encode(ChannelHandlerContext ctx, UnixTime unixTime, ByteBuf byteBuf)
                    throws Exception {
                byteBuf.writeInt((int) unixTime.value());
            }
        })
        .addLast(new ChannelInboundHandlerAdapter() {

            // 当一个连接建立好时会回调此接口
            @Override
            public void channelActive(ChannelHandlerContext ctx) throws Exception {
                ChannelFuture future = ctx.writeAndFlush(new UnixTime());
                future.addListener(ChannelFutureListener.CLOSE);
            }
        });
    }
});
```

客户端也可以做相同的操作，把接受到的数据转换成对象，只不过使用的是```ByteToMessageDecoder```，继承的```ChannelInboundHandlerAdapter```，入站处理器，和出站操作相反，入站会从处理链的首部执行，先解码成对象，再给后面的处理器操作，```ByteToMessageDecoder```会被第一个执行。

```
socketChannel.pipeline()
        .addLast(new ByteToMessageDecoder() {
            @Override
            protected void decode(ChannelHandlerContext channelHandlerContext, ByteBuf byteBuf
                    , List<Object> list) throws Exception {
                // 可读取的字节小于4就继续读取
                if (byteBuf.readableBytes() < 4) {
                    return;
                }
                // 不小于4就表示已经读取完一条消息，传递给下一个处理器，并且丢弃这个消息占用的缓存
                list.add(new UnixTime(byteBuf.readUnsignedInt()));
            }
        })
        .addLast(new ChannelInboundHandlerAdapter() {
            @Override
            public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
                // 获取消息，转换成UnixTime对象
                UnixTime unixTime = (UnixTime)msg;
                System.out.println(unixTime.toString());
            }
        });
    }
});

```