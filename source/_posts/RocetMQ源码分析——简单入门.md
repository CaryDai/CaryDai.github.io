---
title: RocketMQ源码分析——简单入门
date: 2020-07-09 10:37:03
tags: RocketMQ
categories: RocketMQ
description: 简单介绍RocketMQ相关概念并构建源码阅读环境。
---

## 消息队列简介

消息队列是一种应用于分布式应用的各个系统之间，作为消息的中转站，具备**异步**、**削峰**和**解耦**等特性的消息中间件系统。

- 异步体现在消息队列能将用户的请求立即返回，提高响应时间。
- 削峰体现在请求达到峰值时，把这些请求存储起来让服务消费端从消息队列中取，而不是将大量请求直接发到应用服务器或数据库上；
- 解耦体现在消息队列提供的“主题订阅”机制。如果没有消息队列，那么每增加一个新的业务，都需要在服务生产系统中编写相应的调用接口。而有了消息队列，服务生产者可以将生产出来的消息添加到消息队列中，服务消费者只需要订阅它想要的主题消息就可以了。

当然，在系统中引入消息队列也会使得系统可用性降低、复杂性提高。常见问题有：

- 如何保证消息队列的高可用
- 如何保证消息没有被重复消费
- 如何保证消息是被顺序消费的
- 消息丢失了怎么办
- ......

常用的消息队列有：RocketMQ、Kafka、RabbitMQ、ActiveMQ等：

1. 前两者是百万级的吞吐量，后两者是万级的吞吐量；
2. 前两者基于分布式架构，后两者基于主从架构；
3. Kafka 一般用于大数据领域的实时计算以及日志采集；
4. RabbitMQ 基于 erlang 开发，性能极好。

## RocketMQ 架构

[RocketMQ 源码包](https://github.com/apache/rocketmq)中有个 doc 文件夹，里面有中英文说明文档，为了后面的源码阅读，可以`git clone`下来。下图来自官方文档。

![](https://github.com/apache/rocketmq/raw/master/docs/cn/image/rocketmq_architecture_1.png)

从[官方文档](https://github.com/apache/rocketmq/blob/master/docs/cn/architecture.md)中可以得知，RocketMQ 在架构上主要分为四部分：

- **Producer**：消息发布者。Producer 通过 MQ 的负载均衡模块选择相应的 Broker 集群队列进行消息投递，投递的过程支持快速失败并且低延迟。
- **Consumer**：消息消费者。支持以 push 推，pull 拉两种模式对消息进行消费。同时也支持集群方式（每条消息只需要被处理一次）和广播方式（每条消息都会被每个Consumer消费）的消费，它提供实时消息订阅机制，可以满足大多数用户的需求。
- **NameServer**：Topic 路由注册中心。其角色类似 Dubbo 中的 zookeeper，支持 Broker 的动态注册与发现。主要包括两个功能：1. Broker 管理，NameServer 接受 Broker 集群的注册信息并且保存下来作为路由信息的基本数据。然后提供心跳检测机制，检查 Broker 是否还存活；2. 路由信息管理，每个 NameServer 将保存关于 Broker集群的整个路由信息和用于客户端查询的队列信息。然后 Producer 和 Consumer 通过 NameServer 就可以知道整个 Broker 集群的路由信息，从而进行消息的投递和消费。NameServer通常也是集群的方式部署，<u>各实例间相互不进行信息通讯</u>。<u>Broker 是向每一台 NameServer 注册自己的路由信息，所以每一个 NameServer 实例上面都保存一份完整的路由信息。</u>当某个 NameServer 因某种原因下线了，Broker 仍然可以向其它 NameServer 同步其路由信息，Producer,Consumer 仍然可以动态感知 Broker 的路由的信息。
- **Broker**：Broker 主要负责消息的存储、投递和查询以及服务高可用保证。

## 核心概念

### Message

RocketMQ 消息，消息发送或消费时必须指明 topic。Message 中有一个可选的 tag 用于过滤消息。

### Topic

主题信息，消息发送或消费都是根据 topic 进行，一个 topic 一般分布在多个 broker 上，每个 topic 包含多个队列。一个 Broker 为每一 Topic 默认创建4个读队列4个写队列。

### Queue

1个 Topic 包含多个 Queue，Message 实际存储在 Queue 中。每个 Consumer 消费时只会对应一个 Queue 进行消费，也就是说，即使有5个 Consumer，如果 Queue 只有4个，那多出来的那个 Consumer 不会进行消息消费。

### Tag

Tag 是 Topic 的一部分，消息方消费的时候可以根据 tag 进行过滤，选择消费。

## 源码构建

RocketMQ 的部署和使用直接看[官网介绍](http://rocketmq.apache.org/docs/quick-start/)就行。

RocketMQ 的源码构建过程比较简单（相比 Spring），这里不过多介绍，《RocketMQ技术内幕》中写得挺详细。尽量不要选择最新的版本，因为可能会出现意想不到的问题。

可以通过`git show`查看当前分支的版本号，`git tag`查看所有版本号，为了检出特定版本，可以执行`git checkout 版本号`命令。

构建完成后先运行一个 demo。`example`包下有一个`quickstart`包，里面有`Consumer`和`Producer`的简单示例。因为在本地运行，所以把这两个文件下的`setNamesrvAddr`地址改为本地地址。随后运行`NamesrvStartup.main()`和`BrokerStartup.main()`启动`NameServer`和`Broker`，`Broker`启动时没有出现任何信息说明启动正常，如果出现异常，可以查看日志文件找出异常原因。

下一篇开始进行源码分析，先分析`NameServer`。

