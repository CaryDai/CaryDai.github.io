---
title: Redis入门介绍与数据类型
date: 2019-08-25 10:26:41
categories: Redis
tags: Redis
---

## NoSQL入门和概述

### NoSQL数据类型简介

以一个电商客户、订单、订购、地址模型来对比下**关系型数据库**和**非关系型数据库**。

对于关系型数据库，我们至少需要客户表`customer`，订单表`order_detail/order_item`，订购信息表`order_payment`，收获地址表`order_address`。甚至可能需要库存表（相当于购物车）。客户表和订单表、地址表之间的关系是一对多。随着功能的增加，表的关系会越来越复杂，对表进行`crud`操作会非常麻烦。常常会涉及到关联查询(写Join语句)。

BSON：是一种类json的二进制形式的存储格式，简称Binary JSON，它和JSON一样，支持内嵌的文档对象和数组对象。

高并发的操作是不太建议有关联查询的，互联网公司用冗余数据来避免关联查询。分布式事务是支持不了太多的并发的。在NoSQL中只需要根据键值对来查询即可。

#### 聚合类型

- **KV键值对**
- **BSON**
- 列族
  按列存储数据。
- 图形
  对于复杂的关系用图来描述。

### NoSQL数据库的四大分类

- KV键值
  新浪：BerkeleyDB + redis
  美团：redis + tair
  阿里、百度：memcache + redis
- 文档型数据库（bson格式比较多）
  CouchDB
  **MongoDB**：是一个**基于分布式文件存储的数据库**。由C++编写，旨在为 WEB 应用提供可扩展的高性能数据存储解决方案。是一个介于关系数据库和非关系数据库之间的产品。
- 列存储数据库
  Cassandra，HBase。分布式文件系统。
- 图数据库
  它不是放图形的，放的是关系比如：朋友圈社交网络、广告推荐系统。专注于构建图谱，如Neo4J，InfoGrid。
- 四者对比

### 在分布式数据库中CAP原理CAP + BASE

- 传统的**ACID**是什么？
  A(Atomicity)原子性、C(Consistency)一致性、I(Isolation)独立性、D(Durability)持久性。

- **CAP**

  - C(Consistency)：强一致性
  - A(Availability)：高可用性
  - P(Partition tolerance)：分区容错性

- **CAP的3进2**
  CAP理论的核心是：一个分布式系统不可能同时很好地满足**一致性**，**可用性**和**分区容错性**这三个需求，最多只能同时较好地满足两个。
  因此，根据CAP原理将`NoSQL`数据库分成了`满足CA原则`、`满足CP原则`、`满足AP原则`三大类：

  - CA - 单点集群，满足一致性，可用性的系统，通常在可扩展性上不太强大。传统的Oracle数据库。
  - CP - 满足一致性，分区容错性的系统，通常性能不是特别高。`Redis`、`MongoDB`
  - **AP** - 满足可用性，分区容错性的系统，通常可能对一致性要求低一些。**大多数网站架构的选择。**

  ![6.CAP.png](http://ww1.sinaimg.cn/large/005Mjp9lly1g6s2pqsd9sj30in0g2myg.jpg)

  由于当前网络硬件肯定会出现延迟丢包等问题，所以**分区容忍性(P)**是我们必须要实现的。
  对于`A(高可用性)`和`C(强一致性)`的选择可以看这样一个例子：比如双11的时候有一款LV的包热卖，那么我们需要在这款包上面展示*浏览数、收藏数、评价数*等信息，而且当时的浏览量肯定很大，高并发的环境。这时就需要高可用性了，网站不能崩；而那些*浏览数、收藏数*等信息的精确程度对用户来说不是特别关键，只需保证若一致性即可。

  **数据库事务一致性需求**：很多web实时系统并不要求严格的数据库事务，对一致性的要求很低，有些场合对写一致性要求并不高。允许实现最终一致性。

  **数据库的写实时性和读实时性需求**：对关系数据库来说，插入一条数据后立刻查询，是肯定可以读出来这条数据的；但是对于很多web应用来说，并不要求这么高的实时性。比方说发一条消息之后，过几秒乃至十几秒之后，我的订阅者才看得到这条消息是完全可以接受的。

  **对复杂的SQL查询，特别是多表关联查询的需求**：任何大数据量的web系统，都非常忌讳多个大表的关联查询，以及复杂的数据分析类型的报表查询。

- **BASE**
  BASE就是为了解决关系数据库强一致性引起的问题而导致的可用性降低而提出的解决方案。BASE其实是下面三个术语的缩写：

  - 基本可用（Basically Available）
  - 软状态（Soft state）
  - 最终一致（Eventually consistent）

  它的思想是通过让系统放松对某一时刻数据一致性的要求来换取系统整体伸缩性和性能上改观。为什么这样说呢，缘由就在于大型系统往往由于地狱分布和极高性能的要求，不可能采用分布式事务来完成这些指标，要想获得这些指标，我们必须采用另一种方式来完成。BASE就是解决这个问题的方法。

- 分布式 + 集群简介
  简单来说：

  1. 分布式：不同的多台服务器上面部署不同的服务模块，他们之间通过Rpc / Rmi之间通信和调用，对外提供服务和组内协作。
  2. 集群：不同的多态服务器上面部署相同的服务模块，通过分布式调度软件进行统一的调度，对外提供服务和访问。

## Redis入门介绍

### 入门概念

#### Redis是什么

Redis：REmote DIctionary Server(远程字典服务器)。开源免费，用C语言编写，是一个高性能的（key/value）分布式的内存服务器，基于内存运行并支持持久化的 NoSQL 数据库，也被人们称为数据结构服务器。

Redis与其它 NoSQL 产品有以下三个特点：

1. Redis支持数据的持久化，可将内存中的数据保存在磁盘中，重启的时候再次加载进行使用；
2. Redis不仅支持简单的 key-value 类型的数据，同时还提供 list，set，zset，hash 等数据结构的存储；
3. Redis支持数据的备份，即 master-slave 模式的数据备份。

#### 能干嘛

- 内存存储和持久化：redis支持异步将内存中的数据写到硬盘上，同时不影响继续服务。
- 取最新N个数据的操作，如：可以将最新的10条评论的 ID 放在 Redis 的 List 集合里面。
- 模拟类似与 HttpSession 这种需要设定过期时间的功能。
- 发布、订阅消息系统。
- 定时器、计数器。

#### 下载与安装

Redis的下载与安装可以看中文版的官网：https://www.redis.net.cn/download/

安装完成后其实在`/usr/local/bin`下，`cd`到这个目录后，可以使用命令`ls -l`进行查看。为启动 Redis 服务器，可以在`bin`目录下直接敲`redis-server`，默认的端口是**6379**；同时启动另一个终端，`cd`到相同的目录下，输入`redis-cli`即可完成客户端的启动。
![7.redis服务端的启动.png](http://ww1.sinaimg.cn/large/005Mjp9lly1g6s2q60r31j30k20cnq49.jpg)

![](D:\Learning\Redis\截图\8.启动redis客户端进行测试.png)

当然上述的启动操作也可以在`/opt/redis-5.0.4/src`（这是我的安装目录）下进行。
用命令查看后台的Redis服务有没有起来，可以用以下命令：

```bash
ps -ef|grep redis
```

默认情况下，Redis 并不会以一个守护进程的方式在后台运行，可以通过修改配置文件的方式更改其默认的行为。但是首先需要备份一份配置文件。根据课程的做法，我在`/home/dqj`下创建一个新的文件夹`myredis`用来存放`redis`的备份文件。随后用`vim`工具进行修改：

```bash
root@ubuntu:/usr/local/bin# mkdir /home/dqj/myredis
root@ubuntu:/usr/local/bin# cd /opt/redis-5.0.4/
root@ubuntu:/opt/redis-5.0.4# ls -l
total 260
-rw-rw-r--  1 root root 99445 Mar 18 09:21 00-RELEASENOTES
-rw-rw-r--  1 root root    53 Mar 18 09:21 BUGS
-rw-rw-r--  1 root root  1894 Mar 18 09:21 CONTRIBUTING
-rw-rw-r--  1 root root  1487 Mar 18 09:21 COPYING
drwxrwxr-x  6 root root  4096 Jul 16 07:09 deps
-rw-r--r--  1 root root    92 Jul 16 07:20 dump.rdb
-rw-rw-r--  1 root root    11 Mar 18 09:21 INSTALL
-rw-rw-r--  1 root root   151 Mar 18 09:21 Makefile
-rw-rw-r--  1 root root  4223 Mar 18 09:21 MANIFESTO
-rw-rw-r--  1 root root 20555 Mar 18 09:21 README.md
-rw-rw-r--  1 root root 62155 Mar 18 09:21 redis.conf
-rwxrwxr-x  1 root root   275 Mar 18 09:21 runtest
-rwxrwxr-x  1 root root   280 Mar 18 09:21 runtest-cluster
-rwxrwxr-x  1 root root   281 Mar 18 09:21 runtest-sentinel
-rw-rw-r--  1 root root  9710 Mar 18 09:21 sentinel.conf
drwxrwxr-x  3 root root  4096 Jul 16 07:18 src
drwxrwxr-x 10 root root  4096 Mar 18 09:21 tests
drwxrwxr-x  8 root root  4096 Mar 18 09:21 utils
root@ubuntu:/opt/redis-5.0.4# cp redis.conf /home/dqj/myredis/
root@ubuntu:/opt/redis-5.0.4# vim /home/dqj/myredis/redis.conf
```

![9.修改为默认启动成后台进程.png](http://ww1.sinaimg.cn/large/005Mjp9lly1g6s2r02pg2j30jz032dfw.jpg)

如果想按照这个配置文件的方式启动，则需要这样写（注意在`/usr/local/bin`目录下）：

```bash
root@ubuntu:/usr/local/bin# redis-server /home/dqj/myredis/redis.conf
```

这时可以不需要启动另一个终端来启动客户端，可以直接在当前终端下写，因为Redis服务已经在后台启动了：

```bash
root@ubuntu:/usr/local/bin# redis-cli -p 6379
```

#### Redis的一些基础知识

- `Redis`是单进程运行的。以单进程模型来处理客户端的请求，对读写事件的响应。

- `Redis`默认有16个数据库，类似数组下标从0开始，初始默认使用零号库，可以使用`select`语句来进行数据库之间的切换：

  ```bash
  127.0.0.1:6379> select 7
  OK
  127.0.0.1:6379[7]> get foo
  (nil)
  127.0.0.1:6379[7]> select 0
  OK
  127.0.0.1:6379> get foo
  "bar"
  ```

- `Dbsize`查看当前数据库的`key`的数量，使用`keys *`，可以具体查看`key`的名称。

- `FLUSHDB`：清空当前库。

- `FLUSHALL`：清空所有库。

- 统一密码管理，16个库都是同样的密码，要么都OK，要么一个也连不上。

- `Redis`索引都是从0开始。

## Redis数据类型

### Redis的五大数据类型

- String（字符串）
  `string`是`redis`最基本的类型，一个`key`对应一个`value`。`string`类型是二进制安全的。意思是`redis`的`string`可以包含任何数据。比如`jpg`图片或者序列化的对象。一个`redis`中字符串`value`最多可以是`512M`。
- Hash（哈希，类似`java`里的`Map`）
  `Redis hash`是一个键值对集合，它是一个`string`类型的`field`和`value`的映射表，`hash`特别适合用于存储对象。类似`Java`里面的`Map<String,Object>`。
- List（列表）
  `Redis`列表是简单的字符串列表，按照插入顺序排序。你可以添加一个元素到列表的头部（左边）或者尾部（右边）。它的底层实际是个**链表**。
- Set（集合）
  `Redis`的`Set`是`string`类型的无序无重复集合，它是通过`HashTable`实现的。
- Zset（sorted set：有序集合）
  `Redis Zset`和`Set`一样也是`string`类型元素的集合，且不允许重复的成员。不同的是`Zset`每个元素都会关联一个`double`类型的分数，`redis`正是通过分数来为集合中的成员进行从小到大的排序。`zset`的成员是唯一的，但分数（score）却可以重复。

哪里去获得`redis`常见的数据类型操作命令?            —— Http://redisdoc.com/

### Redis键（key）

常用`key`命令：
![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/11.key%E5%B8%B8%E7%94%A8%E5%91%BD%E4%BB%A4.png)

- `keys *`：查看当前库中的`key`
- `exist key的名字`：判断某个`key`是否存在。
- `move key db`：把当前库中的`key`移到指定的库中。
  ![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/10.%E6%BC%94%E7%A4%BAmove%E6%8C%87%E4%BB%A4.png)
- `expire key 秒钟`：为给定的`key`设置过期时间。
- `ttl key`：查看还有多少秒过期，`-1`表示永不过期，`-2`表示已经过期。
- `type key`：查看你的`key`是什么类型。

### Redis字符串（String）

常用`String`命令：
![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/12.string%E5%B8%B8%E7%94%A8%E5%91%BD%E4%BB%A4.png)

#### 案例

- set / get / del / append / strlen

  ```bash
  127.0.0.1:6379> set k1 v1
  OK
  127.0.0.1:6379> set k2 v2
  OK
  127.0.0.1:6379> keys *
  1) "k2"
  2) "k1"
  127.0.0.1:6379> del k2
  (integer) 1
  127.0.0.1:6379> keys *
  1) "k1"
  127.0.0.1:6379> APPEND k1 abc
  (integer) 5
  127.0.0.1:6379> get k1
  "v1abc"
  127.0.0.1:6379> STRLEN k1
  (integer) 5
  ```

- Incr / decr / incrby / decrby：一定要是数字才能进行加减。

  ```bash
  127.0.0.1:6379> set k1 1
  OK
  127.0.0.1:6379> INCR k1
  (integer) 2
  127.0.0.1:6379> get k1
  "2"
  127.0.0.1:6379> INCR k1
  (integer) 3
  127.0.0.1:6379> INCR k1
  (integer) 4
  127.0.0.1:6379> INCR k1
  (integer) 5
  127.0.0.1:6379> get k1
  "5"
  127.0.0.1:6379> DECR k1
  (integer) 4
  127.0.0.1:6379> DECR k1
  (integer) 3
  127.0.0.1:6379> DECR k1
  (integer) 2
  127.0.0.1:6379> INCRBY k1 3
  (integer) 5
  127.0.0.1:6379> INCRBY k1 3
  (integer) 8
  127.0.0.1:6379> DECRBY k1 2
  (integer) 6
  127.0.0.1:6379> DECRBY k1 2
  (integer) 4
  ```

- getrange：获取指定区间范围内的值，类似`between...and`的关系，从零到负一表示全部。

  ```bash
  127.0.0.1:6379> GETRANGE k1 0 -1
  "v1123456"
  127.0.0.1:6379> GETRANGE k1 0 3
  "v112"
  ```

  setrange：设置指定区间范围内的值，格式是`setrange key 位置 值`。

  ```bash
  127.0.0.1:6379> SETRANGE k1 0 abc
  (integer) 8
  127.0.0.1:6379> get k1
  "abc23456"
  ```

- setex(set with expire) 键 秒值 / setnx(set if not exist)

  ```bash
  127.0.0.1:6379> setex k2 10 v2
  OK
  127.0.0.1:6379> ttl k2		'查看还有多少存活时间'
  (integer) 7
  127.0.0.1:6379> get k2
  "v2"
  127.0.0.1:6379> ttl k2
  (integer) -2
  127.0.0.1:6379> get k2
  (nil)
  
  127.0.0.1:6379> setnx k1 v11
  (integer) 0
  127.0.0.1:6379> get k1
  "4"
  127.0.0.1:6379> setnx k11 v11
  (integer) 1
  127.0.0.1:6379> get k11
  "v11"
  ```

- mset / mget / msetnx

  ```bash
  127.0.0.1:6379> MSET k1 v1 k2 v2 k3 v3
  OK
  127.0.0.1:6379> MGET
  (error) ERR wrong number of arguments for 'mget' command
  127.0.0.1:6379> MGET k1 k2 k3
  1) "v1"
  2) "v2"
  3) "v3"
  127.0.0.1:6379> MSETNX k3 v3 k4 v4
  (integer) 0
  127.0.0.1:6379> GET k4
  (nil)
  127.0.0.1:6379> MSETNX k4 v4 k5 v5
  (integer) 1
  127.0.0.1:6379> MGET k4 k5
  1) "v4"
  2) "v5"
  ```

### Redis列表（List）

常用`List`命令：
![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/13.%E5%B8%B8%E7%94%A8List%E5%91%BD%E4%BB%A4.png)

#### 案例

- lpush / rpush / lrange

  ```bash
  127.0.0.1:6379> LPUSH list01 1 2 3 4 5
  (integer) 5
  127.0.0.1:6379> LRANGE list01 0 -1
  1) "5"
  2) "4"
  3) "3"
  4) "2"
  5) "1"
  127.0.0.1:6379> RPUSH list02 1 2 3 4 5
  (integer) 5
  127.0.0.1:6379> LRANGE list02 0 -1
  1) "1"
  2) "2"
  3) "3"
  4) "4"
  5) "5"
  ```

- lpop / rpop

  ```bash
  127.0.0.1:6379> LPOP list01
  "5"
  127.0.0.1:6379> LPOP list02
  "1"
  127.0.0.1:6379> LRANGE list01 0 -1
  1) "4"
  2) "3"
  3) "2"
  4) "1"
  127.0.0.1:6379> LRANGE list02 0 -1
  1) "2"
  2) "3"
  3) "4"
  4) "5"
  127.0.0.1:6379> RPOP list01
  "1"
  127.0.0.1:6379> RPOP list02
  "5"
  ```

- lindex：按照索引下标获得元素（从上到下）

  ```bash
  127.0.0.1:6379> LRANGE list01 0 -1
  1) "4"
  2) "3"
  3) "2"
  127.0.0.1:6379> LRANGE list02 0 -1
  1) "2"
  2) "3"
  3) "4"
  127.0.0.1:6379> LINDEX list01 3
  (nil)
  127.0.0.1:6379> LINDEX list01 2
  "2"
  127.0.0.1:6379> LINDEX list02 2
  "4"
  ```

- llen

  ```bash
  127.0.0.1:6379> LLEN list01
  (integer) 3
  ```

- lrem key(List Remove)：删N个value

  ```bash
  127.0.0.1:6379> RPUSH list03 1 1 2 2 2 3  4 5 5 6 6 6
  (integer) 12
  127.0.0.1:6379> LREM list03 3 2		'把3个2删除'
  (integer) 3
  127.0.0.1:6379> LRANGE list03 0 -1
  1) "1"
  2) "1"
  3) "3"
  4) "4"
  5) "5"
  6) "5"
  7) "6"
  8) "6"
  9) "6"
  ```

- `ltrim key 开始index 结束index`：截取指定范围的值后再赋值给key

  ```bash
  127.0.0.1:6379> LPUSH list01 1 2 3 4 5 6 7 8 9
  (integer) 9
  127.0.0.1:6379> LRANGE list01 0 -1
  1) "9"
  2) "8"
  3) "7"
  4) "6"
  5) "5"
  6) "4"
  7) "3"
  8) "2"
  9) "1"
  127.0.0.1:6379> LTRIM list01 3 5	'将list01中下标为3到5之间的值截取出来赋给list01'
  OK
  127.0.0.1:6379> LRANGE list01 0 -1
  1) "6"
  2) "5"
  3) "4"
  ```

- `rpoplpush 源列表 目的列表`

  ```bash
  127.0.0.1:6379> LRANGE list01 0 -1
  1) "6"
  2) "5"
  3) "4"
  127.0.0.1:6379> LRANGE list02 0 -1
  1) "2"
  2) "3"
  3) "4"
  127.0.0.1:6379> RPOPLPUSH list01 list02
  "4"
  127.0.0.1:6379> LRANGE list02 0 -1
  1) "4"
  2) "2"
  3) "3"
  4) "4"
  ```

- `lset key index value`

  ```bash
  127.0.0.1:6379> LRANGE list01 0 -1
  1) "6"
  2) "5"
  127.0.0.1:6379> LSET list01 0 *
  OK
  127.0.0.1:6379> LRANGE list01 0 -1
  1) "*"
  2) "5"
  ```

- `linsert key before/after 值1 值2`

  ```bash
  127.0.0.1:6379> LINSERT list01 before * java
  (integer) 3
  127.0.0.1:6379> LRANGE list01 0 -1
  1) "java"
  2) "*"
  3) "5"
  127.0.0.1:6379> LINSERT list01 after * oracle
  (integer) 4
  127.0.0.1:6379> LRANGE list01 0 -1
  1) "java"
  2) "*"
  3) "oracle"
  4) "5"
  ```

#### 性能总结

`List`是一个字符串链表，left、right 都可以插入添加。如果键不存在，创建新的链表；如果键已存在，新增内容。如果值全部移除，对应的键也就消失了。链表的操作无论是头和尾效率都很高，但假如对中间元素的操作，效率就很惨淡了。

### Redis集合（Set）

常用`Set`命令：
![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/14.Set%E5%B8%B8%E7%94%A8%E5%91%BD%E4%BB%A4.png)

#### 案例

- sadd / smembers / sismember(s is member)

  ```bash
  127.0.0.1:6379> SADD set01 1 1 2 2 3 3
  (integer) 3
  127.0.0.1:6379> SMEMBERS set01
  1) "1"
  2) "2"
  3) "3"
  127.0.0.1:6379> SISMEMBER set01 1
  (integer) 1
  127.0.0.1:6379> SISMEMBER set01 x
  (integer) 0
  ```

- scard：获取集合里面的元素个数

  ```bash
  127.0.0.1:6379> SCARD set01
  (integer) 3
  ```

- `srem key value`：删除集合中的元素

  ```bash
  127.0.0.1:6379> SREM set01 3
  (integer) 1
  127.0.0.1:6379> SMEMBERS set01
  1) "1"
  2) "2"
  ```

- `srandmember key 某个整数（随机出几个数）`

  ```bash
  127.0.0.1:6379> SADD set02 1 2 3 4 5 10 11 12
  (integer) 8
  127.0.0.1:6379> SRANDMEMBER set02 5		'从set02中随机抽取5个数'
  1) "12"
  2) "3"
  3) "10"
  4) "2"
  5) "4"
  127.0.0.1:6379> SRANDMEMBER set02 5
  1) "11"
  2) "12"
  3) "1"
  4) "5"
  5) "10"
  ```

- `spop key`：随机出栈

  ```bash
  127.0.0.1:6379> SMEMBERS set02
  1) "1"
  2) "2"
  3) "3"
  4) "4"
  5) "5"
  6) "10"
  7) "11"
  8) "12"
  127.0.0.1:6379> SPOP set02
  "1"
  127.0.0.1:6379> SPOP set02
  "4"
  127.0.0.1:6379> SPOP set02
  "11"
  ```

- `smove key1 key2 在key1里某个值`：将key1里的某个值赋给key2

  ```bash
  127.0.0.1:6379> SADD set02 x y z
  (integer) 3
  127.0.0.1:6379> SMEMBERS set02
  1) "z"
  2) "y"
  3) "x"
  127.0.0.1:6379> SMOVE set01 set02 1
  (integer) 1
  127.0.0.1:6379> SMEMBERS set02
  1) "z"
  2) "y"
  3) "x"
  4) "1"
  ```

- 数学集合类

  1. 差集：`sdiff key01 key02`  删选出在第一个`set`里面而不在后面任何一个`set`里面的项。

     ```bash
     127.0.0.1:6379> SADD set01 1 2 3 4 5
     (integer) 5
     127.0.0.1:6379> SADD set02 1 2 3 a b
     (integer) 5
     127.0.0.1:6379> SDIFF set01 set02
     1) "4"
     2) "5"
     ```

  2. 交集：`sinter key1 key2`

     ```bash
     127.0.0.1:6379> SINTER set01 set02
     1) "1"
     2) "2"
     3) "3"
     ```

  3. 并集：`sunion key1 key2`

     ```bash
     127.0.0.1:6379> SUNION set01 set02
     1) "b"
     2) "3"
     3) "1"
     4) "a"
     5) "5"
     6) "4"
     7) "2"
     ```

### Redis哈希（Hash）

及其重要！`KV`模式不变，但`V`是一个键值对。
常用`Hash`方法：
![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/15.Hash%E5%B8%B8%E7%94%A8%E5%91%BD%E4%BB%A4.png)

#### 案例

- **hset / hget / hmset / hmget / hgetall / hdel**

  ```bash
  127.0.0.1:6379> hset user id 11		'此时key是`user`，但value是`id 11`'
  (integer) 1
  127.0.0.1:6379> hget user id
  "11"
  
  127.0.0.1:6379> hmset customer id 11 name z3 age 25
  OK
  127.0.0.1:6379> hmget customer id name age
  1) "11"
  2) "z3"
  3) "25"
  
  127.0.0.1:6379> HGETALL customer
  1) "id"
  2) "11"
  3) "name"
  4) "z3"
  5) "age"
  6) "25"
  
  127.0.0.1:6379> HDEL customer name
  (integer) 1
  127.0.0.1:6379> hmget customer id name age
  1) "11"
  2) (nil)
  3) "25"
  ```

- hlen

  ```bash
  127.0.0.1:6379> HLEN customer
  (integer) 2
  ```

- `hexists key 在key里面的某个值的key`

  ```bash
  127.0.0.1:6379> HEXISTS customer id
  (integer) 1
  127.0.0.1:6379> HEXISTS customer name
  (integer) 0
  ```

- **hkeys / hvals**

  ```bash
  127.0.0.1:6379> HKEYS customer
  1) "id"
  2) "age"
  127.0.0.1:6379> HVALS customer
  1) "11"
  2) "25"
  ```

- hincrby / hincrbyfloat

  ```bash
  127.0.0.1:6379> HINCRBY customer age 2
  (integer) 27
  127.0.0.1:6379> HSET customer score 91.5
  (integer) 1
  127.0.0.1:6379> HINCRBYFLOAT customer score 0.5
  "92"
  ```

- hsetnx(如果不存在才往里面加)

  ```bash
  127.0.0.1:6379> HSETNX customer age 26
  (integer) 0
  127.0.0.1:6379> HSETNX customer email abc@qq.com
  (integer) 1
  ```

### Redis有序集合Zset（sorted set）

在`set`基础上，加一个`score`值。之前`set`是`k1 v1 v2 v3`，现在`zset`是`k1 score1 v1 score2 v2`。

`Zset`常用命令：
![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/16.Zset%E5%B8%B8%E7%94%A8%E5%91%BD%E4%BB%A4.png)

#### 案例

- zadd / zrange

  ```bash
  127.0.0.1:6379> zadd zset01 60 v1 70 v2 80 v3 90 v4 100 v5
  (integer) 5
  127.0.0.1:6379> ZRANGE zset01 0 -1
  1) "v1"
  2) "v2"
  3) "v3"
  4) "v4"
  5) "v5"
  127.0.0.1:6379> ZRANGE zset01 0 -1 withscores
   1) "v1"
   2) "60"
   3) "v2"
   4) "70"
   5) "v3"
   6) "80"
   7) "v4"
   8) "90"
   9) "v5"
  10) "100"
  ```

- `zrangebyscore key 开始score 结束score`

  ```bash
  127.0.0.1:6379> ZRANGEBYSCORE zset01 70 90
  1) "v2"
  2) "v3"
  3) "v4"
  127.0.0.1:6379> ZRANGEBYSCORE zset01 70 (90		'(表示不包含'
  1) "v2"
  2) "v3"
  127.0.0.1:6379> ZRANGEBYSCORE zset01 (70 (90
  1) "v3"
  127.0.0.1:6379> ZRANGEBYSCORE zset01 70 90
  1) "v2"
  2) "v3"
  3) "v4"
  127.0.0.1:6379> ZRANGEBYSCORE zset01 70 90 limit 2 2  'limit 2 2 表示从前面的结果中第2个开始截取2个'
  1) "v4"
  ```

- `zrem key 某score下对应的value值`：作用是删除元素

  ```bash
  127.0.0.1:6379> ZREM zset01 v5
  (integer) 1
  127.0.0.1:6379> ZRANGE zset01 0 -1 withscores	'没有100分对应的值了'
  1) "v1"
  2) "60"
  3) "v2"
  4) "70"
  5) "v3"
  6) "80"
  7) "v4"
  8) "90"
  ```

- `zcard/zcount key score区间`、`zrank key values值，作用是获得下标值`、`zscore key 对应值：获得分数`

  ```bash
  127.0.0.1:6379> ZCARD zset01
  (integer) 4		'zset01有4个值'
  127.0.0.1:6379> ZCOUNT zset01 60 80
  (integer) 3		'zset01中60~80之间有三个值'
  127.0.0.1:6379> ZRANK zset01 v4
  (integer) 3		'v4的下标值为3'
  127.0.0.1:6379> ZSCORE zset01 v4
  "90"
  ```

- `zrevrank key values值`：逆序获得下标值。`zrevrange`：逆序获得key中的值。

  ```bash
  127.0.0.1:6379> ZREVRANK zset01 v4
  (integer) 0
  127.0.0.1:6379> ZREVRANGE zset01 0 -1
  1) "v4"
  2) "v3"
  3) "v2"
  4) "v1"
  ```

- `zrevrangebyscore key score1 score2`

  ```bash
  127.0.0.1:6379> ZREVRANGEBYSCORE zset01 80 60
  1) "v3"
  2) "v2"
  3) "v1"
  ```

  