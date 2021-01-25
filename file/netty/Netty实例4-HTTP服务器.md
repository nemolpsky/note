### Netty实例4-HTTP服务器

其实最常用的网络通信就是浏览器浏览页面，浏览页面的操作本质上就是把浏览器当做客户端，通过域名访问各种各样的服务端，Netty有很多集成好的HTTP工具，可以实现基于HTTP协议的服务端，下面这段代码在访问之后就会返回一个最简单的HTML页面。



```
public class HttpServer {

    public static void main(String[] args) throws Exception {

        EventLoopGroup acceptGroup = new NioEventLoopGroup(1);
        EventLoopGroup handlerGroup = new NioEventLoopGroup(2);

        try {
            ServerBootstrap bootstrap = new ServerBootstrap();
            bootstrap.group(acceptGroup, handlerGroup)
                    .channel(NioServerSocketChannel.class)
                    .handler(new LoggingHandler(LogLevel.INFO))
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel socketChannel) throws Exception {
                            ChannelPipeline pipeline = socketChannel.pipeline();
                            // 解码HTTP请求
                            pipeline.addLast(new HttpServerCodec());
                            // 根据上面的解码数据把HTTP请求封装成FullHttpRequest对象
                            pipeline.addLast(new HttpObjectAggregator(65536));
                            pipeline.addLast(new ChannelInboundHandlerAdapter() {
                                @Override
                                public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
                                    FullHttpRequest request = (FullHttpRequest) msg;
                                    // 打印HTTP请求信息
                                    System.out.println(request.toString());
                                    // 生成一个HTML页面
                                    StringBuilder builder = new StringBuilder();
                                    builder.append("<html>")
                                            .append("<head>").append("<meta charset=\"utf-8\"/>")
                                            .append("<title>title</title>")
                                            .append("<body>")
                                            .append("<a> This a simple Http response.").append("</a>")
                                            .append("</body>")
                                            .append("</html>");
                                    // 创建HTTP响应对象
                                    FullHttpResponse response = new DefaultFullHttpResponse(HTTP_1_1, OK
                                            , Unpooled.wrappedBuffer(builder.toString().getBytes()));
                                    // 设置请求头
                                    response.headers().set(CONTENT_TYPE, "text/html")
                                            .setInt(CONTENT_LENGTH, response.content().readableBytes());
                                    // 返回响应信息
                                    ChannelFuture future = ctx.writeAndFlush(response);
                                    if (HttpUtil.isKeepAlive(request)) {
                                        future.addListener(ChannelFutureListener.CLOSE);
                                    }
                                }
                            });
                        }
                    });
            Channel channel = bootstrap.bind(8888).sync().channel();
            channel.closeFuture().sync();
        } finally {
            // 释放资源
            acceptGroup.shutdownGracefully();
            handlerGroup.shutdownGracefully();
        }
    }
}
```

核心代码还是在处理链的编写，入站信息是从处理链的头部开始，```HttpServerCodec```是用于解析HTTP请求中的信息，```HttpObjectAggregator```则是把每个HTTP请求信息包装成一个```FullHttpRequest```对象，最后再生成一个```FullHttpResponse```对象作为响应，如果你对HTTP协议熟悉的话，就会明白HTTP协议本质上就是按照既定格式进行交互的文本信息。注意```HttpServerCodec```比较特殊，它即是出占处理器，也是入站处理器，所以它会把```FullHttpResponse```对象转换成字节，这也就是为什么浏览器可以识别这个HTML然后显示了。

```
@Override
protected void initChannel(SocketChannel socketChannel) throws Exception {
    ChannelPipeline pipeline = socketChannel.pipeline();
    // 解码HTTP请求
    pipeline.addLast(new HttpServerCodec());
    // 根据上面的解码数据把HTTP请求封装成FullHttpRequest对象
    pipeline.addLast(new HttpObjectAggregator(65536));
    pipeline.addLast(new ChannelInboundHandlerAdapter() {
        @Override
        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
            FullHttpRequest request = (FullHttpRequest) msg;
            // 打印HTTP请求信息
            System.out.println(request.toString());
            // 生成一个HTML页面
            StringBuilder builder = new StringBuilder();
            builder.append("<html>")
                    .append("<head>").append("<meta charset=\"utf-8\"/>")
                    .append("<title>title</title>")
                    .append("<body>")
                    .append("<a> This a simple Http response.").append("</a>")
                    .append("</body>")
                    .append("</html>");
            // 创建HTTP响应对象
            FullHttpResponse response = new DefaultFullHttpResponse(HTTP_1_1, OK
                    , Unpooled.wrappedBuffer(builder.toString().getBytes()));
            // 设置请求头
            response.headers().set(CONTENT_TYPE, "text/html")
                    .setInt(CONTENT_LENGTH, response.content().readableBytes());
            // 返回响应信息
            ChannelFuture future = ctx.writeAndFlush(response);
            if (HttpUtil.isKeepAlive(request)) {
                future.addListener(ChannelFutureListener.CLOSE);
            }
        }
    });
}

```