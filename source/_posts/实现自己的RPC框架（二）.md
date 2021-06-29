---
title: 实现自己的RPC框架（二）
date: 2019-12-25 10:56:49
tags: RPC
categories: 分布式
description: 第二版RPC，使用“Socket传输 + Java序列化 + BIO + JDK动态代理 + 反射”实现。
---

## 思路

在第一个版本的末尾，指出可以使用动态代理与反射对版本一进行改进。所以整体的思路应该如下图所示：

![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/RPC/v2%E8%BF%87%E7%A8%8B%E5%9B%BE.png)

## 核心代码

项目结构：

![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/RPC/v2%E9%A1%B9%E7%9B%AE%E7%BB%93%E6%9E%84.png)

自定义事件处理器：

```java
public class MyInvocationHandler implements InvocationHandler {
    // 被代理的接口
    private Class target;

    public MyInvocationHandler(Class target) {
        this.target = target;
    }

    private static final int PORT = 8090;   // 服务器端端口号
    private static final String address = "127.0.0.1";  // 服务器端IP地址

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        ObjectOutputStream output = null;
        ObjectInputStream input = null;
        Socket socket = null;
        try {
            socket = new Socket(address, PORT);
            System.out.println("Client传递信息中...");

            // 将请求信息序列化
            output = new ObjectOutputStream(socket.getOutputStream());
            // 客户端需要把调用的方法名、方法所属的类名或接口名、方法参数类型、方法参数值发送给服务器端。
            output.writeUTF(method.getName());
            output.writeUTF(target.getName());
            output.writeObject(method.getParameterTypes());
            output.writeObject(args);
            System.out.println("Client发送信息完毕。");

            // 将响应信息反序列化
            input = new ObjectInputStream(socket.getInputStream());
            // 接收方法执行结果
            Object o = input.readObject();
            System.out.println("Client收到消息！");
            return o;
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

`invoke`方法里面对调用方法进行增强，不再指定特定的方法名、方法参数等信息，而是使用反射得到这些信息。

服务端用来接收并处理消息的类：

```java
public class ServerStub {
    // 用来存放Server端服务对象的缓存
    private HashMap<String, Class> remoteServices = new HashMap<String, Class>();

    // 把一个Server端对象放到缓存中
    public void register(Class className, Class remoteImpl) {
        remoteServices.put(className.getName(), remoteImpl);
    }

    public void run() throws Throwable {
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
                // 依次从客户端接收方法名、方法所属的类名或接口名、方法参数类型、方法参数值
                String methodName = input.readUTF();
                String className = input.readUTF();
                Class<?>[] parameterTypes = (Class<?>[]) input.readObject();
                Object[] params = (Object[]) input.readObject();
                System.out.println("Server收到：" + className + ", " + methodName);

                // 根据上述收到的信息，调用相应类中的方法并返回结果。
                Class serverClass = remoteServices.get(className);
                Method method = serverClass.getMethod(methodName, parameterTypes);
                Object result = method.invoke(serverClass.newInstance(), params);

                // 返回结果
                output = new ObjectOutputStream(socket.getOutputStream());
                output.writeObject(result);
                System.out.println("Server返回：" + result);
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

可以看到这里Server端服务对象的存储我选择了`HashMap`。当时是这样想的：一个服务接口对应一个实现类，这不就是类似`key-value`键值对的形式么，所以考虑用`HashMap`来存储。之后用反射接收客户端传递的信息，并从`HashMap`中得到对应方法的实现，并调用返回结果。

客户端的启动类：

```java
public class RPCClient {
    public static void main(String[] args){
        MyInvocationHandler invocationHandler = new MyInvocationHandler(HelloService.class);
        // 返回代理类实例。
        HelloService proxy = (HelloService) Proxy.newProxyInstance(HelloService.class.getClassLoader(),
                new Class<?>[]{HelloService.class}, invocationHandler);
        String result = proxy.sayHi("XD");
        System.out.println(result);
    }
}
```

注意这里`Proxy.newProxyInstance`的第二个参数要写成数组的形式，一开始我写的是`HelloService.class.getInterfaces()`却报了错：

```bash
Exception in thread "main" java.lang.ClassCastException: com.sun.proxy.$Proxy0 cannot be cast to hdu.dqj.RPCServer.Service.HelloService
	at hdu.dqj.Client.RPCClient.main(RPCClient.java:18)
```

报的是类型转换错误。

但是`getInterfaces()`返回的也是一个 Class 数组啊。后来在[这篇博客](https://blog.csdn.net/pange1991/article/details/81222616)中找到了同样的问题。原来第二个参数的解释是：` the list of interfaces for the proxy class to implement `，即“==代理类要实现的接口列表==”。通过输出`HelloService.class.getInterfaces()`可以看到：`[Ljava.lang.Class;@1b6d3586`，即它返回的实际上是接口`HelloService`继承的上一级接口。

因此只要写出代理类要实现的接口列表即可，可以这样写：`new Class<?>[]{HelloService.class}`，也可以这样写：`HelloServiceImpl.class.getInterfaces()`。

服务端的启动类：

```java
public class RPCServer {
    public static void main(String[] args) throws Throwable {
        // 为每个客户端创建一个线程。
        new Thread(() -> {
            try {
                ServerStub server = new ServerStub();
                server.register(HelloService.class, HelloServiceImpl.class);
                server.run();
            } catch (Throwable e) {
                e.printStackTrace();
            }
        }).start();
    }
}
```

## 缺点与改进

- 当前的 BIO 通信模型中，服务端只能接收一个客户端的请求，如果有多个客户端向服务端请求数据呢？一种解决方法是使用多线程的方式，服务端使用多线程来支持多个客户端的连接。这是典型的“一请求一应答”的方式，那么这里到底需要几个线程呢？其实取决于客户端的数量，多个客户端，就启动多个线程来处理，所以可以使用**线程池**来解决这个问题。
- 可以使用 ZooKeeper 作为服务注册中心，实现服务的注册与发现。

下个版本将对以上两点进行改进。