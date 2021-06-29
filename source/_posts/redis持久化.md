---
title: Redis持久化
date: 2019-08-27 20:15:24
categories: Redis
tags: Redis
---

## 解析配置文件redis.conf

### 它在哪？

默认是在下面的目录：
![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/17.%E9%85%8D%E7%BD%AE%E6%96%87%E4%BB%B6%E7%9A%84%E4%BD%8D%E7%BD%AE.png)

但是我把它拷贝了一份到`/home/dqj/myredis/`下。在`Linux`下开发，出厂默认的配置文件不要去改，拷贝出来一份后再进行修改。

### Units单位

![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/18.redis%E5%8D%95%E4%BD%8D.png)

1. 配置文件开头定义了一些基本的度量单位，只支持`bytes`，不支持`bit`。
2. 对大小写不敏感。

### 常见配置redis.conf介绍

1. `Redis`默认不是以守护进程的方式运行，可以通过把`daemonize no`改为`daemonize yes`来启用守护进程。

2. 当`Redis`以守护进程方式运行时，`Redis`默认会把`pid`写入`/var/run/redis.pid`文件，可以通过`pidfile`指定：`pidfile /var/run/redis.pid`。

3. 指定`Redis`监听端口，默认端口是`6379`，作者在自己的一篇博文中解释了为什么选用`6379`作为默认端口，因为6379在手机按键上是`MERZ`对应的号码，而`MERZ`取自意大利女歌手`Alessia Merz`的名字。
   `port 6379`

4. 绑定的主机地址
   `bind 127.0.0.1`

5. 当客户端闲置多长时间后，关闭连接。如果指定为0，表示关闭该功能（即永远不断开连接）。
   `timeout 300`

6. 指定日志记录级别，`Redis`总共支持四个级别：`debug`、`verbose`、`notice`、`warning`，默认为`verbose`。
   `loglevel verbose`

7. 日志记录方式，默认为标准输出，如果配置`Redis`为守护进程方式运行，而这里又配置为日志记录方式为标准输出，则日志将会发送给`/dev/null`

   `logfile stdout`

8. 设置数据库的数量，默认数据库为0，可以使用`SELECT <dbid>`命令在连接上指定数据库id
   `databases 16`

9. 指定在多长时间内，有多少次更新操作，就将数据同步到数据文件，可以多个条件配合。
   `save <seconds> <changes>`
   `Redis`默认配置文件中提供了三个条件：
   `save 900 1`
   `save 300 10`
   `save 60 10000`
   分别表示900秒（15分钟）内有一个更改，300秒（5分钟）内有10个更改，60秒内有10000个更改。

10. 指定存储至本地数据库时是否压缩数据，默认为`yes`，`Redis`采用`LZF`压缩，如果为了节省`CPU`时间，可以关闭该选项，但会导致数据库文件变得巨大。
    `rdbcompression yes`

11. 指定本地数据库文件名，默认值为`dump.rdb`
    `dbfilename dump.rdb`

12. 指定本地数据库存放目录
    `dir ./`

13. 设置当本机为`slav`服务时，设置`master`服务的IP地址及端口，在`Redis`启动时，它会自动从`master`进行数据同步
    `slaveof <masterip> <masterport>`

14. 当`master`服务设置了密码保护时，`slav`服务连接`master`的密码
    `masterauth <master-password>`

15. 设置`Redis`连接密码，如果配置了连接密码，客户端在连接`Redis`时需要通过`AUTH <password>`命令提供密码。默认关闭密码。
    `requirepass foobared`

16. 设置同一时间最大客户端连接数，默认无限制，`Redis`可以同时打开的连接数为`Redis`进程可以打开的最大文件描述符数，如果设置`maxclients 0`，表示不作限制。当客户端连接数到达限制时，`Redis`会关闭新的连接并向客户端返回`max number of clients reached`错误信息。
    `maxclients 128`

17. 指定`Redis`最大内存限制，`Redis`在启动时会把数据加载到内存中，达到最大内存后，`Redis`会尝试清除已到期或即将到期的`Key`。当此方法处理后，若仍然达到最大内存设置，将无法再进行写入操作，但仍然可以进行读取操作。`Redis`新的`vm`机制，会把`Key`存放内存，`Value`会存放在`swap`区。
    `maxmemory <bytes>`

18. 指定是否在每次更新操作后进行日志记录，`Redis`在默认情况下是异步地把数据写入磁盘，如果不开启，可能会在断电时导致一段时间内的数据丢失。因为`Redis`本身同步数据文件是按上面`save`条件来同步的，所以有的数据会在一段时间内只存在于内存中。默认为`no`。

19. 指定更新日志文件名，默认为`appendonly.aof`
    `appendfilename appendonly.aof`

20. 指定更新日志条件，共有3个可选值：
    no：表示等操作系统进行数据缓存同步到磁盘（快）

    always：表示每次更新操作后，手动调用`fsync()`将数据写到磁盘（慢，安全）
    everysec：表示每秒同步一次（折中，默认值）
    `appendfsync everysec`

21. 指定是否启用虚拟内存机制，默认值为`no`，简单的介绍一下，`VM`机制将数据分页存放，由`Redis`将访问量较少的页，即“冷数据”`swap`到磁盘上，访问多的页面由磁盘自动换出到内存中
    `vm-enabled no`

22. 虚拟内存文件路径，默认值为`/tmp/redis.swap`，不可多个`Redis`实例共享。
    `vm-swap-file /tmp/redis.swap`

23. 将所有大于`vm-max-memory`的数据存入虚拟内存，无论`vm-max-memory`设置多小，所有索引数据都是内存存储的（`Redis`的索引数据就是`keys`），也就是说，当`vm-max-memory`设置为0的时候，其实是所有`value`都存于磁盘。默认值为0。
    `vm-max-memory 0`

24. `Redis swap`文件分成了很多的`page`，一个对象可以保存在多个`page`上面，但一个`page`上不能被多个对象共享，`vm-page-size`是要根据存储的数据大小来设定的，作者建议如果存储很多小对象，`page`大小最好设置为32或者64bytes；如果存储很多大对象，则可以使用更大的`page`；如果不确定，就是用默认值。
    `vm-page-size 32`

25. 设置`swap`文件中的`page`数量，由于页表（一种表示页面空闲或使用的`bitmap`）是放在内存中的，在磁盘上每8个`pages`将消耗`1byte`的内存。
    `vm-pages 134217728`

26. 设置访问`swap`文件的线程数，最好不要超过机器的核数，如果设置为0，那么所有对`swap`文件的操作都是串行的，可能会造成比较长时间的延迟。默认值为4。
    `vm-max-threads 4`

27. 设置在向客户端应答时，是否把较小的包合并为一个包发送，默认为开启。
    `glueoutputbuf yes`

28. 指定在超过一定的数量或者最大的元素超过某一临界值时，采用一种特殊的哈希算法。
    `hash-max-zipmap-entires 64`
    `hash-max-zipmap-values 512`

29. 指定是否激活重置哈希，默认为开启。
    `activerehashing yes`

30. 指定包含其它的配置文件，可以在同一主机上多个`Redis`实例之间使用同一份配置文件，而同时各个实例又拥有自己的特定配置文件。
    `include /path/to/local.conf`

## Redis的持久化（重要）

### RDB（Redis DataBase）

在指定的时间间隔内，将内存中的数据集快照写入磁盘。也就是行话讲的`Snapshot`快照，它恢复时是将快照文件直接读到内存里。
`Redis`会单独创建（fork）一个子进程来进行持久化，会先将数据写入到一个临时文件中，待持久化过程都结束了，再用这个临时文件替换上次持久化好的文件。整个过程中， 主进程是不进行任何IO操作的，这就确保了极高的性能。如果需要**进行大规模数据的恢复，且对于数据恢复的完整性不是非常敏感**，那`RDB`方式要比`AOF`方式更加的高效。`RDB`的缺点是最后一次持久化后的数据可能丢失。
RDB 保存的是 `dump.rdb`文件。

#### SNAOSHOTTING快照

RDB 是整个内存压缩过的 Snapshot，RDB 的数据结构，可以配置复合的快照触发条件。默认是15分钟内改了1次 / 5分钟内改了10次 / 1分钟内改了10000次。如果想禁用 RDB 持久化的策略，只要不设置任何 save 指令，或者给 save 传一个空字符串也可以：`save ""`。

- stop-writes-on-bgsave-error
  默认是`yes`，表示后台在存储数据出错时，前台要停止写数据。如果设置成`no`，表示你不在乎数据不一致或者有其它的手段发现和控制。
- rdbcompression
  默认是`yes`。对于存储到磁盘中的快照，可以设置是否进行压缩存储。如果是的话，redis会采用 LZF 算法进行压缩。如果你不想消耗CPU来进行压缩的话，可是设置为`no`。
- rdbchecksum
  默认是`yes`。在存储快照后，还可以让redis使用 CRC64 算法来进行数据校验，但是这样做会增加大约10%的性能损耗。如果希望获取最大性能提升，可以关闭此功能。
- dbfilename
  快照的名字：`dump.rdb`。

#### Fork

Fork 的作用是复制一个与当前进程一样的进程。新进程的所有数据（变量、环境变量、程序计数器等）数值都和原进程一致，但是是一个全新的进程，并作为原进程的子进程。

#### 如何触发RDB快照

1. 配置文件中默认的三种快照配置策略。
2. 命令 save 或者 bgsave。
   执行`save`时只管保存，其它不管，全部阻塞。执行`bgsave`时，Redis 会在后台异步进行快照操作，快照同时还可以响应客户端请求。可以通过`lastsave`命令获取最后一次成功执行快照的时间。
3. 执行`flushall`命令，也会产生`dump.rdb`文件，但里面是空的，无意义。
4. 当执行shutdown 命令时，也会主动地备份数据。

#### 优势

适合大规模的数据恢复，对数据完整性和一致性要求不高。

#### 劣势

在一定时间间隔做一次备份，所以如果 redis 意外 down 掉的话，就会丢失最后一次快照后的所有修改。
Fork 的时候，内存中的数据被克隆了一份，导致2倍的膨胀性需要考虑。

#### 动手测试

1. 首先进入到`/home/dqj/myredis/`，修改`redis-conf`文件。在 VIM 中输入`SNAP`，即可定位到`SNAPSHOTTING`，按回车，向下找到将触发存储的三种策略，修改其中的一种策略进行试验。
   ![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/20.snapshotting.png)
   将第二种策略改为`save 120 10`，即“**120秒内进行10次及以上修改`key`的操作会触发持久化操作**”。
2. 启动 Redis，并进行10次修改`key`的操作：
   ![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/21.2%E5%88%86%E9%92%9F%E5%86%8510%E6%AC%A1%E6%93%8D%E4%BD%9C.png)
3. 在另一个终端上可以看到新生成了`dump.rdb`文件
   ![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/22.%E6%96%B0%E7%94%9F%E6%88%90%E4%BA%86dump.rdb%E6%96%87%E4%BB%B6.png)
4. 现在将它备份一份，名字为`dump_bk.rdb`。实际情况可能会在多个机器上进行备份。
   ![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/23.%E6%8B%B7%E8%B4%9D%E4%B8%80%E4%BB%BDdump.rdb%E6%96%87%E4%BB%B6.png)
5. 这个时候执行`FLUSHALL`操作，它会把所有的 key 值清空。接着关闭 Redis
   ![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/24.%E6%89%A7%E8%A1%8CFLUSHALL%E6%93%8D%E4%BD%9C%E5%B9%B6%E5%85%B3%E9%97%ADredis.png)
6. 关闭后会生成一个新的`dump.rdb`文件，但是这个文件此时已经不保存有 key 值的信息了，因为在保存前执行过`FLUSHALL`操作。
   ![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/25.%E5%85%B3%E9%97%AD%E5%90%8E%E6%96%B0%E7%94%9F%E6%88%90%E7%9A%84dum.rdb%E6%96%87%E4%BB%B6.png)
7. 此时重新启动 redis 发现 key 值都不见了
   ![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/26.%E9%87%8D%E6%96%B0%E5%90%AF%E5%8A%A8redis%E5%8F%91%E7%8E%B0%E9%87%8C%E9%9D%A2%E7%9A%84%E9%94%AE%E5%AF%B9%E4%B8%8D%E8%A7%81%E4%BA%86.png)
8. 要想恢复这些 key 值，可以使用备份文件。先将原来的`dump.rdb`删除，再将备份文件拷贝给新的`dump.rdb`。
   ![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/27.%E7%94%A8%E5%A4%87%E4%BB%BD%E6%96%87%E4%BB%B6%E9%87%8D%E6%96%B0%E7%94%9F%E6%88%90%E5%8E%9F%E6%9D%A5%E7%9A%84dump.rdb%E6%96%87%E4%BB%B6.png)
9. 此时重新启动 Redis，就会加载新创建的`dump.rdb`文件，这个文件里有 key 值信息。
   ![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/28.%E9%87%8D%E6%96%B0%E5%90%AF%E5%8A%A8redis%E5%8F%91%E7%8E%B0key%E5%80%BC%E9%83%BD%E5%9B%9E%E6%9D%A5%E4%BA%86.png)



### AOF（Append Only FIle）

**以日志的形式来记录每个写操作**。将Redis执行过的所有写指令记录下来（读操作不记录），只许追加文件但不可以改写文件。redis启动之初会读取该文件重新构建数据，换言之，redis重启的话就根据日志文件的内容将写指令从前到后执行一次以完成数据的恢复工作。

#### 配置文件内容

- appendonly
  默认`appendonly`关闭，若要使用则需改成`yes`。
- appendfilename
  就是`appendonly.aof`，不要去改。
- **appendfsync**
  写数据到磁盘的行为，redis 有三种模式：
  1. always
     同步持久化。每次发生数据变更会被立即记录到磁盘，性能较差但数据完整性比较好。
  2. everysec
     出厂默认设置。异步操作，每秒记录。如果一秒内宕机，有数据丢失。
  3. no
- no-appendfsync-on-rewrite：重写时是否可以运用`appendfsync`，用默认 no 即可，保证数据安全性。
- auto-aof-rewrite-min-size：设置重写的基准值。（具体的数值）
- auto-aof-rewrite-percentage：设置重写基准值。（百分比）

#### Rewrite（重要）

AOF 采用文件追加方式，文件会越来越大，为避免出现此种情况，新增了**重写机制**，当 AOF 文件的大小超过所设定的阈值时，Redis 就会启动 AOF 文件的内容压缩，只保留可以恢复数据的最小指令集。可以使用命令`bgrewriteaof`。

重写原理：AOF 文件持续增长而过大时，会 **fork 出一条新进程来将文件重写**（也是先写临时文件最后再 rename），遍历新进程的内存中的数据，每条记录有一条set语句。重写 aof 文件的操作，并没有读取旧的 aof 文件，而是将整个内存中的数据库内容用命令的方式重新写了一个新的 aof 文件，这点和快照有点像。

触发机制：Redis 会记录上次重写时的 AOF 大小，默认配置是当 AOF 文件大小是上次 rewrite 后大小的一倍且文件大于 64M 时触发。

#### 优势

- 每修改同步：`appendfsync always`同步持久化。每次发生数据变更会被立即记录到磁盘，性能较差但数据完整性比较好。
- 每秒同步：`appendfsync everysec`异步操作，每秒记录。如果一秒内宕机，有数据丢失。
- 不同步：`appendfsync no`从不同步。

#### 劣势

相同数据集数据而言 aof 文件要远大于 rdb 文件，恢复速率慢于 rdb。
AOF运行效率慢于 RDB，每秒同步策略效率较好，不同步效率和 RDB 相同。

#### 动手测试

1. 这次拷贝一份配置文件：`cp redis.conf redis_aof.conf`。在新的配置文件里修改配置。

2. 用`VIM`进入配置文件，输入`/APPEND`即可定位到对应的位置，将`appendonly`开启：
   ![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/29.%E5%90%AF%E5%8A%A8appendonly.png)

3. 将原来存在于`/usr/local/bin`下的`dump.rdb`和`dump_bk.rdb`先删除。使用以下命令启动 Redis：

   ```bash
   redis-server /home/dqj/myredis/redis_aof.conf
   redis-cli -p 6379
   ```

4. 设置 key 值，设完 key 值后再进行`FLUSHALL`操作，关闭 redis 并退出。

5. 此时发现`/usr/local/bin`下生成了`appendonly.aof`文件，通过`cat`命令查看该文件：
   ![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/30.%E6%9F%A5%E7%9C%8Bappendonly.aof%E6%96%87%E4%BB%B6.png)

6. 可以看到这个文件里记录了我之前的操作。随后重新开启 redis ，发现里面没有 key 值数据。因为之前我执行了`FLUSHALL`命令，同样的，在`appendonly.aof`中也记录了这个操作。
   ![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/31.%E9%87%8D%E6%96%B0%E5%90%AF%E5%8A%A8%E5%8F%91%E7%8E%B0%E6%B2%A1%E6%9C%89key%E5%80%BC.png)

7. 若想恢复原先的 key 值，则需要`vim appendonly.aof`，定位到最后一行，执行`dd`命令，将最后一行数据`FLUSHALL`删除。随后重新启动 redis，通过`keys *`即可看到 key 值。

8. 此时在`appendonly.aof`的最后一行胡乱输入一通并保存。退出 redis，此时`/usr/local/bin`下有`appendonly.aof`和`dump.rdb`这两个文件。但是`dump.rdb`是正常文件，而`appendonly.aof`则是被我修改坏了的。

   ![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/32.%E6%8D%9F%E5%9D%8Fappendonly.aof.png)

   ![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/33.%E4%B8%A4%E4%B8%AA%E6%96%87%E4%BB%B6.png)

9. 此时启动 redis 命令，注意到这个时候`appendonly.aof`和`dump.rdb`这两个文件同时存在，启动的时候输入如下内容：
   ![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/34.%E6%97%A0%E6%B3%95%E6%AD%A3%E5%B8%B8%E5%90%AF%E5%8A%A8.png)
   没有正常启动，说明当这两个文件共存时，redis 找的是`appendonly.aof`这个文件。

10. 可以通过`/usr/local/bin`下的`redis-check-aof`进行修复：
    ![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/35.%E4%BF%AE%E5%A4%8Dappendonly.aof.png)

11. 重新启动 redis 可以看到 key 值都复现了
    ![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/36.%E9%87%8D%E6%96%B0%E5%90%AF%E5%8A%A8%E5%8F%91%E7%8E%B0key%E5%80%BC%E9%83%BD%E5%87%BA%E7%8E%B0%E4%BA%86.png)

### 总结

- RDB 持久化方式能够在指定的时间间隔内对你的数据进行快照存储。
- AOF持久化方式记录每次对服务器的写操作，当服务器重启的时候会重新执行这些命令来恢复原始的数据。AOF 命令以 redis 协议追加保存每次写的操作到文件末尾。Redis 还能对 AOF 文件进行后台重写，使得 AOF 文件的体积不至于过大。
- 只做缓存：如果你只希望你的数据在服务器运行的时候存在，你也可以不使用任何持久化方式。
- 同时开启两种持久化方式（官方建议）：
  在这种情况下，当 redis 重启时会优先载入 AOF 文件来恢复原始的数据，因为在通常情况下 AOF 文件保存的数据集要比 RDB 文件保存的数据集要完整。
  RDB 的数据不实时，同时使用两者时服务器重启也只会找 AOF 文件。那要不要只使用 AOF 呢？作者建议不要，因为 RDB 更适合用于备份数据库（AOF 在不断变化不好备份），快速重启。而且不会有 AOF 那样可能潜在的 bug，留着作为一个万一的手段。
- 官方推荐两个都用；如果对数据不敏感，可以选单独用RDB；不建议单独用AOF，因为可能出现Bug;如果只是做纯内存缓存，可以都不用。