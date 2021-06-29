---
title: RocketMQ源码分析——NameServer路由注册与故障剔除
date: 2020-07-10 16:58:06
tags: RocketMQ
categories: RocketMQ
description: NameServer的路由注册与故障剔除源码分析
---

## 前言

在介绍路由注册与剔除前，首先需要熟悉几个相关类。

### RouteInfoManager

`org\apache\rocketmq\namesrv\routeinfo\RouteInfoManager`是`NameServer`路由信息管理类，主要存储了以下信息：

```java
// Topic消息队列路由信息，消息发送时根据路由表进行负载均衡
private final HashMap<String/* topic */, List<QueueData>> topicQueueTable;
// Broker基础信息，包含brokerName、所属集群名称、主备Broker地址
private final HashMap<String/* brokerName */, BrokerData> brokerAddrTable;
// Broker集群信息，存储集群中所有Broker名称
private final HashMap<String/* clusterName */, Set<String/* brokerName */>> clusterAddrTable;
// Broker状态信息。NameServer每次收到心跳包时会替换该信息
private final HashMap<String/* brokerAddr */, BrokerLiveInfo> brokerLiveTable;
// Broker上的FilterServer列表，用于类模式消息过滤
private final HashMap<String/* brokerAddr */, List<String>/* Filter Server */> filterServerTable;
```

这里要先简单介绍一下 Broker 。看下面这张图（来自网络）：

<img src="https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/RocketMQ/2.Broker%E5%92%8CTopic.jpg" style="zoom:80%;" />

一个 Topic 部署在多台 Broker 上，每台 Broker 可以存储多个 Topic 信息。

RocketMQ 基于订阅发布机制，一个Topic 拥有多个消息队列，一个 Broker 为每一 Topic 默认创建4个读队列4个写队列。多个Broker 组成一个集群， BrokerName 由相同的多台Broker组成 Master-Slave 架构， brokerId 为0代表 Master ， 大于0表示 Slave 。

### QueueData

```java
public class QueueData implements Comparable<QueueData> {
    // Queue所属的Broker名字
    private String brokerName;
    // 该Broker上，针对该Topic，配置的读队列个数
    private int readQueueNums;
    // 该Broker上，针对该Topic，配置的写队列个数
    private int writeQueueNums;
    private int perm;
    private int topicSynFlag;
}
```

结合上面的图其实挺容易理解，`RouteInfoManager`中的`topicQueueTable`消息队列表中指明了一个`topic`对应多个`QueueData`，所以`QueueData`就是该主题当前所属 Broker 消息队列，分读队列和写队列。

### BrokerLiveInfo

```java
class BrokerLiveInfo {
    // 上次收到 Broker 心跳包的时间
    private long lastUpdateTimestamp;
    private DataVersion dataVersion;
    private Channel channel;
    private String haServerAddr;
}
```

它是`RouteInfoManager`的一个内部类，记录了 Broker 状态信息。

## 路由注册

路由注册指的是 Broker 在启动的时候，向 NameServer 注册自己的 IP 地址等信息。

> 《RocketMQ中提到》：RocketMQ 路由注册是通过 Broker 与 NameServer 的心跳功能实现的。Broker 启动时向集群中所有的 NameServer 发送心跳语句，每隔 30s 向集群中所有 NameServer 发送心跳包，NameServer 收到 Broker 心跳包时会更新 brokerLiveTable（Broker 状态信息） 缓存中 BrokerLivelnfo 的 lastUpdateTimestamp ，然后 NameServer 每隔 10s 扫描 brokerLiveTable ，如果连续 120s 没有收到心跳包，NameServer 将移除该 Broker 的路由信息同时关闭 Socket 连接。

### Broker 发送心跳包

Broker 发送心跳包的代码段位于`org\apache\rocketmq\broker\BrokerController#start`中：

```java
// Broker端心跳包发送
this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

    @Override
    public void run() {
        try {
            // 向所有NameServer发送心跳包
            BrokerController.this.registerBrokerAll(true, false, brokerConfig.isForceRegister());
        } catch (Throwable e) {
            log.error("registerBrokerAll Exception", e);
        }
    }
    // 默认是每隔30s发送一次
}, 1000 * 10, Math.max(10000, Math.min(brokerConfig.getRegisterNameServerPeriod(), 60000)), TimeUnit.MILLISECONDS);
```

第8行`registerBrokerAll`最终会调用`BrokerOuterAPI#registerBrokerAll`：

```java
public List<RegisterBrokerResult> registerBrokerAll(
    final String clusterName,
    final String brokerAddr,
    final String brokerName,
    final long brokerId,
    final String haServerAddr,
    final TopicConfigSerializeWrapper topicConfigWrapper,
    final List<String> filterServerList,
    final boolean oneway,
    final int timeoutMills,
    final boolean compressed) {

    final List<RegisterBrokerResult> registerBrokerResultList = Lists.newArrayList();
    List<String> nameServerAddressList = this.remotingClient.getNameServerAddressList();
    if (nameServerAddressList != null && nameServerAddressList.size() > 0) {

        // 请求包头对象
        final RegisterBrokerRequestHeader requestHeader = new RegisterBrokerRequestHeader();
        // broker地址
        requestHeader.setBrokerAddr(brokerAddr);
        // brokerId：brokerId,0:Master;大于0:Slave
        requestHeader.setBrokerId(brokerId);
        // broker名称
        requestHeader.setBrokerName(brokerName);
        // 集群名称
        requestHeader.setClusterName(clusterName);
        // master地址，初次请求时该值为空，slave向NameServer注册后返回
        requestHeader.setHaServerAddr(haServerAddr);
        // 设置压缩格式
        requestHeader.setCompressed(compressed);

        // 创建请求体对象
        RegisterBrokerBody requestBody = new RegisterBrokerBody();
        // 主题配置
        requestBody.setTopicConfigSerializeWrapper(topicConfigWrapper);
        // 设置消息过滤服务器列表
        requestBody.setFilterServerList(filterServerList);
        final byte[] body = requestBody.encode(compressed);
        final int bodyCrc32 = UtilAll.crc32(body);
        requestHeader.setBodyCrc32(bodyCrc32);
        final CountDownLatch countDownLatch = new CountDownLatch(nameServerAddressList.size());
        // 遍历所有NameServer列表
        for (final String namesrvAddr : nameServerAddressList) {
            brokerOuterExecutor.execute(new Runnable() {
                @Override
                public void run() {
                    try {
                        // 分别向NameServer注册
                        RegisterBrokerResult result = registerBroker(namesrvAddr,oneway, timeoutMills,requestHeader,body);
                        if (result != null) {
                            registerBrokerResultList.add(result);
                        }

                        log.info("register broker[{}]to name server {} OK", brokerId, namesrvAddr);
                    } catch (Exception e) {
                        log.warn("registerBroker Exception, {}", namesrvAddr, e);
                    } finally {
                        countDownLatch.countDown();
                    }
                }
            });
        }

        try {
            countDownLatch.await(timeoutMills, TimeUnit.MILLISECONDS);
        } catch (InterruptedException e) {
        }
    }

    return registerBrokerResultList;
}
```

这个方法首先封装了请求头和请求体，接着将请求体编码成 byte 数组，对请求头添加 CRC 校验。接着遍历所有NameServer 列表，分别注册 Broker 的信息。CountDownLatch 计数为 NameServer 列表的长度，保证所有 NameServer 都遍历完之后再返回所有的注册结果。

发送的心跳包中包含 BrokerId、Broker 地址、Broker 名称、Broker 所属集群名称、Broker 关联的 FilterServer 列表。

### NameServer 处理心跳包

NameServer 的网络请求处理类为`org\apache\rocketmq\namesrv\processor\DefaultRequestProcessor`，这个类里面根据请求的不同类型，转发到相应处理逻辑。当 NameServer 接收到`RequestCode.REGISTER_BROKER`的请求类型时，请求最终会转发到`RouteInfoManager#registerBroker`。核心逻辑如下：

```java
// 路由注册需要加锁，防止并发修改RouteInfoManager中的路由表
this.lock.writeLock().lockInterruptibly();

// 1.维护clusterAddrTable
// 首先判断Broker所属集群是否存在
Set<String> brokerNames = this.clusterAddrTable.get(clusterName);
if (null == brokerNames) {
    // 若不存在，则创建并添加到clusterAddrTable
    brokerNames = new HashSet<String>();
    this.clusterAddrTable.put(clusterName, brokerNames);
}
// 然后将broker名加入到Broker集群中
brokerNames.add(brokerName);

boolean registerFirst = false;

// 2.维护brokerAddrTable
// 首先从brokerAddrTable中根据brokerName获取Broker信息
BrokerData brokerData = this.brokerAddrTable.get(brokerName);
if (null == brokerData) {
    // 如果brokerData不存在，说明是第一次注册
    registerFirst = true;
    // 新建BrokerData并放入到brokerAddrTable
    brokerData = new BrokerData(clusterName, brokerName, new HashMap<Long, String>());
    this.brokerAddrTable.put(brokerName, brokerData);
}
Map<Long, String> brokerAddrsMap = brokerData.getBrokerAddrs();
//Switch slave to master: first remove <1, IP:PORT> in namesrv, then add <0, IP:PORT>
//The same IP:PORT must only have one record in brokerAddrTable
Iterator<Entry<Long, String>> it = brokerAddrsMap.entrySet().iterator();
while (it.hasNext()) {
    Entry<Long, String> item = it.next();
    if (null != brokerAddr && brokerAddr.equals(item.getValue()) && brokerId != item.getKey()) {
        it.remove();
    }
}

// 3.维护topicQueueTable
// 如果BrokerData存在，则直接替换原先的
String oldAddr = brokerData.getBrokerAddrs().put(brokerId, brokerAddr);
// 将registerFirst设置为false，表示非第一次注册
registerFirst = registerFirst || (null == oldAddr);

if (null != topicConfigWrapper
    && MixAll.MASTER_ID == brokerId) {
    // 如果Broker为Master，并且Broker Topic配置信息发生变化或者是初次注册
    if (this.isBrokerTopicConfigChanged(brokerAddr, topicConfigWrapper.getDataVersion())
        || registerFirst) {
        // 获取所有的topic配置信息
        ConcurrentMap<String, TopicConfig> tcTable =
            topicConfigWrapper.getTopicConfigTable();
        if (tcTable != null) {
            // 遍历tcTable
            for (Map.Entry<String, TopicConfig> entry : tcTable.entrySet()) {
                // 如果该broker的topic是初次注册，则需要创建topic路由元信息并填充topicQueueTable；
                // 如果该broker的topic配置信息发生变化，则需要更新topic路由元信息并填充topicQueueTable
                // 其实就是为默认主题自动注册路由信息，其中包含MixAll.DEFAULT_TOPIC的路由信息。当消息生产者发送主题时，
                // 如果该主题未创建并且BrokerConfig的autoCreateTopicEnable为true时，将返回MixAll.DEFAULT_TOPIC的路由信息。
                this.createAndUpdateQueueData(brokerName, entry.getValue());
            }
        }
    }
}

// 4.维护BrokerLiveInfo，即存活Broker信息表，它是执行路由删除的重要依据
BrokerLiveInfo prevBrokerLiveInfo = this.brokerLiveTable.put(brokerAddr,
                                                             new BrokerLiveInfo(
                                                                 System.currentTimeMillis(),
                                                                 topicConfigWrapper.getDataVersion(),
                                                                 channel,
                                                                 haServerAddr));
if (null == prevBrokerLiveInfo) {
    log.info("new broker registered, {} HAServer: {}", brokerAddr, haServerAddr);
}

// 5.维护filterServerTable
// 注册Broker的过滤器Server地址列表，一个Broker上会关联多个FilterServer消息过滤服务器
if (filterServerList != null) {
    if (filterServerList.isEmpty()) {
        this.filterServerTable.remove(brokerAddr);
    } else {
        this.filterServerTable.put(brokerAddr, filterServerList);
    }
}

// 如果此Broker为从节点，则需要查找Broker的Master节点信息，并更新对应的masterAddr属性。
if (MixAll.MASTER_ID != brokerId) {
    String masterAddr = brokerData.getBrokerAddrs().get(MixAll.MASTER_ID);
    if (masterAddr != null) {
        BrokerLiveInfo brokerLiveInfo = this.brokerLiveTable.get(masterAddr);
        if (brokerLiveInfo != null) {
            result.setHaServerAddr(brokerLiveInfo.getHaServerAddr());
            result.setMasterAddr(masterAddr);
        }
    }
}
```

可能看起来有点长，但是其实逻辑很清晰。NameServer 每收到一个心跳包，便会更新`brokerLiveTable`中关于 Broker 的状态信息以及各个路由表（`clusterAddrTable`、`brokerAddrTable`、`topicQueueTable`、`filterServerTable`）

其中，这里的锁用到了`ReentrantReadWriteLock`读写锁。其实也好理解，NameServer 允许多个 Producer 或 Consumer 并发读，但是同一时刻只处理一个 Broker 心跳包，后面的心跳包会被放到阻塞队列中进行等待。对于读多写少的场景，`ReentrantReadWriteLock`是最佳选择。

### 总结

至此为止，NameServer 的路由注册模块就分析完了，总结一下：

Broker 每隔 30s 会向 NameServer 集群中的所有 NameServer 发送心跳包，NameServer 收到心跳包后，使用读写锁的写锁加锁并更新当前 Broker 的基础信息、集群信息、状态信息、Topic消息队列路由信息以及`FilterServer`列表。

## 路由删除

前面提到，Broker 会每隔 30s 向所有 NameServer 发送心跳包，但是如果 Broker 宕机，NameServer 将无法收到心跳包，此时就需要路由删除。《RocketMQ 中提到》：

> NameServer 会每隔10s 扫描 brokerLiveTable 状态表，如果 BrokerLiveInfo 的 lastUpdateTimestamp 的时间戳距当前时间超过 120s ，则认为 Broker 失效，移除该 Broker，关闭与 Broker 连接，同时更新 topicQueueTable、brokerAddrTable、brokerLiveTable、filterServerTable

### 触发路由删除

RocketMQ 有两个触发点来触发路由删除：

1. NameServer 定时扫描 brokerLiveTable 检测上次心跳包与当前系统时间的时间差，如果时间戳大于120s ，则需要移除该 Broker 信息。
2. Broker 在正常被关闭的情况下，会执行 unregisterBroker 指令。

不管是何种方式触发的路由删除，路由删除的方法都是一样的，就是从 topicQueueTable、brokerAddrTable、brokerLiveTable、filterServerTable 删除与该Broker 相关的信息。

### 进行路由删除

以第一种为例，删除路由的路口在`RouteInfoManager#scanNotActiveBroker`。其实上一篇讲 NameServer 启动流程的时候提到，NamesrvController 的`initialize()`方法中开启了两个定时任务，其中一个就是心跳检测，它调用的就是`scanNotActiveBroker`方法：

```java
public void scanNotActiveBroker() {
    Iterator<Entry<String, BrokerLiveInfo>> it = this.brokerLiveTable.entrySet().iterator();
    // 遍历brokerLiveInfo路由表
    while (it.hasNext()) {
        Entry<String, BrokerLiveInfo> next = it.next();
        // 拿到上一次收到心跳包的时间戳
        long last = next.getValue().getLastUpdateTimestamp();
        // 如果上次收到的时间加上120s小于现在的时间，说明已经有超过120s没有收到心跳包了
        if ((last + BROKER_CHANNEL_EXPIRED_TIME) < System.currentTimeMillis()) {
            // 关闭与该Broker的连接
            RemotingUtil.closeChannel(next.getValue().getChannel());
            // 将该Broker从brokerLiveTable中移除
            it.remove();
            log.warn("The broker channel expired, {} {}ms", next.getKey(), BROKER_CHANNEL_EXPIRED_TIME);
            // 删除与该Broker相关的路由信息
            this.onChannelDestroy(next.getKey(), next.getValue().getChannel());
        }
    }
}
```

逻辑很简单，上面的注释也很清楚，下面看下16行的`onChannelDestroy`方法，如果是这种情况下删除路由，会进入到下面的代码段：

```java
if (brokerAddrFound != null && brokerAddrFound.length() > 0) {

    try {
        try {
            // 删路由需要先获取写锁，而且该写锁对中断进行响应
            this.lock.writeLock().lockInterruptibly();
            // 1.从brokerLiveTable和filterServerTable移除brokerAddrFound
            this.brokerLiveTable.remove(brokerAddrFound);
            this.filterServerTable.remove(brokerAddrFound);
            // 2.维护brokerAddrTable，移除brokerAddrFound对应的broker
            String brokerNameFound = null;
            boolean removeBrokerName = false;
            Iterator<Entry<String, BrokerData>> itBrokerAddrTable =
                this.brokerAddrTable.entrySet().iterator();
            // 遍历brokerAddrTable
            while (itBrokerAddrTable.hasNext() && (null == brokerNameFound)) {
                BrokerData brokerData = itBrokerAddrTable.next().getValue();

                // 遍历brokerData的brokerAddrs，它是一个HashMap，key为brokerId，value为broker的ip地址
                Iterator<Entry<Long, String>> it = brokerData.getBrokerAddrs().entrySet().iterator();
                while (it.hasNext()) {
                    Entry<Long, String> entry = it.next();
                    Long brokerId = entry.getKey();
                    String brokerAddr = entry.getValue();
                    // 找到具体的Broker，从BrokerData中移除
                    if (brokerAddr.equals(brokerAddrFound)) {
                        brokerNameFound = brokerData.getBrokerName();
                        it.remove();
                        log.info("remove brokerAddr[{}, {}] from brokerAddrTable, because channel destroyed",
                                 brokerId, brokerAddr);
                        break;
                    }
                }

                // 如果移除后在BrokerData中不再包含其它Broker，则在brokerAddrTable中移除该brokerName对应的条目
                if (brokerData.getBrokerAddrs().isEmpty()) {
                    removeBrokerName = true;
                    itBrokerAddrTable.remove();
                    log.info("remove brokerName[{}] from brokerAddrTable, because channel destroyed",
                             brokerData.getBrokerName());
                }
            }

            // 3.维护clusterAddrTable
            if (brokerNameFound != null && removeBrokerName) {
                Iterator<Entry<String, Set<String>>> it = this.clusterAddrTable.entrySet().iterator();
                while (it.hasNext()) {
                    Entry<String, Set<String>> entry = it.next();
                    String clusterName = entry.getKey();
                    Set<String> brokerNames = entry.getValue();
                    // 根据BrokerName，从clusterAddrTable中找到Broker并从集群中移除
                    boolean removed = brokerNames.remove(brokerNameFound);
                    if (removed) {
                        log.info("remove brokerName[{}], clusterName[{}] from clusterAddrTable, because channel destroyed",
                                 brokerNameFound, clusterName);

                        // 如果移除后，集群中不包含任何broker，则将该集群从clusterAddrTable中移除
                        if (brokerNames.isEmpty()) {
                            log.info("remove the clusterName[{}] from clusterAddrTable, because channel destroyed and no broker in this cluster",
                                     clusterName);
                            it.remove();
                        }

                        break;
                    }
                }
            }

            // 4.维护topicQueueTable
            if (removeBrokerName) {
                Iterator<Entry<String, List<QueueData>>> itTopicQueueTable =
                    this.topicQueueTable.entrySet().iterator();
                while (itTopicQueueTable.hasNext()) {
                    Entry<String, List<QueueData>> entry = itTopicQueueTable.next();
                    String topic = entry.getKey();
                    List<QueueData> queueDataList = entry.getValue();

                    Iterator<QueueData> itQueueData = queueDataList.iterator();
                    while (itQueueData.hasNext()) {
                        QueueData queueData = itQueueData.next();
                        if (queueData.getBrokerName().equals(brokerNameFound)) {
                            itQueueData.remove();
                            log.info("remove topic[{} {}], from topicQueueTable, because channel destroyed",
                                     topic, queueData);
                        }
                    }

                    if (queueDataList.isEmpty()) {
                        itTopicQueueTable.remove();
                        log.info("remove topic[{}] all queue, from topicQueueTable, because channel destroyed",
                                 topic);
                    }
                }
            }
        } finally {
            // 需要手动释放锁
            this.lock.writeLock().unlock();
        }
    } catch (Exception e) {
        log.error("onChannelDestroy Exception", e);
    }
}
```

其实路由删除的逻辑和路由注册挺像，只不过路由注册是接收到心跳包后执行，路由删除是触发了路由删除事件后执行。同样都是维护那5个路由表（`brokerLiveTable`、`filterServerTable`、`brokerAddrTable`、`clusterAddrTable`、`topicQueueTable`），维护的过程也都需要加锁。

### 总结

至此，路由删除过程也分析完成。其实整个逻辑还是挺清晰的，上面其实已经总结好了。这两个过程都比较简单，核心是维护了那5个路由表。

## 路由发现

最后补充一下路由发现机制。前面的路由注册与路由删除是 NameServer 与 Broker 之间，而路由发现则是 NameServer 与 Producer 和 Consumer 之间。

在之前的文章中提到：NameServer 在 Topic 路由变更的时候，不会马上将变更信息推送给客户端。实际上是由**客户端定时拉取主题最新的路由**。那到底是怎么实现的呢？一起来看下。

首先有一个主题路由结果实体类`TopicRouteData`：

```java
public class TopicRouteData extends RemotingSerializable {
    // 顺序消息配置内容，来自于kvConfig
    private String orderTopicConf;
    // topic队列元数据
    private List<QueueData> queueDatas;
    // topic分布的broker元数据
    private List<BrokerData> brokerDatas;
    // broker上过滤服务器地址
    private HashMap<String/* brokerAddr */, List<String>/* Filter Server */> filterServerTable;
    ...
}
```

之前有讲过，NameServer 默认网络请求处理类`DefaultRequestProcessor`会根据请求的不同类型，调用各自的方法。当收到`RequestCode.GET_ROUTEINTO_BY_TOPIC`请求时，将调用`getRouteInfoByTopic`方法：

```java
public RemotingCommand getRouteInfoByTopic(ChannelHandlerContext ctx,
                                           RemotingCommand request) throws RemotingCommandException {
    final RemotingCommand response = RemotingCommand.createResponseCommand(null);
    final GetRouteInfoRequestHeader requestHeader =
        (GetRouteInfoRequestHeader) request.decodeCommandCustomHeader(GetRouteInfoRequestHeader.class);

    // 调用RouterInfoManager的方法，从路由表topicQueueTable、brokerAddrTable、filterServerTable
    // 中分别填充TopicRouteData中的List<QueueData>、List<BrokerData>和filterServer地址表
    TopicRouteData topicRouteData = this.namesrvController.getRouteInfoManager().pickupTopicRouteData(requestHeader.getTopic());

    // 找到主题对应的路由信息
    if (topicRouteData != null) {
        // 并且该主题为顺序信息
        if (this.namesrvController.getNamesrvConfig().isOrderMessageEnable()) {
            // 从NameServer KVConfig中获取关于顺序消息相关的配置填充路由信息
            String orderTopicConf =
                this.namesrvController.getKvConfigManager().getKVConfig(NamesrvUtil.NAMESPACE_ORDER_TOPIC_CONFIG,
                                                                        requestHeader.getTopic());
            topicRouteData.setOrderTopicConf(orderTopicConf);
        }

        byte[] content = topicRouteData.encode();
        response.setBody(content);
        response.setCode(ResponseCode.SUCCESS);
        response.setRemark(null);
        return response;
    }

    // 没找到路由信息
    response.setCode(ResponseCode.TOPIC_NOT_EXIST);
    response.setRemark("No topic route info in name server for the topic: " + requestHeader.getTopic()
                       + FAQUrl.suggestTodo(FAQUrl.APPLY_TOPIC_URL));
    return response;
}
```

为什么是定时拉取的，到时候讲到消息发送的时候再说吧。

## 总结

贴一张书中的图作为本篇总结：

![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/RocketMQ/3.NameServer%E8%B7%AF%E7%94%B1%E6%B3%A8%E5%86%8C%E3%80%81%E5%88%A0%E9%99%A4%E6%9C%BA%E5%88%B6.png)