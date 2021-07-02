---
title: RocketMQ源码分析——生产者启动流程
date: 2020-07-11 11:14:57
tags: RocketMQ
categories: 消息队列
description: RocketMQ生产者启动流程源码分析
---

## 前言

之前的文章中提到，当 Broker 宕机时，NameServer 至少要 120s 才能将该 Broker 从路由表中移除。**那么在 Broker 宕机期间，如果消息生产者 Producer 获取到的路由信息包含了已经宕机的 Broker，就会导致消息发送失败**，这个时候就需要相关处理逻辑来应对这种情况的发生。本篇先分析生产者启动流程，之后将分析消息发送流程。

## Producer启动流程

要想探知启动流程，最好的方式就是`Debug`。源码的`example`包中有`Producer`生产者，可以使用它来作为启动入口。

我们先来看下`org\apache\rocketmq\example\quickstart\Producer.java`：

```java
public class Producer {
    public static void main(String[] args) throws MQClientException, InterruptedException {

        DefaultMQProducer producer = new DefaultMQProducer("please_rename_unique_group_name");

        producer.setNamesrvAddr("127.0.0.1:9876");

        producer.start();

        ...
    }
}
```

现在分析的是生产者启动流程，所以只需要关心前面三句。分别在第4行和第8行打上断点，`debug`方式启动`Nameserver`、`Broker`和`Producer`。

```java
public DefaultMQProducer(final String producerGroup) {
    this(null, producerGroup, null);
}

public DefaultMQProducer(final String namespace, final String producerGroup, RPCHook rpcHook) {
    this.namespace = namespace;
    this.producerGroup = producerGroup;
    defaultMQProducerImpl = new DefaultMQProducerImpl(this, rpcHook);
}
```

因为`DefaultMQProducer`继承自`ClientConfig`，所以会先调用父类的默认构造函数，主要就是设置了一些字段值，后面我们会看到。接着执行类`DefaultMQProducer`成员变量/实例变量的初始化，包括设置`createTopicKey`、`defaultTopicQueueNums`、`sendMsgTimeout`等等字段。最后才会执行该构造函数的方法体。

在执行`start()`方法前，看下`producer`的状态：

![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/RocketMQ/%E5%AE%A2%E6%88%B7%E7%AB%AF%E5%90%AF%E5%8A%A8.png)

接着继续进到`start()`方法里面：

```java
public void start() throws MQClientException {
    this.setProducerGroup(withNamespace(this.producerGroup));
    // 调用defaultMQProducerImpl的start()方法
    this.defaultMQProducerImpl.start();
    ...
}
```

```java
public void start(final boolean startFactory) throws MQClientException {
    switch (this.serviceState) {
            // 默认为CREATE_JUST状态
        case CREATE_JUST:
            // 先默认成启动失败，等最后完全启动成功的时候再置为ServiceState.RUNNING
            this.serviceState = ServiceState.START_FAILED;

            // 检查productGroup是否符合要求
            this.checkConfig();

            if (!this.defaultMQProducer.getProducerGroup().equals(MixAll.CLIENT_INNER_PRODUCER_GROUP)) {
                // 改变生产者的instanceName为进程id
                this.defaultMQProducer.changeInstanceNameToPID();
            }

            // 单例模式，获取MQClientInstance对象，客户端实例。也就是Producer所部署的机器实例对象，负责操作的主要对象。
            this.mQClientFactory = MQClientManager.getInstance().getAndCreateMQClientInstance(this.defaultMQProducer, rpcHook);

            // 向MQClientInstance注册，将当前生产者加入到MQClientInstance管理中，方便后续调用网络请求、进行心跳检测等。
            boolean registerOK = mQClientFactory.registerProducer(this.defaultMQProducer.getProducerGroup(), this);
            if (!registerOK) {
                this.serviceState = ServiceState.CREATE_JUST;
                throw new MQClientException("The producer group[" + this.defaultMQProducer.getProducerGroup()
                                            + "] has been created before, specify another name please." + FAQUrl.suggestTodo(FAQUrl.GROUP_NAME_DUPLICATE_URL),
                                            null);
            }

            // 将topic信息存到topicPublishInfoTable这个map里
            this.topicPublishInfoTable.put(this.defaultMQProducer.getCreateTopicKey(), new TopicPublishInfo());

            if (startFactory) {
                // 启动MQClientInstance，如果MQClientInstance已经启动，则本次启动不会真正执行
                mQClientFactory.start();
            }

            log.info("the producer [{}] start OK. sendMessageWithVIPChannel={}", this.defaultMQProducer.getProducerGroup(),
                     this.defaultMQProducer.isSendMessageWithVIPChannel());
            // 都启动完成，没报错的话，就将状态改为运行中
            this.serviceState = ServiceState.RUNNING;
            break;
        case RUNNING:
        case START_FAILED:
        case SHUTDOWN_ALREADY:
            throw new MQClientException("The producer service state not OK, maybe started once, "
                                        + this.serviceState
                                        + FAQUrl.suggestTodo(FAQUrl.CLIENT_SERVICE_NOT_OK),
                                        null);
        default:
            break;
    }

    this.mQClientFactory.sendHeartbeatToAllBrokerWithLock();
}
```

16行，创建`MQClientInstance`：

```java
public MQClientInstance getAndCreateMQClientInstance(final ClientConfig clientConfig, RPCHook rpcHook) {
    // 创建clientId
    String clientId = clientConfig.buildMQClientId();
    // 根据clientId去factoryTable中取MQClientInstance，同一个clientId只会对应一个MQClientInstance
    MQClientInstance instance = this.factoryTable.get(clientId);
    if (null == instance) {
        instance =
            new MQClientInstance(clientConfig.cloneClientConfig(),
                                 this.factoryIndexGenerator.getAndIncrement(), clientId, rpcHook);
        MQClientInstance prev = this.factoryTable.putIfAbsent(clientId, instance);
        if (prev != null) {
            instance = prev;
            log.warn("Returned Previous MQClientInstance for clientId:[{}]", clientId);
        } else {
            log.info("Created new MQClientInstance for clientId:[{}]", clientId);
        }
    }

    return instance;
}
```

因为第一次创建，所以会使用`MQClientInstance`的构造函数进行实例化，创建完成返回该实例。

实例创建完成后，向`MQClientInstance`注册，将当前生产者加入到`MQClientInstance`管理中，方便后续调用网络请求、进行心跳检测等。

最后会进入真正的启动方法`MQClientInstance#start()`：

```java
public void start() throws MQClientException {

    synchronized (this) {
        // 默认为CREATE_JUST状态
        switch (this.serviceState) {
            case CREATE_JUST:
                // 先默认成启动失败，等最后完全启动成功的时候再置为ServiceState.RUNNING
                this.serviceState = ServiceState.START_FAILED;
                // If not specified,looking address from name server
                if (null == this.clientConfig.getNamesrvAddr()) {
                    this.mQClientAPIImpl.fetchNameServerAddr();
                }
                // Start request-response channel
                this.mQClientAPIImpl.start();
                // Start various schedule tasks
                /*
                     * 启动各种定时任务
                     * 1.每隔2分钟去检测namesrv的变化
                     * 2.每隔30s从nameserver获取topic的路由信息有没有发生变化，或者说有没有新的topic路由信息
                     * 3.每隔30s清除下线的broker
                     * 4.每隔5s持久化所有的消费进度
                     * 5.每隔1分钟检测线程池大小是否需要调整
                     */
                this.startScheduledTask();
                // Start pull service
                // 启动拉取消息服务
                this.pullMessageService.start();
                // Start rebalance service
                // 启动Rebalance负载均衡服务
                this.rebalanceService.start();
                // Start push service
                this.defaultMQProducer.getDefaultMQProducerImpl().start(false);
                log.info("the client factory [{}] start OK", this.clientId);
                // 都启动完成，没报错的话，就将状态改为运行中
                this.serviceState = ServiceState.RUNNING;
                break;
            case RUNNING:
                break;
            case SHUTDOWN_ALREADY:
                break;
            case START_FAILED:
                throw new MQClientException("The Factory object[" + this.getClientId() + "] has been created before, and failed.", null);
            default:
                break;
        }
    }
}
```

第24行`this.startScheduledTask();`启动各种定时任务，其中有一个定时任务是：每隔30s清除下线的 Broker：

```java
// 每隔30s清除下线的broker
this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

    @Override
    public void run() {
        try {
            MQClientInstance.this.cleanOfflineBroker();
            MQClientInstance.this.sendHeartbeatToAllBrokerWithLock();
        } catch (Exception e) {
            log.error("ScheduledTask sendHeartbeatToAllBroker exception", e);
        }
    }
}, 1000, this.clientConfig.getHeartbeatBrokerInterval(), TimeUnit.MILLISECONDS);
```

如果 Broker 已下线，就将其从缓存中清除。

## 总结

至此，生产者启动流程就分析完了。主要是启动了各种定时任务、拉取消息服务、负载均衡服务、默认的推送服务。

下一篇将介绍消息发送。