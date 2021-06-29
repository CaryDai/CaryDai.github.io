---
title: Redis主从复制
date: 2019-09-2 21:39:12
categories: Redis
tags: Redis
---

## 主从简介

配置多台Redis 服务器，以主机和备机的身份分开。主机数据更新后，根据配置和策略，自动同步到备机的
master/salver 机制。Master 以写为主，Slave 以读为主，二者之间自动同步数据。

目的：

- 读写分离提高Redis 性能；
- 避免单点故障，容灾快速恢复。

原理：每次从机联通后，都会给主机发送`sync`指令，主机立刻进行存盘操作，发送`RDB` 文件给从机。从机收到`RDB`文件后，进行全盘加载。之后每次主机的写操作，都会立刻发送给从机，从机执行相同的命令。

![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/46.%E4%B8%BB%E6%9C%BA%E4%B8%8E%E4%BB%8E%E6%9C%BA.png)

### 一主二仆

1. 首先拷贝多份配置文件并分别命名为`redis6379.conf`、`redis6380.conf`、`redis6381.conf`。修改各自的端口号为 6379、6380、6381。并修改相关日志文件和`dbfilename`。
2. 打开三个终端，使用命令`title 端口号`将三个终端的名字分别改为79/80/81。
3. 以`79`作为主机，`80`、`81`作为从机：
   ![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/47.%E4%B8%BB%E4%BB%8E%E6%9C%BA.png)
   可以看到从机可以把主机的所有值都备份过去。
4. 此时使用`info replication`命令查看：
   `79`的信息：
   ![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/48.79%E7%9A%84info.png)
   `80`的信息：
   ![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/49.80%E7%9A%84info.png)
5. 当主机关停后，两台从机的状态还是`slave`，并不会有一台变成主机：
   ![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/50.%E4%B8%BB%E6%9C%BA%E5%AE%95%E6%9C%BA%E6%97%B6.png)
6. 当主机重新启动后，设置了一个新值，从机还是能正常取得新的值：
   ![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/51.%E4%B8%BB%E6%9C%BA%E6%B4%BB%E4%BA%86.png)
7. 当从机宕机后，如果重新开启，则它的角色又是一台主机（每次与master断开之后，都需要重新连接，除非你配置进`redis.conf`文件）：
   ![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/52.%E4%BB%8E%E6%9C%BA%E6%AD%BB%E4%BA%86.png)

### 薪火相传

上一个 Slave 可以是下一个 Slave 的 Master，Slave 同样可以接收其它 salves 的连接和同步请求，那么该 slave 作为了链条中下一个的 master，可以有效减轻 master 的写压力。

中途变更转向，会清楚之前的数据，重新建立拷贝最新的。

把`81`变成`80`的从机：
![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/53.%E8%96%AA%E7%81%AB%E7%9B%B8%E4%BC%A0.png)

可以看到`80`的身份还是从机，但是它又连着1个从机。

### 反客为主

`SLAVEOF no one`：使当前数据库停止与其它数据库的同步，转成主数据库。在`80`中执行这个命令，它就会成为一个新的主机。

## 哨兵模式

简单讲，哨兵模式就是“反客为主”的自动版。能够从后台监控主机是否故障，如果出故障了**根据投票数自动将从库转换为主库**。

1. 首先在`/home/dqj/myredis`下编写哨兵的配置文件`sentinel.conf`，内容为：

   ```bash
   sentinel monitor host6379 127.0.0.1 6379 1
   ```

   哨兵监控6379这台主机。最后这个1表示主机宕机后，投票数大于1的从机将会成为主机。

2. cd到`/usr/local/bin`目录下，执行`redis-sentinel /home/dqj/myredis/sentinel.conf`命令启动哨兵：
   ![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/54.%E5%93%A8%E5%85%B5%E6%A8%A1%E5%BC%8F1.png)

3. 此时关停79主机，过一会儿可以看到哨兵进行了新的主机选择。这里选了81作为新的主机：
   ![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/54.%E5%93%A8%E5%85%B5%E6%A8%A1%E5%BC%8F2.png)

   ![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/54.%E5%93%A8%E5%85%B5%E6%A8%A1%E5%BC%8F3.png)

4. 这个时候再重新启动79这台机子，发现哨兵会将它分配到80主机下，作为80的从机：
   ![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/54.%E5%93%A8%E5%85%B5%E6%A8%A1%E5%BC%8F4.png)

