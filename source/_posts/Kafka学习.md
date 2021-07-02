---
title: Kafka学习
date: 2021-07-02 11:43:05
tags: Kafka
categories: 消息队列
description: 公司的Mafka正是基于Kafka，因此有必要学习一波。
---

## 认识Kafka

Apache Kafka 是一个分布式发布 - 订阅消息系统和一个强大的队列。Kafka 适合离线和在线消息消费。 Kafka 消息保留在磁盘上，并在群集内复制以防止数据丢失。 Kafka 构建在 ZooKeeper 同步服务之上。 具有高性能、持久化、多副本备份、横向扩展能力。生产者往队列里写消息，消费者从队列里取消息进行业务逻辑处理。一般在架构设计中起到**解耦、削峰、异步**处理的作用。

### 相关概念

- 生产者（Producer）和消费者（Consumer）
- broker：Kafka 集群中有很多台 Server，其中每一台 Server 都可以存储消息，将每一台 Server 称为一个 kafka 实例，也叫做 broker。
- 主题（topic）：一个 topic 里保存的是同一类消息，相当于对消息的分类，每个 producer 将消息发送到 kafka 中，都需要指明要存的 topic 是哪个，也就是指明这个消息属于哪一类。
- 分区（partition）：每个 topic 都可以分成多个 partition，每个 partition 在存储层面是 append log 文件。任何发布到此 partition 的消息都会被直接追加到 log 文件的尾部。为什么要进行分区呢？最根本的原因就是：kafka基于文件进行存储，当文件内容大到一定程度时，很容易达到单个磁盘的上限，因此，采用分区的办法，一个分区对应一个文件，这样就可以将数据分别存储到不同的server上去，另外这样做也可以负载均衡，容纳更多的消费者。
- 偏移量（offset）：：一个分区对应一个磁盘上的文件，而消息在文件中的位置就称为 offset（偏移量），offset 为一个 long 型数字，它可以唯一标记一条消息。由于kafka 并没有提供其他额外的索引机制来存储 offset，文件只能顺序的读写，所以在kafka中几乎不允许对消息进行“随机读写”。

结合下图理解上述概念：

![Kafka架构图](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/Kafka/kafka-cluster-overview.png)

上图表示“zerg.hydra”的主题消息从producer发送到consumer，其被发送到broker1～broker3这三台Server上，总共被分为Partion1～Partion3三个分区。（图中P1 R3的框应为细线，P2 R3的框应为粗线）

## 基本原理

### 分布式和分区（distributed、partitioned）

消息保存在 Topic 中，而为了能够实现大数据的存储，一个 topic 划分为多个分区，每个分区对应一个文件，可以分别存储到不同的机器上，以实现分布式的集群存储。另外，每个 partition 可以有一定的副本，备份到多台机器上，以提高可用性。

总结起来就是：一个 topic 对应的多个 partition 分散存储到集群中的多个 broker 上，存储方式是一个 partition 对应一个文件，每个 broker 负责存储在自己机器上的 partition 中的消息读写。

### 副本（replicated）

kafka 还可以配置 partitions 需要备份的个数(replicas)，每个 partition 将会被备份到多台机器上，以提高可用性，备份的数量可以通过配置文件指定。

这种冗余备份的方式在分布式系统中是很常见的，那么既然有副本，就涉及到对同一个文件的多个备份如何进行管理和调度。kafka 采取的方案是：每个 partition 选举一个 server 作为“leader”，由 leader 负责所有对该分区的读写，其他 server 作为 follower 只需要简单的与 leader 同步，保持跟进即可。如果原来的 leader 失效，会重新选举由其他的 follower 来成为新的 leader。

至于如何选取 leader，这正是 Zookeeper 所擅长的，Kafka 使用 ZK 在 Broker 中选出一个 Controller，用于 Partition 分配和 Leader 选举。

另外，这里我们可以看到，实际上作为 leader 的 server 承担了该分区所有的读写请求，因此其压力是比较大的，从整体考虑，从多少个 partition 就意味着会有多少个leader，kafka 会将 leader 分散到不同的 broker 上，确保整体的负载均衡。

### 整体数据流程

Kafka 的总体数据流满足下图，该图可以说是概括了整个 Kafka 的基本原理。

![Kafka基本原理](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/Kafka/kafka%E5%9F%BA%E6%9C%AC%E5%8E%9F%E7%90%86.jpg)

- 数据生产过程（Produce）

  对于生产者要写入的一条记录，可以指定四个参数：分别是 topic、partition、key 和 value，其中 topic 和 value（要写入的数据）是必须要指定的，而 key 和 partition 是可选的。

  对于一条记录，先对其进行序列化，然后按照 Topic 和 Partition，放进对应的发送队列中。如果 Partition 没填，那么情况会是这样的：a、Key 有填。按照 Key 进行哈希，相同 Key 去一个 Partition。b、Key 没填。Round-Robin 来选 Partition。

  ![Kafka数据流](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/Kafka/kafka%20data%20flow.png)

  producer 将会和Topic下所有 partition leader 保持 socket 连接，消息由 producer 直接通过 socket 发送到 broker。其中 partition leader 的位置( host : port )注册在 zookeeper 中，producer 作为 zookeeper client，已经注册了 watch 用来监听 partition leader 的变更事件，因此，可以准确的知道谁是当前的 leader。

  producer 端采用异步发送：将多条消息暂且在客户端 buffer 起来，并将他们批量的发送到 broker，小数据 IO 太多，会拖慢整体的网络延迟，批量延迟发送事实上提升了网络效率。

- 数据消费过程（Consumer）

  对于消费者，不是以单独的形式存在的，每一个消费者属于一个 consumer group，一个 group 包含多个 consumer。特别需要注意的是：订阅 Topic 是以一个消费组来订阅的，发送到 Topic 的消息，只会被订阅此 Topic 的每个 group 中的一个 consumer 消费。

  如果所有的 Consumer 都具有相同的 group，那么就像是一个点对点的消息系统；如果每个 consumer 都具有不同的 group，那么消息会广播给所有的消费者。

  具体说来，这实际上是根据 partition 来分的，**一个 Partition 只能被消费组里的一个消费者消费，但是可以同时被多个消费组消费**，消费组里的每个消费者是关联到一个 partition 的，因此有这样的说法：对于一个 topic,同一个 group 中不能有多于 partitions 个数的 consumer 同时消费,否则将意味着某些 consumer 将无法得到消息。

- 消息传送机制

  Kafka 支持 3 种消息投递语义,在业务中，常常都是使用 At least once 的模型。

  - At most once：最多一次，消息可能会丢失，但不会重复。
  - At least once：最少一次，消息不会丢失，可能会重复。
  - Exactly once：只且一次，消息不丢失不重复，只且消费一次。

## Zookeeper作用

Apache Kafka 的一个关键依赖是 Apache Zookeeper，它是一个分布式配置和同步服务。Zookeeper 是 Kafka 代理和消费者之间的协调接口。Kafka 服务器通过 Zookeeper 集群共享信息。Kafka 在 Zookeeper 中存储基本元数据，例如关于主题，代理，消费者偏移(队列读取器)等的信息。

由于所有关键信息存储在 Zookeeper 中，并且它通常在其整体上复制此数据，因此Kafka代理/ Zookeeper 的故障不会影响 Kafka 集群的状态。Kafka 将恢复状态，一旦 Zookeeper 重新启动。 这为Kafka带来了零停机时间。Kafka 代理之间的领导者选举也通过使用 Zookeeper 在领导者失败的情况下完成。

## Kafka基本操作

安装完成后按以下顺序操作（在 Kafka 安装目录下，`/usr/local/Cellar/kafka/2.8.0/libexec`）：

1. 启动 Zookeeper

   ```bash
   bin/zookeeper-server-start.sh config/zookeeper.properties
   ```

2. 开一个新的终端，启动 Kafka broker

   ```bash
   bin/kafka-server-start.sh config/server.properties
   ```

3. 开一个新的终端，创建 Topic

   ```bash
   bin/kafka-topics.sh --create --topic quickstart-events --bootstrap-server localhost:9092
   ```

4. 往 Topic 中写入消息（Producer）

   ```bash
   bin/kafka-console-producer.sh --topic quickstart-events --bootstrap-server localhost:9092
   > This is my first event
   > This is my second event
   ```

5. 开一个新的终端，读取消息（Consumer）

   ```bash
   bin/kafka-console-consumer.sh --topic quickstart-events --from-beginning --bootstrap-server localhost:9092
   ```

可以自己编写 Producer 和 Consumer，并启动 zk 和 Kafka，进行试验。

## 参考资料

- [Apache Kafka 教程](https://www.w3cschool.cn/apache_kafka/apache_kafka_introduction.html)
- [Kafka官网quickstart](https://kafka.apache.org/quickstart)
