---
title: 实现自己的RPC框架（三）
date: 2020-01-10 12:52:21
tags: RPC
categories: 分布式
description: 第三版RPC，在第二版基础上加了在服务端用线程池来处理多个客户端请求。此外，以ZooKeeper作为服务注册中心。
---

## 增加线程池

首先在服务端加入线程池以更好地管理线程资源。为了代码逻辑更加清晰，我先把Server端的服务缓存模块单独抽离出来：

`ServerServices`的代码：

```java
/**
 * @Author dqj
 * @Date 2020/1/10
 * @Version 1.0
 * @Description Server端的缓存。
 */
public class ServerServices {
    // 用来存放Server端对象的缓存
    public static HashMap<String, Class> remoteServices = new HashMap<>();

    // 把一个Server端对象放到缓存中
    private void register(Class className, Class remoteImpl) {
        remoteServices.put(className.getName(), remoteImpl);
    }

    // 将Server端的的服务都缓存进来
    public void registerServices() {
        this.register(HelloService.class, HelloServiceImpl.class);
    }
}

```

接着在服务端的启动类中写线程池的代码，使用`ThreadPoolExecutor`类，调用它的构造函数来构造出一个线程池。接着在与客户端建立连接后，不断监听客户端的请求，若收到了客户端的请求，则启动一个线程来处理该请求。

`RPCServer`的代码：

```java
public class RPCServer {
    public static void main(String[] args) throws Throwable {
        // 创建一个核心线程数为5，最大线程数为10，阻塞队列大小为4的线程池。
        ThreadPoolExecutor executor = new ThreadPoolExecutor(5,10,200, TimeUnit.MILLISECONDS,
                new ArrayBlockingQueue<Runnable>(4));
        // 注册服务
        ServerServices services = new ServerServices();
        services.registerServices();
        // 建立连接
        ServerSocket listener = new ServerSocket(8090);
        System.out.println("Server等待客户端的连接...");
        // 监听请求
        while (true) {
            Socket socket = listener.accept();
            executor.execute(new ServerStub(socket));
        }
    }
}
```

`ServerStub`稍作修改，写一个构造函数用来接收一个 socket 请求：

```java
public class ServerStub implements Runnable {
    private Socket socket;
    
    public ServerStub(Socket socket) {
        this.socket = socket;
    }
    
    ...
}
```

然后在客户端多写一个`RPCClient2`，并调用服务端的`HelloService`的`add`方法。启动服务端，相继启动客户端进行方法调用，输出结果如下：

![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/RPC/v3%E6%9C%8D%E5%8A%A1%E7%AB%AF%E8%BE%93%E5%87%BA%E7%BB%93%E6%9E%9C.png)

## 加入ZooKeeper

ZooKeeper 可以用于服务的注册与发现。关于服务的注册与发现，主要是把服务名以及服务相关的服务器 IP 地址注册到注册中心；在使用服务的时候，只要根据服务名，就可以得到所有服务地址 IP，然后根据一定的负载均衡策略来选择一台服务器进行处理。

基于 ZooKeeper 的服务注册与发现架构如下图所示：

![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/RPC/v3%E5%9F%BA%E4%BA%8EZookeeper%E7%9A%84%E6%9C%8D%E5%8A%A1%E6%B3%A8%E5%86%8C%E4%B8%8E%E5%8F%91%E7%8E%B0.png)

在实际编码的过程中，需要在服务提供方和服务消费方各创建一个 ZooKeeper 的客户端对象，用于和 ZooKeeper 服务器进行交互。

首先需要将 ZooKeeper 的包引入进来，考虑以 Maven 的方式引入包，所以需要先将原项目转化为 Maven 项目。在项目名称上鼠标右击----选择`Add Framework Support`，再在弹出的窗口中，左边勾选`maven`，点击 OK，IDEA 会自动将项目转化为 maven 项目。在`pom.xml`中输入以下代码即可引入：

```xml
<dependencies>
    <dependency>
        <groupId>org.apache.zookeeper</groupId>
        <artifactId>zookeeper</artifactId>
        <version>3.4.8</version>
    </dependency>
</dependencies>
```

ZooKeeper 的 api 文档在这里：http://zookeeper.apache.org/doc/r3.4.8/api/index.html

实际使用过程中，有几个比较重要的函数：

- 构造函数——用于创建一个 ZooKeeper 客户端对象

  ```java
  ZooKeeper(String connectString, int sessionTimeout, Watcher watcher)
  ```

  - `connectString`是`host:port`形式，表示 ZooKeeper 服务器的 IP 地址和端口号，默认的用于给 client 提供服务的端口是`2181`；
  - `sessionTimeout`表示会话的超时时间；
  - `Watcher`是服务消费方用于监听服务变动的，它将会处理对应的触发事件。

- `create`函数——用于根据给定路径创建一个`Znode`节点

  ```java
  public String create(String path, byte[] data, List<ACL> acl, CreateMode createMode) throws KeeperException, InterruptedException
  ```

  - `path`表示生成的节点的路径；
  - `data`表示节点的初始值；
  - `acl`表示节点的访问权限，对于ZooKeeper客户端来说，有3种标准的 ACL：
    - OPEN_ACL_UNSAFE：使所有ACL都“开放”了：任何应用程序在节点上可进行任何操作，能创建、列出和删除它的子节点。
    - READ_ACL_UNSAFE：对任何应用程序都是只读的。
    - CREATOR_ALL_ACL：赋予了节点的创建者所有的权限，在创建者采用此ACL创建节点之前，已经被服务器所认证（例如，采用 “ *digest*”方案）。
  - `createMode`表示指定要创建的节点是临时的还是持久的，有4个取值：`EPHEMERAL`、`EPHEMERAL_SEQUENTIAL`、`PERSISTENT`、`PERSISTENT_SEQUENTIAL`。

  返回节点的路径。

### 配置 ZooKeeper

还有一个准备工作是在虚拟机中启动 ZooKeeper 服务，具体是在 ZooKeeper 的安装路径下执行`sudo bin/zkServer.sh start`用于启动 ZooKeeper 服务；接着可以执行`sudo bin/zkCli.sh`，这是 ZooKeeper 提供的一个简单的命令行客户端，我们可以通过这个客户端来查看 ZooKeeper 的一些信息。

如果忘了 zookeeper 的安装位置，可以使用`whereis`命令查看，比如我的位置：

```bash
dqj@ubuntu:/$ whereis zookeeper
zookeeper: /usr/local/zookeeper
```

如果发现 zookeeper 启动了，但是查看状态时却出错：

```bash
root@ubuntu:/usr/local/zookeeper/zookeeper-3.4.14# bin/zkServer.sh start
ZooKeeper JMX enabled by default
Using config: /usr/local/zookeeper/zookeeper-3.4.14/bin/../conf/zoo.cfg
Starting zookeeper ... STARTED
root@ubuntu:/usr/local/zookeeper/zookeeper-3.4.14# bin/zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /usr/local/zookeeper/zookeeper-3.4.14/bin/../conf/zoo.cfg
Error contacting service. It is probably not running.
```

此时需要修改`zkServer.sh文件`，在文件开头添加如下内容：

```bash
export JAVA_HOME=/opt/jdk/jdk1.8.0_211
export PATH=$JAVA_HOME/bin:$PATH
```

然后重新启动 zookeeper，这样就能看到进程信息了：

```bash
root@ubuntu:/usr/local/zookeeper/zookeeper-3.4.14# bin/zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /usr/local/zookeeper/zookeeper-3.4.14/bin/../conf/zoo.cfg
Mode: standalone
```

或者使用`ps -ef|grep zookeeper`查看。

常用的命令如下：

```bash
# 查看zookeeper：进入bin目录 输入以下命令
ps -aux | grep 'zookeeper' 

# zookeeper 的启动命令
bin/zkServer.sh start

# zookeeper 的停止命令
bin/zkServer.sh stop

# zookeeper 的状态查看命令
bin/zkServer.sh status
```

### 服务注册端

下面先来看服务注册端的代码，服务注册端需要把 RPC Server 端的**服务名**和**IP地址及端口号**注册到 ZooKeeper 的ZNode 节点中。我这里是把所有的服务都注册在 ZooKeeper 的`/rpcservers`节点下，代码如下：

```java
public class ZookeeperServiceRegister {

    private ZooKeeper zk = null;

    public void zkClient() throws IOException {
        zk = new ZooKeeper(ZkConstants.zkhosts, ZkConstants.SESSION_TIMEOUT, null);
    }

    /**
     * 向Zookeeper服务器创建服务节点，服务注册的信息就存在节点中，假设存在/rpcservers下。
     * @param serviceName
     * @param serviceAddress
     */
    public void serviceRegistry(String serviceName, String serviceAddress) throws KeeperException, InterruptedException {
        // 首先创建父节点（持久化）
        if (zk.exists(ZkConstants.REGISTRY_PATH, false) == null) {
            zk.create(ZkConstants.REGISTRY_PATH, null, ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
        }
        // 创建服务节点
        String result = zk.create(ZkConstants.REGISTRY_PATH + "/" + serviceName, serviceAddress.getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL);
        if (result != null) {
            System.out.println(serviceName + "注册到zookeeper成功。");
        } else {
            System.out.println("注册失败。");
        }
    }
}
```

然后在RPC服务端的启动类中加入以下代码，表示RPC服务端将自己提供的服务注册到了 ZooKeeper 中：

```java
// 将Server端的服务注册到ZooKeeper
ZookeeperServiceRegister serviceRegister = new ZookeeperServiceRegister();
serviceRegister.zkClient();
serviceRegister.serviceRegistry("hdu.dqj.RPCServer.Service.HelloService", "127.0.0.1:8090");
```

可以在`zkCli`中通过输入命令`ls /`查看注册成功的节点：

![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/RPC/v3%E8%8A%82%E7%82%B9%E6%B3%A8%E5%86%8C%E6%88%90%E5%8A%9F.png)

### 服务发现端

服务发现端需要根据提供的服务名去 ZooKeeper 中找到对应节点下的数据，即提供该服务的所有服务器地址。这里的负载均衡策略比较简单，使用随机选择的方式，代码如下：

```java
public class ZookeeperServiceDiscover {

    private ZooKeeper zk = null;
    private String serviceName = null;   // 客户端要查找的服务名

    public ZookeeperServiceDiscover(String serviceName) {
        this.serviceName = serviceName;
    }

    public void zkClient() throws IOException {
        zk = new ZooKeeper(ZkConstants.zkhosts, ZkConstants.SESSION_TIMEOUT, new Watcher() {
            @Override
            public void process(WatchedEvent watchedEvent) {}
        });
    }

    /**
     * 从zk中获取服务器地址
     * @param serviceName
     * @return
     * @throws Exception
     */
    public String serviceDiscover(String serviceName) throws Exception {
        // 获取rpcservers节点下的所有子节点，并注册监听
        List<String> children = zk.getChildren(ZkConstants.REGISTRY_PATH, true);
        // 服务对应的服务器列表
        ArrayList<String> serverList = new ArrayList<>();
        for (String child : children) {
            if (serviceName.equals(child)) {
                byte[] data = zk.getData(ZkConstants.REGISTRY_PATH + "/" + child, false, null);
                serverList.add(new String(data));
            }
        }
        String serverAddress = "";
        if (serverList.size() == 1) {
            serverAddress = serverList.get(0);  // 服务对应一个ip
        } else {
            // 服务对应多个ip，则随机选一个
            serverAddress = serverList.get(ThreadLocalRandom.current().nextInt(serverList.size()));
        }
        return serverAddress;
    }
}
```

客户端的事件处理器根据客户端想要的服务，去 ZooKeeper 中查找对应的服务器地址：

```java
public MyInvocationHandler(Class target) throws Exception {
    this.target = target;
    String serviceName = target.getName();
    System.out.println(serviceName);
    ZookeeperServiceDiscover serviceDiscover = new ZookeeperServiceDiscover(serviceName);
    serviceDiscover.zkClient(); // 创建服务发现的客户端
    // 服务端的地址（包括ip地址和端口号）
    String serverAddress = serviceDiscover.serviceDiscover(serviceName);
    String[] str = serverAddress.split(":");
    address = str[0];
    PORT = Integer.parseInt(str[1]);
}
```

启动`RPCServer`和`RPCClient`，可以看到交互成功：

![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/RPC/v3%E5%8A%A0%E5%85%A5ZooKeeper%E5%90%8E%E8%BF%90%E8%A1%8C%E6%88%90%E5%8A%9F.png)

## 注意点

1. 在创建节点时，一开始我把参数`createMode`写成`PERSISTENT_SEQUENTIAL`，这样会报错：`KeeperErrorCode = NoNode for rpcservers`，意思是没法找到`rpcservers`节点。是因为加了`_SEQUENTIAL`，就会在节点后自动加上序列数，如`rpcservers0000000001   rpcservers0000000002   rpcservers0000000000`等，所以为了避免错误，这里不用使用序列的形式。我想序列的形式应该是在有很多服务器都同提供一种服务时，用于区分这些服务器的。
2. 当有多台服务器都提供某种服务时，肯定会涉及到负载均衡策略，我这里写的比较简单。还可以使用轮询、哈希等方式。

代码我传到 [github](https://github.com/CaryDai/RPC_V3) 上了。

## 参考资料

- [zookeeper api文档](http://zookeeper.apache.org/doc/r3.4.8/api/index.html)
- [《ZooKeeper官方指南》ZooKeeper 使用 ACL 进行访问控制](http://ifeve.com/zookeeper-access-control-using-acls/)
- [zookeeper 启动出错问题排查](https://www.iteye.com/blog/flyer0126-2230958)

