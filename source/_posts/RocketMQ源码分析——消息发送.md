---
title: RocketMQ源码分析——消息发送
date: 2020-07-12 16:31:05
tags: RocketMQ
categories: 消息队列
description: 分析Producer消息发送流程
---

## 消息发送方式

RocketMQ 支持3种消息发送方式：同步（sync）、异步（async）、单向（oneway）。

- 同步：发送者向 MQ 执行发送消息 API 时，同步等待，直到消息服务器返回发送结果。
- 异步：发送者向 MQ 执行发送消息 API 时，指定消息发送成功后的回掉函数，然后调用消息发送 API 后，立即返回，消息发送者线程不阻塞，直到运行结束，消息发送成功或失败的回调任务在一个新的线程中执行。
- 单向：消息发送者向 MQ 执行发送消息 API 时，直接返回，不等待消息服务器的结果，也不注册回调函数，简单地说，就是只管发，不在乎消息是否成功存储在消息服务器上。

## RocketMQ消息

RocketMQ 消息封装类：`org\apache\rocketmq\common\message\Message`

```java
public class Message implements Serializable {
    private static final long serialVersionUID = 8445773977080406428L;

    // 消息所属主题
    private String topic;
    // 消息Flag，给用户处理
    private int flag;
    // 扩展属性
    private Map<String, String> properties;
    // 消息体
    private byte[] body;
    private String transactionId;
    ...
}
```

其中扩展属性又包括下面几个：

- `tag` ：消息TAG ，用于消息过滤。
- `keys`：Message 索引键，多个用空格隔开，RocketMQ 可以根据这些 key 快速检索到消息。
- `waitStoreMsgOK`：消息发送时是否等消息存储完成后再返回。
- `delayTimeLevel`：消息延迟级别，用于定时消息或消息重试。

这些扩展属性存储在 Message 的 properties 中。

## 消息发送流程

接着上一篇的代码继续，现在走到这里了：

```java
Message msg = new Message("TopicTest" /* Topic */,
                          "TagA" /* Tag */,
                          ("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET) /* Message body */);

/*
 * Call send message to deliver message to one of brokers.
 */
SendResult sendResult = producer.send(msg);
```

消息发送流程主要的步骤：验证消息、查找路由、消息发送（包含异常处理机制）。

### 验证消息

进入`send(msg)`方法：

```java
public SendResult send(
    Message msg) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
    // 验证消息
    Validators.checkMessage(msg, this);
    // 设置消息对应的topic
    msg.setTopic(withNamespace(msg.getTopic()));
    return this.defaultMQProducerImpl.send(msg);
}
```

默认以同步方式发送消息。继续进去，最后会到`DefaultMQProducerImpl#sendDefaultImpl`方法，里面会先验证生产者状态，并且再次验证消息：

```java
// 消息发送之前，首先确保生产者处于运行状态
this.makeSureStateOK();
// 验证消息是否符合规范，具体的规范要求是主题名称、消息体不能为空、消息长度不能等于0且默认不能超过允许发送消息的最大长度4M
Validators.checkMessage(msg, this.defaultMQProducer);
```

### 查找路由

接下里需要查找哪些 Broker 节点存有当前 topic 对应的消息：

```java
// 查找主题的路由信息
TopicPublishInfo topicPublishInfo = this.tryToFindTopicPublishInfo(msg.getTopic());
```

```java
private TopicPublishInfo tryToFindTopicPublishInfo(final String topic) {
    // 从缓存中取当前topic对应的路由信息
    TopicPublishInfo topicPublishInfo = this.topicPublishInfoTable.get(topic);
    // 如果没有缓存或没有包含消息队列
    if (null == topicPublishInfo || !topicPublishInfo.ok()) {
        this.topicPublishInfoTable.putIfAbsent(topic, new TopicPublishInfo());
        // 向NameServer查询该topic的路由信息，没找到则抛出异常
        this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic);
        topicPublishInfo = this.topicPublishInfoTable.get(topic);
    }

    // 如果生产者中缓存了topic的路由信息，或者该路由信息中包含了消息队列，则直接返回该路由信息
    if (topicPublishInfo.isHaveTopicRouterInfo() || topicPublishInfo.ok()) {
        return topicPublishInfo;
    } else {
        // 如果路由信息未查到，再次尝试用默认主题 DefaultMQProducerImpl#createTopicKey 去查询，
        // 如果BrokerConfig#autoCreateTopicEnable为true时，NameServer将返回路由信息，如果
        // autoCreateTopicEnable为false将抛出无法找到topic路由异常
        this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic, true, this.defaultMQProducer);
        topicPublishInfo = this.topicPublishInfoTable.get(topic);
        return topicPublishInfo;
    }
}
```

这边`mQClientFactory`通过负载均衡服务选择一个 Broker。

返回的路由信息：

![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/RocketMQ/2.%E4%BB%8ENameServer%E4%B8%AD%E6%9F%A5%E5%88%B0%E8%B7%AF%E7%94%B1%E4%BF%A1%E6%81%AF.png)

`queueData`保存了当前 Topic（TopicTest） 在`broker-a`这个 Broker 中的队列信息，一个 Broker 为每一 Topic 默认创建4个读队列4个写队列。

`brokerDatas`则保存 Broker 的地址信息，因为我本地只有一台 Broker ——`broker-a`，所以这里只显示当前`broker-a`。

### 消息发送

现在还是在`DefaultMQProducerImpl#sendDefaultImpl`方法里。现在已经知道路由信息了，接下来就需要选择一个 MessageQueue 了。

先选择消息队列：

```java
// 上次获取的BrokerName
String lastBrokerName = null == mq ? null : mq.getBrokerName();
// 选择当前Broker中相应topic的一个queue，然后往这个queue里发消息。
MessageQueue mqSelected = this.selectOneMessageQueue(topicPublishInfo, lastBrokerName);
```

这边就不深入下去了，这里还涉及到是否启用 Broker 故障延迟机制，默认不启用。

当前选择的消息队列为：

![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/RocketMQ/4.%E5%BD%93%E5%89%8D%E9%80%89%E6%8B%A9%E7%9A%84%E6%B6%88%E6%81%AF%E9%98%9F%E5%88%97.png)

可以参见《RocketMQ技术内幕》`3.4.3`节内容。

到目前为止，我们知道了 Broker 的地址，也知道了往哪个消息队列中发，这样就可以准备发送了。

消息发送核心入口：

```java
sendResult = this.sendKernelImpl(msg, mq, communicationMode, sendCallback, topicPublishInfo, timeout - costTime);
```

```java
private SendResult sendKernelImpl(final Message msg,
                                      final MessageQueue mq,
                                      final CommunicationMode communicationMode,
                                      final SendCallback sendCallback,
                                      final TopicPublishInfo topicPublishInfo,
                                      final long timeout) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
    long beginStartTime = System.currentTimeMillis();
    // 1.根据MessageQueue获取Broker的网络地址
    String brokerAddr = this.mQClientFactory.findBrokerAddressInPublish(mq.getBrokerName());
    // 如果MQClientInstance的brokerAddrTable未缓存该Broker的信息
    if (null == brokerAddr) {
        // 从NameServer查找主题的路由信息
        tryToFindTopicPublishInfo(mq.getTopic());
        // 从缓存中取得存有当前Topic的Broker，默认取得主节点
        brokerAddr = this.mQClientFactory.findBrokerAddressInPublish(mq.getBrokerName());
    }

    SendMessageContext context = null;
    if (brokerAddr != null) {
        brokerAddr = MixAll.brokerVIPChannel(this.defaultMQProducer.isSendMessageWithVIPChannel(), brokerAddr);

        byte[] prevBody = msg.getBody();
        try {
            //for MessageBatch,ID has been set in the generating process
            if (!(msg instanceof MessageBatch)) {
                // 2.为消息分配全局唯一ID
                MessageClientIDSetter.setUniqID(msg);
            }

            boolean topicWithNamespace = false;
            if (null != this.mQClientFactory.getClientConfig().getNamespace()) {
                msg.setInstanceId(this.mQClientFactory.getClientConfig().getNamespace());
                topicWithNamespace = true;
            }

            int sysFlag = 0;
            boolean msgBodyCompressed = false;
            // 如果消息体默认超过4K(compressMsgBodyOverHowmuch)，会对消息体采用zip压缩
            if (this.tryToCompressMessage(msg)) {
                // 并设置消息的系统标记为COMPRESSED_FLAG
                sysFlag |= MessageSysFlag.COMPRESSED_FLAG;
                msgBodyCompressed = true;
            }

            final String tranMsg = msg.getProperty(MessageConst.PROPERTY_TRANSACTION_PREPARED);
            // 如果是事务Prepared消息
            if (tranMsg != null && Boolean.parseBoolean(tranMsg)) {
                // 设置消息的系统标记为TRANSACTION_PREPARED_TYPE
                sysFlag |= MessageSysFlag.TRANSACTION_PREPARED_TYPE;
            }

            ...

            // 3.如果注册了消息发送钩子函数，则执行消息发送之前的增强逻辑
                if (this.hasSendMessageHook()) {
                    context = new SendMessageContext();
                    context.setProducer(this);
                    context.setProducerGroup(this.defaultMQProducer.getProducerGroup());
                    context.setCommunicationMode(communicationMode);
                    context.setBornHost(this.defaultMQProducer.getClientIP());
                    context.setBrokerAddr(brokerAddr);
                    context.setMessage(msg);
                    context.setMq(mq);
                    context.setNamespace(this.defaultMQProducer.getNamespace());
                    String isTrans = msg.getProperty(MessageConst.PROPERTY_TRANSACTION_PREPARED);
                    if (isTrans != null && isTrans.equals("true")) {
                        context.setMsgType(MessageType.Trans_Msg_Half);
                    }

                    if (msg.getProperty("__STARTDELIVERTIME") != null || msg.getProperty(MessageConst.PROPERTY_DELAY_TIME_LEVEL) != null) {
                        context.setMsgType(MessageType.Delay_Msg);
                    }
                    this.executeSendMessageHookBefore(context);
                }    
                
            // 4.构建消息发送请求包
            SendMessageRequestHeader requestHeader = new SendMessageRequestHeader();
            // 设置生产者组
            requestHeader.setProducerGroup(this.defaultMQProducer.getProducerGroup());
            // 设置主题名称
            requestHeader.setTopic(msg.getTopic());
            // 设置默认创建主题Key
            requestHeader.setDefaultTopic(this.defaultMQProducer.getCreateTopicKey());
            // 该主题在单个Broker默认队列数
            requestHeader.setDefaultTopicQueueNums(this.defaultMQProducer.getDefaultTopicQueueNums());
            // 设置队列ID（队列序号）
            requestHeader.setQueueId(mq.getQueueId());
            // 设置消息系统标记（MessageSysFlag）
            requestHeader.setSysFlag(sysFlag);
            // 设置消息发送时间
            requestHeader.setBornTimestamp(System.currentTimeMillis());
            // 设置消息标记（RocketMQ对消息中的flag不做任何处理，供应用程序使用）
            requestHeader.setFlag(msg.getFlag());
            // 设置消息扩展属性
            requestHeader.setProperties(MessageDecoder.messageProperties2String(msg.getProperties()));
            // 设置消息重试次数
            requestHeader.setReconsumeTimes(0);
            requestHeader.setUnitMode(this.isUnitMode());
            // 是否为批量消息
            requestHeader.setBatch(msg instanceof MessageBatch);
            if (requestHeader.getTopic().startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
                String reconsumeTimes = MessageAccessor.getReconsumeTime(msg);
                if (reconsumeTimes != null) {
                    requestHeader.setReconsumeTimes(Integer.valueOf(reconsumeTimes));
                    MessageAccessor.clearProperty(msg, MessageConst.PROPERTY_RECONSUME_TIME);
                }

                String maxReconsumeTimes = MessageAccessor.getMaxReconsumeTimes(msg);
                if (maxReconsumeTimes != null) {
                    requestHeader.setMaxReconsumeTimes(Integer.valueOf(maxReconsumeTimes));
                    MessageAccessor.clearProperty(msg, MessageConst.PROPERTY_MAX_RECONSUME_TIMES);
                }
            }

            SendResult sendResult = null;
            // 根据不同的消息发送模式，发送消息
            switch (communicationMode) {
                case ASYNC:
                    ...
                case ONEWAY:
                case SYNC:
                    long costTimeSync = System.currentTimeMillis() - beginStartTime;
                    if (timeout < costTimeSync) {
                        throw new RemotingTooMuchRequestException("sendKernelImpl call timeout");
                    }
                    sendResult = this.mQClientFactory.getMQClientAPIImpl().sendMessage(
                        brokerAddr,
                        mq.getBrokerName(),
                        msg,
                        requestHeader,
                        timeout - costTimeSync,
                        communicationMode,
                        context,
                        this);
                    break;
                default:
                    assert false;
                    break;
            }

            // 如果注册了消息发送钩子函数，执行after逻辑。注意，就算消息发送过程中发生RemotingException、
            // MQBrokerException、InterruptedException时该方法也会执行
            if (this.hasSendMessageHook()) {
                context.setSendResult(sendResult);
                this.executeSendMessageHookAfter(context);
            }

            return sendResult;
        ...

    // 找不到Broker信息，抛出MQClientException，提示Broker不存在
    throw new MQClientException("The broker[" + mq.getBrokerName() + "] not exist", null);
}
```

首先从缓存表中获取 Topic 所在 Broker 的网络地址，接着进行消息发送前的预处理操作，包括分配消息id、是否需要压缩、是否需要注册钩子函数等。最后封装请求头，根据不同的消息发送模式发送消息。

封装的请求头如下：

![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/RocketMQ/5.%E6%B6%88%E6%81%AF%E5%8F%91%E9%80%81%E8%AF%B7%E6%B1%82%E5%A4%B4.png)

## 总结

至此，消息发送流程基本介绍完了，虽然有一些细节没有提及，但是大致流程已经清晰了。

