---
title: RocketMQ源码分析——NameServer启动流程
date: 2020-07-010 14:15:37
tags: RocketMQ
categories: 消息队列
description: 介绍NameServer启动流程
---

## 前言

上一篇谈到 RocketMQ 的架构，可以看到 NameServer 作为 RocketMQ 的服务注册中心，提供了路由管理、服务注册以及服务发现功能，可以说它就是 RocketMQ 的“大脑”，就像 Dubbo 中的 Zookeeper 一样。

![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/RocketMQ/1.rocketmq%E7%89%A9%E7%90%86%E9%83%A8%E7%BD%B2%E5%9B%BE.png)

《RocketMQ技术内幕》中提到：

Broker 消息服务器在启动时向所有 NameServer 注册，消息生产者（Producer）在发送消息之前先从 NameServer 获取 Broker 服务器地址列表，然后根据负载算法从列表中选择一台消息服务器进行消息发送。NameServer 与每台 Broker 服务器保持长连接，并间隔 30s 检测 Broker 是否存活，如果检测到 Broker 宕机， 则从路由注册表中将其移除。但是<u>路由变化不会马上通知消息生产者</u>，为什么要这样设计呢？这是为了降低 NameServer 实现的复杂性，在消息发送端提供容错机制来保证消息发送的高可用性。

NameServer 本身的高可用可通过部署多台 NameServer 服务器来实现，但**彼此之间互不通信，也就是 NameServer 服务器之间在某一时刻的数据并不会完全相同**，但这对消息发送不会造成任何影响，这也是 RocketMQ NameServer 设计的一个亮点， RocketMQ NameServer 设计追求简单高效。

## NameServer启动流程

我这边的版本是`4.5.2`。

主要看启动类：`org\apache\rocketmq\namesrv\NamesrvStartup.java`

### 创建NameServer实例

`main`方法进去会调用`main0`方法，看下`main0`方法：

```java
public static NamesrvController main0(String[] args) {

    try {
        NamesrvController controller = createNamesrvController(args);
        start(controller);
        String tip = "The Name Server boot success. serializeType=" + RemotingCommand.getSerializeTypeConfigInThisServer();
        log.info(tip);
        System.out.printf("%s%n", tip);
        return controller;
    } catch (Throwable e) {
        e.printStackTrace();
        System.exit(-1);
    }

    return null;
}
```

看4、5两行。第4行创建了一个`NamesrvController`对象，虽然这个类里面没有任何说明信息。。。（RocketMQ的源码中相关类的说明信息很少）但是从它的属性中可以得知，这是一个`NameServer`的实现类，里面有服务启动配置、通信模块配置、键值对的存储等等。

进入`createNamesrvController`方法，截取部分代码：

```java
// 解析配置文件
final NamesrvConfig namesrvConfig = new NamesrvConfig();
final NettyServerConfig nettyServerConfig = new NettyServerConfig();
// 设置namesrv的服务端口
nettyServerConfig.setListenPort(9876);
// c指定启动的时候加载配置文件
if (commandLine.hasOption('c')) {
    // 命令行启动指定配置文件，前面用c开头
    String file = commandLine.getOptionValue('c');
    if (file != null) {
        InputStream in = new BufferedInputStream(new FileInputStream(file));
        properties = new Properties();
        properties.load(in);
        MixAll.properties2Object(properties, namesrvConfig);
        MixAll.properties2Object(properties, nettyServerConfig);

        // 设置命令行启动namesrv指定的配置文件路径
        namesrvConfig.setConfigStorePath(file);

        System.out.printf("load config properties file OK, %s%n", file);
        in.close();
    }
}

// 打印namesrv的配置信息，命令行上面加p
if (commandLine.hasOption('p')) {
    InternalLogger console = InternalLoggerFactory.getLogger(LoggerName.NAMESRV_CONSOLE_NAME);
    MixAll.printObjectProperties(console, namesrvConfig);
    MixAll.printObjectProperties(console, nettyServerConfig);
    System.exit(0);
}

// 把命令行属性解析成properties
MixAll.properties2Object(ServerUtil.commandLine2Properties(commandLine), namesrvConfig);
```

这里创建了`NamesrvConfig`(NameServer业务参数) 和 `NettyServerConfig`(NameServer网络参数)，然后将配置文件和启动命令中的选项值，填充到各自的对象中。

来看下`NamesrvConfig`：

```java
// rocketmq主目录，可以通过-Drocketmq.home.dir=path或通过设置环境变量ROCKETMQ_HOME来配置RocketMQ的主目录。
private String rocketmqHome = System.getProperty(MixAll.ROCKETMQ_HOME_PROPERTY, System.getenv(MixAll.ROCKETMQ_HOME_ENV));
// NameServer存储KV配置属性的持久化路径
private String kvConfigPath = System.getProperty("user.home") + File.separator + "namesrv" + File.separator + "kvConfig.json";
// nameServer默认配置文件路径，不生效。nameServer启动时如果要通过配置文件配置NameServer启动属性的话，请使用-c选项
private String configStorePath = System.getProperty("user.home") + File.separator + "namesrv" + File.separator + "namesrv.properties";
private String productEnvName = "center";
private boolean clusterTest = false;
// 是否支持顺序消息，默认不支持
private boolean orderMessageEnable = false;
```

`NettyServerConfig`：

```java
// NameServer监昕端口，该值默认会被初始化为9876
private int listenPort = 8888;
// Netty业务线程池线程个数
private int serverWorkerThreads = 8;
// Netty public 任务线程池线程个数，Netty网络设计，根据业务类型会创建不同的线程池，比如处理消息发送、消息消费、心跳检测等。
//如果该业务类型（RequestCode）未注册线程池，则由public线程池执行
private int serverCallbackExecutorThreads = 0;
// IO线程池线程个数，主要是NameServer、Broker端解析请求、返回相应的线程个数，
// 这类线程主要是处理网络请求的，解析请求包，然后转发到各个业务线程池完成具体的业务操作，然后将结果再返回调用方。
private int serverSelectorThreads = 3;
// send oneway 消息请求并发度（Broker端参数）
private int serverOnewaySemaphoreValue = 256;
// 异步消息发送最大并发度（Broker端参数）
private int serverAsyncSemaphoreValue = 64;
// 网络连接最大空闲时间，默认120s。如果连接空闲时间超过该参数设置的值，连接将被关闭
private int serverChannelMaxIdleTimeSeconds = 120;

// 网络socket发送缓存区大小，默认64k
private int serverSocketSndBufSize = NettySystemConfig.socketSndbufSize;
// 网络socket接收缓存区大小，默认64k
private int serverSocketRcvBufSize = NettySystemConfig.socketRcvbufSize;
// ByteBuffer是否开启缓存，建议开启
private boolean serverPooledByteBufAllocatorEnable = true;
```

### start方法

`NamesrvController`实例创建完成后，进入下一行`start(controller)`，启动`NamesrvController`实例。该方法内部调用了`NamesrvController#initialize()`方法进行实例的初始化：

```java
public static NamesrvController start(final NamesrvController controller) throws Exception {
    
    if (null == controller) {
        throw new IllegalArgumentException("NamesrvController is null");
    }
    // NamesrvController初始化
    boolean initResult = controller.initialize();
    if (!initResult) {
        controller.shutdown();
        System.exit(-3);
    }

    // 注册JVM钩子函数并启动服务器，以便监听Broker、消息生产者的网络请求
    Runtime.getRuntime().addShutdownHook(new ShutdownHookThread(log, new Callable<Void>() {
        @Override
        public Void call() throws Exception {
            controller.shutdown();
            return null;
        }
    }));
    controller.start();
    return controller;
}
```

#### 初始化NamesrvController

进入`initialize()`方法：

```java
// 加载KV配置
this.kvConfigManager.load();

// 创建NameServer网络处理对象
this.remotingServer = new NettyRemotingServer(this.nettyServerConfig, this.brokerHousekeepingService);

this.remotingExecutor =
    Executors.newFixedThreadPool(nettyServerConfig.getServerWorkerThreads(), new ThreadFactoryImpl("RemotingExecutorThread_"));

this.registerProcessor();

// 下面两端开启两个定时任务，即NameServer的心跳检测
// 定时任务1: NameServer每隔1Os扫描一次Broker，移除处于不激活状态的Broker
this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

    @Override
    public void run() {
        NamesrvController.this.routeInfoManager.scanNotActiveBroker();
    }
}, 5, 10, TimeUnit.SECONDS);

// 定时任务2: nameserver每隔10分钟打印一次KV配置
this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

    @Override
    public void run() {
        NamesrvController.this.kvConfigManager.printAllPeriodically();
    }
}, 1, 10, TimeUnit.MINUTES);
```

可以看到这个实例化方法里面加载了之前的一些配置，并使用`ScheduledExecutorService`启动了心跳检测，`ScheduledExecutorService`的对象`scheduledExecutorService`实际上是具有定时任务、只包含一个线程的线程池。

#### 注册钩子函数

这一方法执行完成后回到`start`方法，第14行注册了一个`Runtime`钩子函数用于监听 Broker、消息生产者的网络请求。这里可以学习到优雅关闭线程池的方法：

```java
Runtime.getRuntime().addShutdownHook(new ShutdownHookThread(log, new Callable<Void>() {
    @Override
    public Void call() throws Exception {
        controller.shutdown();
        return null;
    }
}));
```

这段函数的意思就是在 JVM 进程关闭前，执行`addShutdownHook`方法体中的线程，优雅地关闭线程池。

#### 启动NameServer

最后，启动 Namesrv：

```java
controller.start();
```

这个`start()`方法最终会调用`NettyRemotingServer`的`start`方法。

至此，NameServer 也就启动完成了。

## 总结

简单总结一下 NameServer 启动流程：

1. 在执行 main 方法时，首先会创建一个 NameServer 实例：`NameSrvController`，并注册相关配置信息（NameServer 业务参数和 NameServer 网络参数）。
2. 之后进行 NameServer 的初始化，加载配置信息并启动心跳检测用于检测 Broker 的状态。
3. 最后启动 NameServer，底层通过 Netty 实现。