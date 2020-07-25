### HTTP请求和TCP握手

#### 1. 网络分层
HTTP请求是日常中最常见的一种请求方式，也就是使用了HTTP协议(HyperText Transfer Protocol，超文本传输协议)作为规范。

在TCP/IP协议模型中，将网络分为了四层，要注意的是还有一个ISO/OSI模型，将网络分为七层，不过绝大部分实际应用中还是按照TCP/IP协议模型进行分层，而ISO/OSI模型则更多的作为一个参照标准。

网络分层可以看成是一种封装规范，类似于后端架构中的Controller层、Service层和Dao层一样，本质上都是类，里面都是执行逻辑的代码，但是每一层都有它专门的职责，这样能够更好的便于扩展和维护。

---

#### 2. 使用Wireshark抓包分析数据
Wireshark是一个免费的网络封包分析软件，功能非常强大，安装之后可以获取到本机上的各种不同协议请求的详细数据。

它还支持两种过滤器，分别过滤捕获的内容，以及捕获后显示的内容，详细使用方式不在这里介绍，本文主要依赖Wireshark对HTTP请求以及TCP三次握手和四次握手的过程进行分析。

下面是使用Postman请求接口，并使用Wireshark来抓取数据的结果。可以看到使用了```ip.addr == 148.70.108.183 and tcp```来过滤掉了其他无关的数据，可以看到最前面三条是tcp连接，也就是三次握手，然后分别是http请求和响应，中间则是保持长连接的请求，最后四条tcp连接则是关闭http连接的四次握手。

![1](https://github.com/nemolpsky/Note/raw/master/file/http/image1/1.png)
![2](https://github.com/nemolpsky/Note/raw/master/file/http//image1/2.png)
![3](https://github.com/nemolpsky/Note/raw/master/file/http//image1/3.png)
---


#### 3. 分析HTTP请求和TCP握手

HTTP请求会进行下面的TCP连接操作来建立连接、关闭连接和传输数据。

![5](https://github.com/nemolpsky/Note/raw/master/file/http//image1/5.png)

- 建立连接的三次握手
  
  1. 第一次握手

      ![6](https://github.com/nemolpsky/Note/raw/master/file/http//image1/6.png)
   
      编号60的就是第一次握手的数据，可以看到有四层数据包，而tcp的数据包就在传输层中
      - Frame(物理层)
      - Ethernet II(数据链路层)
      - Internet Protocol Version 4(互联网层)
      - Transmission Control Protocol(传输层)

      从info中可以看到Seq为0，是从客户端请求到服务端，告诉服务端要建立连接，从下面的详细信息中可以看到```[Next sequence number: 1    (relative sequence number)]```这行信息，这表示下一次从客户端到服务端的tcp连接中Seq为1，因为Seq的计算实际上就是上一次的Seq值加1，这个时候客户端等待服务器确认。

    2. 第二次握手

       ![7](https://github.com/nemolpsky/Note/raw/master/file/http//image1/7.png)    

       编号61是从服务端到客户端的tcp请求，告诉客户端，收到你要建立连接的请求了，从info中可以看到服务端到客户端的请求除了SYN(Synchronize Sequence Numbers)之外还多了一个ACK(Acknowledgement Number)用于确认接受到前面的SYN，服务端的Seq也是从0开始，Ack则是从1开始，也就是第一次握手中服务端传递的Seq基础上加1，这个时候服务器等待客户端的确认。

    3. 第三次握手

        ![8](https://github.com/nemolpsky/Note/raw/master/file/http//image1/8.png)

        编号62是第三次握手，还是从客户端到服务端，基于对第二次握手的确认，可以看到Seq为1，而Ack也为1，上面已经说到了，第一次握手其实可以算出下次Seq的值，就是本次Seq加1，而Ack则是在上次接受到的Seq基础上加1，此时发送给服务器确认信息。

- 传输数据

   上面进行三次握手的操作之后，就能确定客户端和服务器已经建立上连接了，然后真正的HTTP请求就会开始了，可以看到总共有两次HTTP连接，一次是请求，一次是响应，但是实际上HTTP协议是应用层上的协议，其本质作用更像是提供一种更方便供开发者使用的接口，传输数据本质上还是需要使用TCP协议，因此可以看到两次HTTP请求里面都包含着TCP请求的数据。

   可以看到这两次HTTP请求的详细信息里都带着HTTP协议的信息，比如请求头、请求体、请求路径以及最重要的请求参数等等。

   1. HTTP请求

      ![9](https://github.com/nemolpsky/Note/raw/master/file/http//image1/9.png)

      ![10](https://github.com/nemolpsky/Note/raw/master/file/http//image1/10.png)    

      编号63可以看到Seq和Ack都是1，然后Len也就是数据长度是341，因此可以看到```[Next sequence number: 342    (relative sequence number)]```的信息，也就是说下次的Seq的值将会是342，等于当前Seq加上Len的值。

      编号64的TCP请求则是Seq为1，Ack为342，也是一样的计算公式，至此，HTTP的请求就已经结束了，它的作用是确认收到了请求。

   2. HTTP响应

      ![11](https://github.com/nemolpsky/Note/raw/master/file/http//image1/11.png)

      ![12](https://github.com/nemolpsky/Note/raw/master/file/http//image1/12.png)    

      编号66的HTTP响应中可以看到返回了状态码200表示请求成功了，把响应的数据成功取回来了。而在HTTP响应中包含的TCP连接的信息表示Seq为1，Ack为342，数据长度是200，因此下次的Seq为201。

      编号73TCP连接则是客户端告诉服务端已经成功接收到数据了，Seq为342，Ack为201。

      至此整个HTTP的响应也结束了。

- 保持长连接

   ![13](https://github.com/nemolpsky/Note/raw/master/file/http//image1/13.png) 

   在HTTP1.1中请求头会默认带着```Connection: keep-alive```，也就是保持长连接，因为三次握手建立连接和四次握手断开连接是比较耗费性能的，如果请求很频繁的话，客户端会定时请求服务端保证连接不关闭，再有请求进来就直接使用本次连接。


- 关闭连接的四次握手

  ![14](https://github.com/nemolpsky/Note/raw/master/file/http//image1/14.png) 

  相比于建立连接，关闭其实要简单的多，客户端发送一个连接个服务端，编号4930，带着FIN的标识，标识要结束本次HTTP请求，然后服务端回一个带Ack的连接编号4931，表示接受到这次关闭的请求了。

  ![15](https://github.com/nemolpsky/Note/raw/master/file/http//image1/15.png) 

  相同的，服务端也做同样的操作，发一个带FIN标志的连接，编号4932给客户端，客户端也回一个带Ack的连接，编号4933，表示接收到了，至此双方都收到到对方关闭连接的请求，也都回复了，整个HTTP连接也就断开了。

---


### 4. SYN和ACK

在TCP连接中SYN和ACK是两个非常重要的东西，它们是用来解决乱序和确认的问题，上面的Seq和Ack就是指这两个东西。

- Sequence Number是包的序号，用来解决网络包乱序(reordering)问题。
- Acknowledgement Number就是ACK，用于确认收到，用来解决不丢包的问题。

上面讲解每次TCP连接的时候会发现其实SYN和ACK两个值都可以提前算好的，计算的规则也很简单，服务端和客户端的SYN都是从0开始，建立连接的三次握手和关闭连接的四次握手中，SYN的值都是上次本端连接SYN值加1，而ACK则是另一端TCP连接的SYN值加1。

只有在传输数据的时候比较特殊，SYN是上次本端连接SYN值加数据长度(Len字段)，ACK则直接是另一端TCP连接的SYN值。


```
Client = C, Server = S
// 三次握手建立连接
C ---> S，SYN=0
S ---> C，SYN=0，ACK=1
C ---> S，SYN=1，ACK=1

// 传输数据
(HTTP)C ---> S，SYN=1，ACK=1，Len=341
S ---> C，SYN=1，ACK=342
(HTTP)S ---> C，SYN=1，ACK=342，Len=200
C ---> S，SYN=342，ACK=201

// 四次握手关闭连接
S ---> C，SYN=201，ACK=342
C ---> S，SYN=342，ACK=202
C ---> S，SYN=342，ACK=202
S ---> C，SYN=202，ACK=343
```

