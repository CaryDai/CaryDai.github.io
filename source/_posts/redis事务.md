---
title: Redis事务
date: 2019-08-28 10:54:32
categories: Redis
tags: Redis
---

## 是什么

Redis 事务可以一次执行多个命令，本质是一组命令的集合。一个事务中的所有命令都会序列化，按顺序地串行化执行而不会被其它命令插入，不许加塞。

## 能干嘛

在一个队列中，一次性、顺序性、排他性地执行一系列命令。

## 怎么玩

### 常用命令

![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/37.redis%E5%B8%B8%E7%94%A8%E5%91%BD%E4%BB%A4.png)

### 正常执行

![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/38.%E6%AD%A3%E5%B8%B8%E6%89%A7%E8%A1%8C%E4%BA%8B%E5%8A%A1.png)

### 放弃事务

当发现这个事务不需要的时候，可是执行`DISCARD`命令来放弃事务。

![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/39.%E6%94%BE%E5%BC%83%E4%BA%8B%E5%8A%A1.png)

### 全体连坐

![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/40.%E5%85%A8%E4%BD%93%E8%BF%9E%E5%9D%90.png)

一个命令错误，会导致整个事务无法执行。

### 冤头债主

![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/41.%E5%8F%AA%E6%89%A7%E8%A1%8C%E6%AD%A3%E5%B8%B8%E7%9A%84%E5%91%BD%E4%BB%A4.png)

这里命令是正确的，但是`k1`是一个字符串，不能进行`+1`算术运算，所以第一条语句不执行，但其它语句正常执行。

通过以上两个例子我们可以发现 redis 部分支持事务。

### watch监控

- 悲观锁、乐观锁、CAS
  悲观锁把整张表锁了。每次去拿数据的时候都认为别人会修改，所以每次在拿数据的时候都会上锁，这样别人想拿这个数据就会 block 直到它拿到锁。传统的关系型数据库里面就用到了很多这种锁机制，比如行锁，表锁，读锁，写锁等，都是在做操作之前先上锁。
  **乐观锁**，顾名思义，就是很乐观，每次去拿数据的时候都认为别人不会修改，所以不把整张表锁住。它在每条记录后面加一个`version`这个“版本号”字段，每次修改完后，版本号就加1。在更新的时候会判断一下在此期间别人有没有去更新数据。乐观锁适用于多读的应用类型，这样可以提高吞吐率。**乐观锁策略：提交版本必须大于记录当前版本才能执行更新**。

看下面的例子：

我先设定两个值`balance`和`debt`，分别代表信用卡的可用额度和欠费金钱。第一次执行事务前，用`WATCH`来监控`balance`：
![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/42.%E7%AC%AC%E4%B8%80%E6%AC%A1%E6%AD%A3%E5%B8%B8%E6%89%A7%E8%A1%8C%E4%BA%8B%E5%8A%A1%E5%90%8Ewatch.png)
正常执行。在这一次事务执行完后，使用`WATCH balance`命令继续监控余额。

此时打开另一个终端，将`balance`的值设为800：
![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/43.%E5%8F%A6%E4%B8%80%E4%B8%AA%E7%BB%88%E7%AB%AF%E4%BF%AE%E6%94%B9%E4%BD%99%E9%A2%9D%E7%9A%84%E5%80%BC.png)

再回到原终端，假设又花了20块钱，此时执行事务发现无法执行：
![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/44.%E5%8E%9F%E7%BB%88%E7%AB%AF%E6%97%A0%E6%B3%95%E6%AD%A3%E5%B8%B8%E6%89%A7%E8%A1%8C%E4%BA%8B%E5%8A%A1.png)

当发现别人改了余额值的时候，可以使用`UNWATCH`解锁
![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/45.unwatch%E5%91%BD%E4%BB%A4.png)

一旦执行了`exec`或`unwatch`，之前加的监控锁都会被取消掉。

小结：Watch 指令，类似于乐观锁，事务提交时，如果 Key 值已被别的客户端改变，比如某个 list 已被别的客户端 pop/push过了，整个事务队列都不会被执行。通过 WATCH 命令在事务执行之前监控了多个 Keys，倘若在 Watch 之后有任何 Key 的值发生了变化， EXEC 命令执行的事务都将被废弃，同时返回`Nullmulti-bulk`应答以通知调用者事务执行失败。

## Redis的发布订阅（了解）

消息订阅是进程间的一种消息通信方式，即发送者（pub）发送消息，订阅者（sub）接收消息。