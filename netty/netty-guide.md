## Netty入门

在前文中，我们介绍了Netty的基本功能、特点，以及netty源码的结构，本篇我们通过netty官方文档中的几个用例，介绍如何通过netty编写简单的网络程序，以及netty网络应用的基本结构和核心类。

>本篇的用例都取自netty官方文档，感兴趣的可以访问[Netty Guide for 4.x](https://netty.io/wiki/user-guide-for-4.x.html)查看

### DiscardServer

首先我们实现一个DiscardServer，它将忽略一切收到的数据，并且不会返回任何数据。

我们首先实现DiscardServerHandler来处理输入的数据，Handler源码如下：

DiscardServerHandler.java

```java
public class DiscardServerHandler extends ChannelInboundHandlerAdapter { // (1)

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) { // (2)
        // Discard the received data silently.
        ((ByteBuf) msg).release(); // (3)
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) { // (4)
        // Close the connection when an exception is raised.
        cause.printStackTrace();
        ctx.close();
    }
}
```

1.  DiscardServerHandler继承了ChannelInboundHandlerAdapter，ChannelInboundHandlerAdapter是ChannelInboundHandler的适配器类，ChannelInboundHandler提供了多种事件处理方法供开发者使用。

2. DiscardServerHandler重写了channelRead方法，当有新数据到来时，netty会调用该方法进行处理，同时会传入收到的数据msg。

3. msg是一个ByteBuf类型的数据，ByteBuf是netty实现的一个基于引用计数的缓冲类，开发者必须显示的调用release方法释放该对象。在DiscardChannelHandler中，我们将收到的数据直接释放即可。

4. 我们重写了exceptionCaught方法，当出现IO异常，或者ChannelHandler的处理出现异常时，netty会调用该接口，同时传入具体的exception。

> ChannelInboundHandlerAdapter实现了java设计模式中经典的适配器模式

目前我们已经清楚了如何使用ChannelHandler处理收到的数据，接下来我们看一下如何编写启动类以启动DiscardServer。

DiscardServer.java

```java
public class DiscardServer {
    
    private int port;
    
    public DiscardServer(int port) {
        this.port = port;
    }
    
    public void run() throws Exception {
        EventLoopGroup bossGroup = new NioEventLoopGroup(); // (1)
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap b = new ServerBootstrap(); // (2)
            b.group(bossGroup, workerGroup)
             .channel(NioServerSocketChannel.class) // (3)
             .childHandler(new ChannelInitializer<SocketChannel>() { // (4)
                 @Override
                 public void initChannel(SocketChannel ch) throws Exception {
                     ch.pipeline().addLast(new DiscardServerHandler());
                 }
             })
             .option(ChannelOption.SO_BACKLOG, 128)          // (5)
             .childOption(ChannelOption.SO_KEEPALIVE, true); // (6)
    
            // Bind and start to accept incoming connections.
            ChannelFuture f = b.bind(port).sync(); // (7)
    
            // Wait until the server socket is closed.
            // In this example, this does not happen, but you can do that to gracefully
            // shut down your server.
            f.channel().closeFuture().sync();
        } finally {
            workerGroup.shutdownGracefully();
            bossGroup.shutdownGracefully();
        }
    }
    
    public static void main(String[] args) throws Exception {
        int port = 8080;
        if (args.length > 0) {
            port = Integer.parseInt(args[0]);
        }

        new DiscardServer(port).run();
    }
}
```

1. NioEventLoopGroup是处理网络请求的多线程处理器，Netty提供了EventLoopGroup的多种实现，在这里我们使用了两个EventLoopGroup，bossGroup负责处理新到的连接，bossGroup接收新的连接后，将会把连接传递给workerGroup进行具体的IO操作。

2. ServerBootstrap帮助我们很方便的配置并启动一个服务端程序，开发者也可以手动配置Channel启动，不过过程比较复杂，大多数情况下使用ServerBootstrap即可。

3. 当有新的连接到来时，我们使用NioServerSocketChannel对象绑定新连接

4. 每当有新的Channel时，netty都会使用childHandler中的对象处理新Channel。ChannelInitializer是特殊的ChannelHandler，用于配置新的Channel，大多数情况下，我们都会在这里加入一些pipeline中的ChannelHandler，在本例中是DiscardServerHandler。

5. 我们的服务基于TCP连接，option方法可以配置netty服务NioServerSocketChannel(该channel用于accept新的连接)，我们可以在ChannelOption中查看可配置的选项。

6. childOption用于配置paretChannel(NioServerSocketChannel)接收新连接创建的channel。

7. 在这一步我们绑定指定端口并启动服务

到此为止，我们已经完成了DiscardServer的所有程序，接下来要为我们的服务添加一些新功能。

### EchoServer

通过上面的例子，我们已经熟悉了netty处理网络请求的具体流程。接下来，我们会在服务中添加功能，将服务端收到的请求原样写回到客户端，以此来学习如何在Channel中读取/写入数据。

我们首先修改DiscardChannelHandler的实现：

```java
@Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        ctx.write(msg); // (1)
        ctx.flush(); // (2)
    }
```

1. ChannelHandlerContext提供多种接口供开发者处理多种Channel上的IO操作。在这里我们使用write方法将数据写入channel，注意这里我们没有调用msg的release方法，因为当write结束时netty会为我们释放

2. write方法仅仅将数据写入到内部缓冲区中，调用flush将会将数据flush到具体的网络连接中

现在，如果你使用telnet连接该服务，服务器每次都将会输入的数据原样返回。

### TimeServer

本部分我们将实现一个TimeServer，每次有客户端连接到服务端时，不需要客户端发送数据，服务端就会返回当前时间给客户端。

我们实现的TimeChannelHandler如下：

TimeChannelHandler.java

```java
public class TimeServerHandler extends ChannelInboundHandlerAdapter {

    @Override
    public void channelActive(final ChannelHandlerContext ctx) { // (1)
        final ByteBuf time = ctx.alloc().buffer(4); // (2)
        time.writeInt((int) (System.currentTimeMillis() / 1000L + 2208988800L));
        
        final ChannelFuture f = ctx.writeAndFlush(time); // (3)
        f.addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) {
                assert f == future;
                ctx.close();
            }
        }); // (4)
    }
    
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        cause.printStackTrace();
        ctx.close();
    }
}
```

1. 当channel对应连接建立成功，并准备进行IO操作时，netty会调用channelActive方法

2. 我们要写入一个四字节的int值，因此这里申请一个4字节大小的buffer

3. 我们将buffer中的数据写入channel中，注意此处，netty的缓冲区记录了read和write两个指针，因此我们不需要像java的ByteBuffer那样调用flip方法重置缓冲区的位置

4.  netty是一个异步框架，因此第三步返回了ChannelFuture，本步骤注册listener，当writeAndFlush操作结束后，关闭该Channel。

### TimeClient

当我们使用telnet连接TimeServer时，会返回乱码，这是因为telnet客户单还无法解析一个四字节的int值。因此我们需要实现TimeClient来解析接收到的数据。

我们首先编写TimeClient的启动类：

TimeClient.java

```java
public class TimeClient {
    public static void main(String[] args) throws Exception {
        String host = args[0];
        int port = Integer.parseInt(args[1]);
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        
        try {
            Bootstrap b = new Bootstrap(); // (1)
            b.group(workerGroup); // (2)
            b.channel(NioSocketChannel.class); // (3)
            b.option(ChannelOption.SO_KEEPALIVE, true); // (4)
            b.handler(new ChannelInitializer<SocketChannel>() {
                @Override
                public void initChannel(SocketChannel ch) throws Exception {
                    ch.pipeline().addLast(new TimeClientHandler());
                }
            });
            
            // Start the client.
            ChannelFuture f = b.connect(host, port).sync(); // (5)

            // Wait until the connection is closed.
            f.channel().closeFuture().sync();
        } finally {
            workerGroup.shutdownGracefully();
        }
    }
}

```

1. 客户端程序中，我们使用Bootstrap配置启动类

2. 如果我们只指定一个EventLoop，它将会同时扮演boss和worker的角色。并且在客户端程序中，并不会用到boss处理器

3. 客户端程序中，我们使用NioSocketChannel绑定连接

4. 客户端程序我们不需要childOption，因为客户端的SocketChannel没有parent channel。

5. 使用connect方法连接服务器

接下来，我们使用TimeClientHandler处理接收到的数据。

TimeClientHandler.java

```java
public class TimeClientHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        ByteBuf m = (ByteBuf) msg; // (1)
        try {
            long currentTimeMillis = (m.readUnsignedInt() - 2208988800L) * 1000L;
            System.out.println(new Date(currentTimeMillis));
            ctx.close();
        } finally {
            m.release();
        }
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        cause.printStackTrace();
        ctx.close();
    }
}

```

1. 同DiscardChannelHandler类似，我们接收ByteBuf对象，并进行解析

TimeClient的程序看起来非常简答 ，不过我们会发现，程序偶尔会抛出IndexOutOfBoundsException异常。这是因为在类似TCP/IP这样的基于流的协议中，客户端接收到的数据会存储在socket缓冲区中，缓冲区中存储的并不是java对象的队列，而是字节队列。假设客户端写了两条信息到客户单，客户端无法看到两条消息而只能看到一系列的字节，这样的话，客户端就无法将数据解析成跟原始数据完全一样的格式了，此时就需要用到netty的Decoder了。

### Decoder

Decoder本质上是一种实现特殊功能的ChannelHandler，它将接收到的字节数组转化成具体的对象，供后续的ChannelHandler处理，TimeDecoder的具体实现如下：

TimeDecoder.java

```java
public class TimeDecoder extends ByteToMessageDecoder { // (1)
    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) { // (2)
        if (in.readableBytes() < 4) {
            return; // (3)
        }
        
        out.add(in.readBytes(4)); // (4)
    }
}
```

1. ByteToMessageDecoder也是ChannelInboundHandler的一种实现，使我们可以更好的处理字节分片。

2. Decoder将接收到的数据写入到ByteBuf中，并在每次有新数据时调用decode方法。

3. 如果接收到的数据不足4字节（一个int的长度），将继续等待数据。

4. 如果数据完整，将会加入到out列表中，后续的handler将会从out中读取到完整的数据，out中除了可以放置ByteBuf外，还可以放置自定义的一些POJO对象

最后，我们只需要将TimeDecoder加入到ChannelPipeline即可

```java
b.handler(new ChannelInitializer<SocketChannel>() {
    @Override
    public void initChannel(SocketChannel ch) throws Exception {
        ch.pipeline().addLast(new TimeDecoder(), new TimeClientHandler());
    }
});
```


### Shutting Down

当我们要终止netty程序时，只需要调用shutdownGracefully方法将所有的EventLoopGroup终止即可。shutdownGracefully会返回一个Future对象，通知开发者EventLoopGroup及其Channel已经被关闭。
