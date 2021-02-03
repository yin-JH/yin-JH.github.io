---
layout: post
title: Redis系列(七)之Redis集群搭建
categories: Redis
description: Redis集群搭建的细节以及Redis集群的实现原理
keyword: Redis cluster
---

### Redis配置集群

#### 搭建主从复制模型

- 启动三个Redis，然后让二号机和三号机追随一号机

  ![image](\images\posts\Redis\2021-1-30-Redis系列七之Redis集群的搭建和哨兵-1.jpg)

  ![image](\images\posts\Redis\2021-1-30-Redis系列七之Redis集群的搭建和哨兵-2.jpg)

- 查看一号机服务器的信息

  ![image](\images\posts\Redis\2021-1-30-Redis系列七之Redis集群的搭建和哨兵-3.jpg)

  1. 日志显示已经和6380、6381建立联系，并且有6mb用于 copy on write

- 查看二号机服务器的信息（和三号机基本一样，所以只看一台）

  ![image](\images\posts\Redis\2021-1-30-Redis系列七之Redis集群的搭建和哨兵-4.jpg)

  1. 日志显示已经连接上主机6379
  2. 日志显示已经将老的数据全部flushing，并且load DB

- 自此，主从复制模型已经搭建完毕，接下来我们来稍微测试一下redis主从复制

#### 主从复制模型测试

- 主机中创建key，并且set值

  ![image](\images\posts\Redis\2021-1-30-Redis系列七之Redis集群的搭建和哨兵-5.jpg)

- 从机来尝试get

  ![image](\images\posts\Redis\2021-1-30-Redis系列七之Redis集群的搭建和哨兵-6.jpg)

- 从机创建key

  ![image](\images\posts\Redis\2021-1-30-Redis系列七之Redis集群的搭建和哨兵-7.jpg)

  从机无法创建，这和我们前面提到的，主机可以全量增删改查，从机只能查一致

### Redis哨兵配置

#### Redis哨兵介绍

Redis 的 Sentinel 系统用于管理多个 Redis 服务器（instance）， 该系统执行以下三个任务：

- **监控（Monitoring）**： Sentinel 会不断地检查你的主服务器和从服务器是否运作正常。
- **提醒（Notification）**： 当被监控的某个 Redis 服务器出现问题时， Sentinel 可以通过 API 向管理员或者其他应用程序发送通知。
- **自动故障迁移（Automatic failover）**： 当一个主服务器不能正常工作时， Sentinel 会开始一次自动故障迁移操作， 它会将失效主服务器的其中一个从服务器升级为新的主服务器， 并让失效主服务器的其他从服务器改为复制新的主服务器； 当客户端试图连接失效的主服务器时， 集群也会向客户端返回新主服务器的地址， 使得集群可以使用新主服务器代替失效服务器。

Redis Sentinel 是一个分布式系统， 你可以在一个架构中运行多个 Sentinel 进程（progress）， 这些进程使用流言协议（gossip protocols)来接收关于主服务器是否下线的信息， 并使用投票协议（agreement protocols）来决定是否执行自动故障迁移， 以及选择哪个从服务器作为新的主服务器。

虽然 Redis Sentinel 释出为一个单独的可执行文件 redis-sentinel ， 但实际上它只是一个运行在特殊模式下的 Redis 服务器， 你可以在启动一个普通 Redis 服务器时通过给定 –sentinel 选项来启动 Redis Sentinel 。

#### Redis哨兵配置

1. 我们先为Redis Sentinel编写配置文件

   ![image](\images\posts\Redis\2021-1-30-Redis系列七之Redis集群的搭建和哨兵-8.jpg)

   第一行的意思是这个哨兵的端口号是26379 

   第二行的意思是，这个哨兵可以监控  redis组名为 mymaster的redis集群，这个集群的主端口号为6379，想要判断这个主机掉线需要至少需要两台哨兵判断这台主机掉线

2. 将配置文件复制两份，一份是端口是26380，另一份端口是26381

   ![image](\images\posts\Redis\2021-1-30-Redis系列七之Redis集群的搭建和哨兵-9.jpg)![image](\images\posts\Redis\2021-1-30-Redis系列七之Redis集群的搭建和哨兵-10.jpg)

3. 在redis的文件目录下启动哨兵

   ![image](\images\posts\Redis\2021-1-30-Redis系列七之Redis集群的搭建和哨兵-11.jpg)

   ![image](\images\posts\Redis\2021-1-30-Redis系列七之Redis集群的搭建和哨兵-12.jpg)

   ![image](\images\posts\Redis\2021-1-30-Redis系列七之Redis集群的搭建和哨兵-13.jpg)

4. 查看哨兵的信息

   ![image](\images\posts\Redis\2021-1-30-Redis系列七之Redis集群的搭建和哨兵-14.jpg)

   我们发现，哨兵已经发现，redis_6379已经有两个从机，分别是6380和6381。而哨兵自己也发现了还有两个哨兵在监视这个主机

#### Redis哨兵如何自动发现其他哨兵

一个 Sentinel 可以与其他多个 Sentinel 进行连接， 各个 Sentinel 之间可以互相检查对方的可用性， 并进行信息交换。

你无须为运行的每个 Sentinel 分别设置其他 Sentinel 的地址， 因为 Sentinel 可以通过发布与订阅功能来自动发现正在监视相同主服务器的其他 Sentinel ， 这一功能是通过向频道 **sentinel**:hello 发送信息来实现的。

与此类似， 你也不必手动列出主服务器属下的所有从服务器， 因为 Sentinel 可以通过询问主服务器来获得所有从服务器的信息。

- 每个 Sentinel 会以每两秒一次的频率， 通过发布与订阅功能， 向被它监视的所有主服务器和从服务器的 **sentinel**:hello 频道发送一条信息， 信息中包含了 Sentinel 的 IP 地址、端口号和运行 ID （runid）。
- 每个 Sentinel 都订阅了被它监视的所有主服务器和从服务器的 **sentinel**:hello 频道， 查找之前未出现过的 sentinel （looking for unknown sentinels）。 当一个 Sentinel 发现一个新的 Sentinel 时， 它会将新的 Sentinel 添加到一个列表中， 这个列表保存了 Sentinel 已知的， 监视同一个主服务器的所有其他 Sentinel 。
- Sentinel 发送的信息中还包括完整的主服务器当前配置（configuration）。 如果一个 Sentinel 包含的主服务器配置比另一个 Sentinel 发送的配置要旧， 那么这个 Sentinel 会立即升级到新配置上。
- 在将一个新 Sentinel 添加到监视主服务器的列表上面之前， Sentinel 会先检查列表中是否已经包含了和要添加的 Sentinel 拥有相同运行 ID 或者相同地址（包括 IP 地址和端口号）的 Sentinel ， 如果是的话， Sentinel 会先移除列表中已有的那些拥有相同运行 ID 或者相同地址的 Sentinel ， 然后再添加新 Sentinel 。

#### Redis哨兵故障转移

1. 首先我们停掉Redis主机

   ![image](\images\posts\Redis\2021-1-30-Redis系列七之Redis集群的搭建和哨兵-15.jpg)

2. 我们发现，从机正在报错，无法找到主机

   ![image](\images\posts\Redis\2021-1-30-Redis系列七之Redis集群的搭建和哨兵-16.jpg)

3. 超过一段时间后，哨兵开始投票，选出6380作为主机

   ![image](\images\posts\Redis\2021-1-30-Redis系列七之Redis集群的搭建和哨兵-17.jpg)

4. 我们可以使用6380进行增删改查操作了，因为6380已经是主机了

   ![image](\images\posts\Redis\2021-1-30-Redis系列七之Redis集群的搭建和哨兵-18.jpg)

5. 这个时候我们再让6379重新上线

   此时，重新上线的6379已经变成了从机，无法再变成主机了

   ![image](\images\posts\Redis\2021-1-30-Redis系列七之Redis集群的搭建和哨兵-19.jpg)

   但是，仍然可以得到6379在宕机的时候6380 set 的值

   ![image](\images\posts\Redis\2021-1-30-Redis系列七之Redis集群的搭建和哨兵-20.jpg)

   这个原理是，redis主会维护一个队列，这个队列里面方有操作的日志，如果一个从机掉线了，在此连接回来的话就会从这个日志队列里面读取它掉线期间没有进行的操作，这个队列的默认大小是1mb

   ![image](\images\posts\Redis\2021-1-30-Redis系列七之Redis集群的搭建和哨兵-21.jpg)

### Redis分区

#### 分区介绍

- 简单而言，分区就是为了承载更多的数据，将数据分不到不同的计算机的Redis上。当公司的业务成长到一定情况下时，分区是不得不进行的环节。

#### 分区目的

- 分区可以让Redis管理更大的内存，Redis将可以使用所有机器的内存。如果没有分区，你最多只能使用一台机器的内存。
- 分区使Redis的计算能力通过简单地增加计算机得到成倍提升,Redis的网络带宽也会随着计算机和网卡的增加而成倍增长。

#### 分区方案划分

- **数据可以按照不同业务划分**

  1. 这种直接从客户端出发，不同的业务发给不同的redis即可

- **数据不可按照业务划分**

  1. hash+取模

     每一条操作都可以拿来通过一个hash算法变成一个数字，这个数字再拿来取模，取模得到的结果就作为不同编号的redis的存放地址

     假如我有三台redis，那么就可以%3，得到的结果就作为记录存放的redis地址

     hash取模的方法有一个天生的弊端，那就是模数制是固定的，这种方式会限制死redis的数量，对系统的扩展性有影响
     
  2. random

     每一个操作结果可以直接随机放入后面的redis中

     这种方式的弊端非常明显，客户端都很难再次得到这些数据，这种方式是有特定的使用场景的

     ![image](\images\posts\Redis\2021-1-30-Redis系列七之Redis集群的搭建和哨兵-22.jpg)

     我可以用redis做一个消息队列，每一个redis中都有一个list，这些list的key都是 k1 ，我客户端随机的向后面redis的k1中存放数据，后面的一个kafka就不断地消费redis中的数据

  3. 一致性哈希

     ![image](\images\posts\Redis\2021-1-30-Redis系列七之Redis集群的搭建和哨兵-23.jpg)

     我们可以看到，这个hash环其实是不存在的，是虚构出来方便理解的，我们的做法其实很简单，就是通过一致性hash计算，得到等长的数，节点一个，存放数据的key一个，然后key离哪一个节点近就存放到哪一个节点中

     这种方式的扩展性就比hash+取模好，并且添加新节点不用将整个缓存中的内容全部洗牌

     添加节点模拟

     ![image](\images\posts\Redis\2021-1-30-Redis系列七之Redis集群的搭建和哨兵-24.jpg)

     我们发现，添加的新节点和以前某个数据更加近，这样就会出现一个问题：后面想要查找这个数据的时候会发现找不到，但是这个数据其实存储在redis01中的

     解决方法

     1. 直接不管，反正是缓存，直接击穿给数据库查询，查询的结果再放入redis04中即可
     2. 设定redis查找范围，我们每次不是只查询离我最近的节点，而是查询离我最近的两个节点，这样可以解决一部分问题

     一致性哈希的小缺点

     - 一致性哈希有可能会出现redis倾斜的现象，即我的哈希环上有2个节点，但是数据存储的时候一个节点中存储的数据远远大于另一个节点。解决的方法很简单，就是我这两个redis我每一个都给他在哈希环上提供10个虚拟地址，尽量均匀分布在哈希环上，这样就可以解决倾斜问题

#### 分区的缺点

- 涉及多个key的操作通常不会被支持。例如你不能对两个集合求交集，因为他们可能被存储到不同的Redis实例（实际上这种情况也有办法，但是不能直接使用交集指令）。
- 同时操作多个key,则不能使用Redis事务，除非数据在同一个redis上
- 分区使用的粒度是key，不能使用一个非常长的排序key存储一个数据集（The partitioning granularity is the key, so it is not possible to shard a dataset with a single huge key like a very big sorted set）.
- 当使用分区的时候，数据处理会非常复杂，例如为了备份你必须从不同的Redis实例和主机同时收集RDB / AOF文件。
- 分区时动态扩容或缩容可能非常复杂。Redis集群在运行时增加或者删除Redis节点，能做到最大程度对用户透明地数据再平衡，但其他一些客户端分区或者代理分区方法则不支持这种特性。然而，有一种*预分片*的技术也可以较好的解决这个问题。

#### Redis分区实战

- 假如说我后端有3个redis，前端有2个client，现在我的客户端正在往redis中写入数据，每一个客户端都有和redis建立连接，这个 连接数量相当多，压力相当大

  ![image](\images\posts\Redis\2021-1-30-Redis系列七之Redis集群的搭建和哨兵-25.jpg)

- 这个时候，我们可以会想起以前学过的技术——反向代理

  ![image](\images\posts\Redis\2021-1-30-Redis系列七之Redis集群的搭建和哨兵-26.jpg)

- 当我们公司的业务量极具提升，一个nginx顶不住时我们可以做nginx集群，并且前面提供一个lvs作为入口

  ![image](\images\posts\Redis\2021-1-30-Redis系列七之Redis集群的搭建和哨兵-27.jpg)

- 这个lvs的任务非常繁重并且重要，我们还可以给它准备一个备机，然后使用keepalived监控

  ![image](\images\posts\Redis\2021-1-30-Redis系列七之Redis集群的搭建和哨兵-28.jpg)

- 这个keepalived还可以顺便监控nginx集群

#### 比较出名好用的redis分区代理技术

1. twemproxy

   这个是twitter的redis分区管理技术，这就是一致性哈希的使用者

   https://github.com/twitter/twemproxy

2. predixy

   这是一个国人制作的redis代理，吊打市面上一众的redis代理，包括tweproxy

   https://github.com/joyieldInc/predixy

