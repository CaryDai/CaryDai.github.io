---
title: 实现自己的RPC框架（四）
date: 2020-02-13 14:18:02
tags: RPC
categories: 分布式
description: 使用Netty替代传统的BIO，增强系统的并发量和性能。
---

## 思路

### 从BIO到NIO

BIO模型（图来自w3cschool）：

![](https://atts.w3cschool.cn/attachments/image/20170808/1502158706351902.jpg)

之前的版本都是使用传统的 BIO（Blocking I/O）进行客户端与服务端的交互。然而这种方式中，服务端等待客户端发数据的这个过程是阻塞的——即服务端每次都需要有一个线程去监听客户端的连接请求（通常称作 “Acceptor 线程”），收到客户端的请求后，又要去建立一个新的线程来处理这个请求：

```java
// 服务端监听请求
while (true) {
    Socket socket = listener.accept();	// 阻塞
    new Thread(() -> {
        ...
        read(data);	// 阻塞
        ...
    })
}
```

这个线程会调用`read()`去读取客户端的数据，那么在读取到数据前，这个线程会被阻塞，在此期间它只能等着，直到其读到数据。

虽然上个版本中，在服务端引入了线程池来管理这些线程，但是线程池只是保证资源占用是可控的——无论有多少个客户端并发访问，都不会导致资源的耗尽和宕机。然而其本质上还是同步阻塞的 BIO 方式，在高并发的情况下，多余的请求只能一直等待。所以考虑使用 NIO 来解决线程阻塞的问题。

NIO是一种同步非阻塞的I/O模型，它支持面向缓冲的，基于通道的I/O操作方法。NIO 使用 Selector 解决了线程的阻塞问题。服务端监测到新的连接之后，不再创建一个新的线程，而是将这个请求交给Selector，Selector会不断地去遍历所有的Socket，一旦有一个Socket建立完成，他会通知Thread，然后Thread处理完数据再返回给客户端 Socket ——**这个过程是不阻塞的**，这样就能让一个Thread处理更多的请求了。（比如我这个线程刚处理完 Socket2 的连接请求，现在 Selector 通知我可以去处理 Socket1 的读数据请求了。这样，一个线程可以处理多个客户端的请求，节省了线程资源开销。）

![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/RPC/NIO%E6%A8%A1%E5%9E%8B.webp)

### 从NIO到Netty

Java 原生的 NIO 代码比较复杂，而 Netty 是一款基于 NIO 开发的网络通信框架，使用它可以高效地使用 NIO 的特性，并且可以减轻编码压力，所以我选择直接使用 Netty 来替代原先的 BIO 通信方式。

Netty 的内容其实并不少，在W3Cschool上可以找到[《Netty 实战精髓》](https://www.w3cschool.cn/essential_netty_in_action/essential_netty_in_action-ofey289e.html)的电子书。它里面有一个例子：实现一个 echo 服务器，看懂了这个例子，再在自己的 RPC 中加入 Netty 就相对容易了。

首先需要理清整个过程的思路，先画个图看下：

![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/RPC/v4%E6%A1%86%E6%9E%B6.png)

项目结构：

![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/RPC/v4%E9%A1%B9%E7%9B%AE%E7%BB%93%E6%9E%84.png)

## Netty服务端

服务端主要由两个文件构成：`NettyServer.java`和`ServerHandler.java`。前者主要用来与客户端建立连接并传输数据，后者用于处理数据。下面来看下这两个文件的主要内容：

**NettyServer.java**

```java
public void start() throws Exception {
    ...    // 将Server服务注册到zookeeper
    
    // 用于处理客户端的连接请求
    NioEventLoopGroup bossGroup = new NioEventLoopGroup();
    // 用于处理与各个客户端连接的IO操作
    NioEventLoopGroup workerGroup = new NioEventLoopGroup();
    try {
        ServerBootstrap b = new ServerBootstrap();
        b.group(bossGroup, workerGroup)   // 设置 EventLoopGroup 用于处理所有的 Channel 的事件（Netty 限制每个Channel 都由一个 Thread 处理）
            .channel(NioServerSocketChannel.class)        // 使用指定的 NIO 传输 Channel
            .childHandler(new ChannelInitializer<SocketChannel>() { // 添加 ServerHandler 到 Channel 的 ChannelPipeline
                @Override
                public void initChannel(SocketChannel ch) throws Exception {
                    // 解码。设置对象序列化最大长度为1M
                    // 指定加载序列化对象的类的类解析器为weakCachingConcurrentResolver
                    ch.pipeline().addLast(new ObjectDecoder(1024*1024,
                                                            ClassResolvers.weakCachingConcurrentResolver(this.getClass().getClassLoader())));
                    // 添加对象编码器
                    ch.pipeline().addLast(new ObjectEncoder());
                    ch.pipeline().addLast(new ServerHandler());
                }
            })
            .option(ChannelOption.SO_BACKLOG, 128)
            .childOption(ChannelOption.SO_KEEPALIVE, false);

        ChannelFuture f = b.bind(port).sync();       // 绑定端口,开始接收进来的连接
        f.channel().closeFuture().sync();            // 等到服务端监听端口关闭
    } finally {
        bossGroup.shutdownGracefully().sync();            // 优雅地释放线程资源
    }
}
```

代码框架参考了“echo服务器”的例子。整体思路是：首先创建一个 Bootstrap 对象，它的作用是用于将各个组件组装起来。

- `ServerBootstrap`其实是服务端的引导类，相应地，客户端的引导类是`Bootstrap`，Netty 正是通过这两个引导类来启动。

- `EventLoopGroup`可以装载一个或多个 EventLoop，而 EventLoop 本身只由一个线程驱动，它保证一个 Thread 对应一个 Channel。这也就意味着这个 Channel 里的所有 ChannelHandler 都由一个线程执行，这也体现了“非阻塞”特性。
  **为什么这里要有两个`NioEventLoopGroup`呢？** 《Netty实战》里面有提到：

  Netty 应用程序通过设置 bootstrap（引导）类开始，该类提供了一个 用于应用程序网络层配置的容器。Bootstrapping 有以下两种类型：

  - 一种是用于客户端的Bootstrap：连接到远程主机和端口，有1个EventLoopGroup
  - 一种是用于服务端的ServerBootstrap：绑定本地端口，有2个EventLoopGroup

  一个 ServerBootstrap 可以认为有2个 Channel 集合，第一个集合包含一个单例 ServerChannel，<u>代表持有一个绑定了本地端口的 socket</u>；<u>第二集合包含所有创建的 Channel，处理服务器所接收到的客户端进来的连接</u>。下图形象的描述了这种情况：
  ![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/RPC/Netty_EventLoopGroup.jpg)

- `channel()`函数指定特定的 Channel。

- `childHandler()`设置添加到 ChannelPipeline 中的 ChannelHandler。

- `option()`和`childOption()`分别用来设置 boss线程组和 worker线程组的TCP连接可选项。详细参数说明可以看这篇文章：[Netty：option和childOption参数设置说明](https://www.jianshu.com/p/0bff7c020af2)。这里我都按其默认值设置。

- `bind()`将通道绑定并返回一个 ChannelFuture，用于在绑定操作完成后通知。

这里序列化使用了 JDK 原生的序列化方式，Netty也提供了其它的序列化方式，比如 Protobuf。

**ServerHandler.java**

```java
public class ServerHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        // 获取客户端发来的数据信息
        RequestObject requestObject = (RequestObject) msg;
        System.out.println("Server接受到客户端的信息 :" + requestObject.toString());

        String methodName = requestObject.getMethodName();
        String className = requestObject.getClassName();
        Class<?>[] parameterTypes = requestObject.getParameterTypes();
        Object[] params = requestObject.getParams();

        // 根据上述收到的信息，调用相应类中的方法并返回结果。
        Class serverClass = ServerServices.remoteServices.get(className);
        Method method = serverClass.getMethod(methodName, parameterTypes);
        Object result = method.invoke(serverClass.newInstance(), params);

        System.out.println("服务端处理完毕，发送结果给客户端：" + result);
        ctx.writeAndFlush(result);  // 将运行结果返回给客户端。
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        cause.printStackTrace();                //5
        ctx.close();                            //6
    }
}
```

ServerHandler 根据收到的方法信息，调用本地方法，最后将结果发送回客户端。

## Netty客户端

客户端的代码和服务端类似，主要由`NettyClient.java`和`ClientHandler.java`构成。这里只放一下 ClientHandler 的代码：

```java
public class ClientHandler extends ChannelInboundHandlerAdapter{

    private RequestObject requestObject;

    public ClientHandler(RequestObject requestObject) {
        this.requestObject = requestObject;
    }

    // 客户端连上服务端的时候触发
    @Override
    public void channelActive(ChannelHandlerContext ctx) {
        System.out.println("客户端准备发消息给服务器...");
        ctx.writeAndFlush(requestObject);
    }

    // 接收服务端的数据
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        try {
            Object res = msg;
            System.out.println("Client收到消息！");
            System.out.println(res);
        } finally {
            ReferenceCountUtil.release(msg);
            ctx.close();
        }
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        cause.printStackTrace();
        ctx.close();
    }
}
```

最终，启动 ZooKeeper，启动服务器，启动客户端，一样可以得到正常结果：

![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/RPC/v4Server%E5%8F%91%E9%80%81%E5%B9%B6%E6%8E%A5%E6%94%B6%E6%B6%88%E6%81%AF.png)

![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/RPC/v4Client%E5%8F%91%E9%80%81%E5%B9%B6%E6%8E%A5%E6%94%B6%E6%B6%88%E6%81%AF.png)

## 注意点

1. 一开始运行成功后，发现客户端没有关闭。后来发现是`f.channel().closeFuture().sync();`这一句的问题。这句话是阻塞的，也就是说当客户端不再与服务器连接时才会关闭，除非链路断了，否则不会终止，所有的 Channel 仍是连接状态。解决方法是在`ClientHandler`中手动关闭：`ctx.close();`
2. 因为最终是在`NettyClient`中把客户端要调用的方法信息序列化后发送给服务端的，所以在自定义的代理类的事件处理器`MyInvocationHandler`中需要做两件事：1）从 ZooKeeper 注册中心得到服务的ip地址与端口；2）将服务的ip地址、端口号和调用的方法信息传递给`NettyClient`。

代码传到 [github](https://github.com/CaryDai/RPC_V4) 上了，欢迎 star。