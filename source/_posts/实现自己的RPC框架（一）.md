---
title: 实现自己的RPC框架(一)
date: 2019-12-03 20:05:44
tags: RPC
categories: 分布式
description: 第一版RPC，极简。
---

## 前言

RPC (Remote Procedure Call) —— 远程过程调用，它是一种通过网络从远程计算机程序上请求服务，而不需要了解底层网络技术的协议。直白点说它可以让你调用远程接口，就像调用本地方法一样，调用者感知不到远程调用的逻辑。

## 思路

第一个版本以实现基本功能为主。Client 端需要有一个代理类（Client 在调用服务时，看起来就像在本地调用一样），这个代理类内部是通过 RPC 方式来远程调用 Server 端的实现类。

Server 端需要有 Client 端接口的具体实现，同样也需要有一个类来进行数据的序列化与反序列化，与 Client 端的代理类对应。

整个流程如下图所示：

![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/RPC/v1%E8%BF%87%E7%A8%8B%E5%9B%BE_.png)

用到的技术有`静态代理`、`Socket编程`、`Java序列化`、`BIO`。

## 核心代码

项目结构：

![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/RPC/v1%E9%A1%B9%E7%9B%AE%E7%BB%93%E6%9E%84_.png)

`HelloService`

```java
public interface HelloService {
    String sayHi(String name) throws Throwable;
}
```

`HelloServiceImpl`

```java
public class HelloServiceImpl implements HelloService {
    // 方法返回的String类型实现了 java.io.Serializable 接口，因此能被序列化。
    @Override
    public String sayHi(String name) {
        return "Hi," + name;
    }
}
```

客户端处理类的实现：

```java
public class ClientStub implements HelloService {
    private static final int PORT = 8090;   // 服务器端端口号
    private static final String address = "127.0.0.1";  // 服务器端IP地址

    public String sayHi(String name) throws Throwable {
        ObjectOutputStream output = null;
        ObjectInputStream input = null;
        Socket socket = null;
        try {
            socket = new Socket(address, PORT);
            System.out.println("Client传递信息中...");
            // 将请求信息序列化
            output = new ObjectOutputStream(socket.getOutputStream());
            // 将请求发给服务提供方
            output.writeObject(name);

            // 将响应信息反序列化
            input = new ObjectInputStream(socket.getInputStream());
            Object response = input.readObject();
            System.out.println("Client收到消息！");
            return (String) response;
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            if (output != null) output.close();
            if (input != null)  input.close();
            if (socket != null) socket.close();
        }
        return null;
    }
}
```

服务端调用服务并返回结果：

```java
public class ServerStub {
    private HelloService helloService = new HelloServiceImpl();

    public static void main(String[] args) throws Throwable {
        new ServerStub().run();
    }

    private void run() throws Throwable {
        ServerSocket listener = new ServerSocket(8090);
        System.out.println("Server等待客户端的连接...");
        ObjectOutputStream output = null;
        ObjectInputStream input = null;
        Socket socket = null;
        try {
            while (true) {
                socket = listener.accept();
                // 将收到的请求信息反序列化
                input = new ObjectInputStream(socket.getInputStream());
                Object o = input.readObject();
                System.out.println("Server收到：" + o.toString());

                // 调用服务
                String res = helloService.sayHi((String)o);

                // 返回结果
                output = new ObjectOutputStream(socket.getOutputStream());
                output.writeObject(res);
                System.out.println("Server返回：" + res);
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            if (output != null) output.close();
            if (input != null) input.close();
            if (socket != null) socket.close();
            listener.close();
        }
    }
}
```

客户端进行调用：

```java
public class RPCClient {
    public static void main(String[] args) throws Throwable {
        HelloService helloService = new ClientStub();
        String str = helloService.sayHi("XD");
        System.out.println(str);
    }
}
```

## 缺点与改进

这个 RPC 实在太简单了，Client 端只有一个接口，Server 端也只有这个接口对应的实现类。但是真实情况下怎么可能只有一个接口和一个实现类，而且 Server 端是根据 Client 端发送过来的`类名、方法名、方法参数等`来进行接口的选择，此时可以使用`反射`来生成对应的实现类，然后把执行结果返回回去。

再来看 Client 端，第一步是把`类名、方法名、方法参数`等传到服务端，这个是针对所有的方法。第一种想法是将每个方法都用 Socket 传输，就像这个版本里写的一样。但是如果有几百个方法呢，总不可能每个方法都写一遍传输逻辑吧。应该想到可以使用动态代理，每次调用时，动态生成一个代理类对象，这个代理类对象才是真正进行 Server 端调用的工作。

下个版本将对以上两点进行改进。