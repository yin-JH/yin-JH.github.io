---
post: layout
title: Redis系列(四)之Redis的管道使用、发布订阅、事务、BloomFilter
categories: Redis
description: Redis的进阶使用
keywords: Redis
---

本篇讲解Redis的管道技术、发布订阅功能、事务、缓存穿透的解决方案(使用BloomFilter)
======

### Redis的管道使用

#### Redis的通信模型

- Redis是一种基于客户端-服务器模型以及请求/相应协议的TCP服务，这意味着，一个Redis协议会遵循以下的通信流程
  1. Redis客户端向服务器发起一个查询请求，并监听socket返回，通常是阻塞状态，等待服务器返回
  2. Redis服务器处理命令并将结果返回客户端

#### Redis的通信模型给Redis带来的影响

- 客户端和服务器之间通过网络相互连接，这个连接可以很快（loop back），也可以很慢（跨越数个路由器），不论快慢，Redis总是能够完成从客户端发送请求到服务器，再从服务器发送结果给客户端。而在这个由客户端发送包到服务器，再由服务器发送包回到客户端的时间就叫做RTT。如果我们没有一次性将一堆命令传给Redis执行的操作的话，Redis的可用性将大大下降，下面举一个例子
- 假设Redis的客户端和服务器之间的延迟非常大，一次RTT需要的时间是250ms，假设我们现在需要往一个sorted_set中添加成员，有1W个成员需要添加，即使Redis每秒钟可以处理100K条请求，但是因为网络受限，我们每秒钟只能处理4条

#### Redis管道介绍(pipeline)

- 通过管道技术，我们可以一次性向服务器发起多条命令，不需要等待服务器返回上一条的结果就可以执行下一条，最后再得到最后一条执行完毕的结果；这样大大提升了Redis的性能
- 管道(pipeline)技术已经流行了数十年了，很早以前就通过管道技术加快了服务器下载邮件的速度，Redis的每一个版本都支持管道技术

#### Redis管道的使用

1. 首先，想要使用管道，我们可以安装一个nc

   ![image](\images\posts\Redis\2021-1-26-Redis系列四之Redis的管道、消息订阅、持久化等-1.jpg)

2. 我们先来使用一下nc

   当我们的redis服务器启动的时候，我们只要能够通过redis的端口给它发送命令就可以了，哪怕这次我们没有使用redis-cli

   ![image](\images\posts\Redis\2021-1-26-Redis系列四之Redis的管道、消息订阅、持久化等-2.jpg)

3. 组合nc来使用管道

   ![image](\images\posts\Redis\2021-1-26-Redis系列四之Redis的管道、消息订阅、持久化等-3.jpg)

### Redis的发布订阅(Pub/Sub)

#### Pub和Sub的简单介绍

订阅，取消订阅和发布实现了发布/订阅消息范式(引自wikipedia)，发送者（发布者）不是计划发送消息给特定的接收者（订阅者）。而是发布的消息分到不同的频道，不需要知道什么样的订阅者订阅。订阅者对一个或多个频道感兴趣，只需接收感兴趣的消息，不需要知道什么样的发布者发布的。这种发布者和订阅者的解耦合可以带来更大的扩展性和更加动态的网络拓扑。

#### Pub和Sub的简单使用

- 当多个Redis client连接上同一个Redis Service的时候，不同的redis cli就可以通过订阅、发布的方式交互

  ![image](\images\posts\Redis\2021-1-26-Redis系列四之Redis的管道、消息订阅、持久化等-4.jpg)

  ![image](\images\posts\Redis\2021-1-26-Redis系列四之Redis的管道、消息订阅、持久化等-5.jpg)

- 1006寝室需要开黑，但是大家都回家了，我们使用redis进行通信。二号机订阅频道1006，等待室友发送消息

  ![image](\images\posts\Redis\2021-1-26-Redis系列四之Redis的管道、消息订阅、持久化等-6.jpg)

- 二号机已经就位了，现在一号机发送消息

  ![image](\images\posts\Redis\2021-1-26-Redis系列四之Redis的管道、消息订阅、持久化等-7.jpg)

  ![image](\images\posts\Redis\2021-1-26-Redis系列四之Redis的管道、消息订阅、持久化等-8.jpg)

- Pub/Sub的运用非常简单，但是要注意的是，只有当你订阅的后才可以接受后面的消息，如果你订阅的前面还有消息的话那么是看不到的

#### Pub/Sub的现实使用架构方案

- 项目背景：现在我们使用Redis的发布订阅功能制作一个QQ群聊功能，所有在群里的成员都可以收到qq消息；所有的群员还可以在群里发送消息

  ![image](\images\posts\Redis\2021-1-26-Redis系列四之Redis的管道、消息订阅、持久化等-9.jpg)

### Redis的事务

#### Redis事务的介绍

- Redis的事务非常简单，没有mysql事务那么复杂

- Redis的事务不支持回滚，这是因为redis是一个高速缓存中间件，redis允许一定程度上的数据不一致来换取高效和快速，这是redis的特性，所以不支持回滚

  ![image](\images\posts\Redis\2021-1-26-Redis系列四之Redis的管道、消息订阅、持久化等-10.jpg)

#### Redis事务的使用

- Redis事务的使用方式是发起 ’multi‘ 指令，此时redis就会开启事务，接下来这个客户端发出的指令全部都会被存放在一片单独的空间，等待指令 ‘exec’ 出现。当exec出现时，redis就会将存储了一堆指令的空间中的指令全部取出来，一起执行

  ![image](\images\posts\Redis\2021-1-26-Redis系列四之Redis的管道、消息订阅、持久化等-11.jpg)

  ![image](\images\posts\Redis\2021-1-26-Redis系列四之Redis的管道、消息订阅、持久化等-12.jpg)

  ![image](\images\posts\Redis\2021-1-26-Redis系列四之Redis的管道、消息订阅、持久化等-13.jpg)

  ![image](\images\posts\Redis\2021-1-26-Redis系列四之Redis的管道、消息订阅、持久化等-14.jpg)

  ![image](\images\posts\Redis\2021-1-26-Redis系列四之Redis的管道、消息订阅、持久化等-15.jpg)

- Redis事务的先后顺序是取决于哪一个事务的exec先出现，假如我有两个客户端，一号机向redis service发起了查询k1的事务，二号机向redis service发起了删除k1的事务，虽然是一号机先发出multi，但是二号机的exec先出现，于是就会先删除k1

  ![image](\images\posts\Redis\2021-1-26-Redis系列四之Redis的管道、消息订阅、持久化等-16.jpg)

  ![image](\images\posts\Redis\2021-1-26-Redis系列四之Redis的管道、消息订阅、持久化等-17.jpg)

  ![image](\images\posts\Redis\2021-1-26-Redis系列四之Redis的管道、消息订阅、持久化等-18.jpg)

- Redis的watch和discard的使用方法

  watch是监控一个key，如果这个key中的元素发生了改变，那么这个事务就会被取消

  ![image](\images\posts\Redis\2021-1-26-Redis系列四之Redis的管道、消息订阅、持久化等-19.jpg)

  ![image](\images\posts\Redis\2021-1-26-Redis系列四之Redis的管道、消息订阅、持久化等-20.jpg)

  ![image](\images\posts\Redis\2021-1-26-Redis系列四之Redis的管道、消息订阅、持久化等-21.jpg)

  注：如果watch了一个key的话就像是设下了一个一次性陷阱，当有事务因为这个watch而取消的时候，这个watch就会被自动取消；通过unwatch方法也可以取消watch

  discard的用处就是取消还没有提交的事务，没什么好讲的

  ![image](\images\posts\Redis\2021-1-26-Redis系列四之Redis的管道、消息订阅、持久化等-22.jpg)

### Redis的Bloom过滤器

#### 布隆过滤器的作用

- 现在有一个现实场景，一个电商平台数据库中有商品数据，但是有的用户搜索的数据在数据库中没有，这些搜索操作肯定会击穿缓存，然后在数据库中跑一遍最后一无所获，这些请求对数据库带来压力但是却没有意义，为了解决这个缓存击穿的问题，这里就创造了BloomFilter
- BloomFilter的底层原理是bitmap，BloomFilter可以利用算法将数据库中所有的商品信息映射到bitmap中，假如某个商品在数据库中存在，那么就可以通过算法映射到bitmap上的某三个bit位，将这些bit位从0改为1，这样就可以表示这个商品数据库中存在，当用户搜索商品的时候，就可以先通过bloom过滤器验证，假如这个商品存在的话，那么就放行让他去数据库中搜索，但是假如有一个bit匹配不上，那么就不放行，直接判定为数据库中不存在

#### 布隆过滤器的原理图

![image](\images\posts\Redis\2021-1-26-Redis系列四之Redis的管道、消息订阅、持久化等-23.jpg)

#### 布隆过滤器的使用方法

- 在redis.io上下载BloomFilter的源码，按照README来进行编译，并将编译好的文件放入合适的位置

- 在redis的文件夹下启动redis并且在启动的同时--loadmodule

  ![image](\images\posts\Redis\2021-1-26-Redis系列四之Redis的管道、消息订阅、持久化等-24.jpg)

- 想要使用bloom过滤器很简单，只需要将每一个商品信息通过BF.add方法添加到缓存中，之后只需要依靠BF.exists方法就可以知道该商品是否存在

  ![image](\images\posts\Redis\2021-1-26-Redis系列四之Redis的管道、消息订阅、持久化等-25.jpg)



