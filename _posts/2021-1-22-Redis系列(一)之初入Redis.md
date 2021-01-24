---
layout: post
title: Redis系列一之初识Redis、Redis的安装及Redis学习的基础知识
categories: Redis, Linux
description: 学习Redis的基础前沿知识
keywords: Redis, Linux, memcached
---

Redis开始前的碎碎念：Redis作为一个非常重要的知识，几乎可以说是每一个梦想成为架构师的小白都必须摸透的缓存技术，我的Redis系列不仅仅是将以前的笔记搬运，而是重新按照当年学习的流程再次重新学习一遍，毕竟是上学期考试前学完的，为了准备期末考试草草了结Redis，搞的现在忘了很多，必须重新捡起来
"\n\n"

### 基础常识和Redis前沿知识

#### 计算机磁盘和内存的速度

- 计算机磁盘的寻址速度一般是毫秒(ms)级别，网络带宽一般是G、百M级别的
- 计算机内存的寻址速度一般是纳秒(ns)级别，1s=1000ms=1000000um=1000000000ns
- 结论：计算机磁盘在寻址上的速度仅仅为内存的1/100000

#### 操作系统在读取磁盘数据的细节

我们都知道，磁盘上是有磁道和扇区的，一个扇区的大小是512B，但是操作系统实际上在从磁盘中读取数据的时候并不是按照一个扇区一个扇区读写的，如果这样的话，那么磁盘需要维护的寻址索引就会非常大，毕竟需要能够表示每一个扇区可能的偏移位置；所以操作系统每一次读取磁盘数据的时候，最低都是4k为单位读取。

#### 数据库是如何管理磁盘数据的来提升搜索速度的

1. 在数据库中，也有最小单位的说法，也就是data page，在数据库中，一个data page的大小正好是4k，正好与操作系统读取的大小匹配

2. 数据库为了提升检索速度，采用了索引技术

   ![image](\images\posts\Redis\2021-1-22-Redis系列一之初识Redis-1.jpg)

   - 我们假设一个场景，现在这是关系型数据库，存储的是员工编号，真实的数据在磁盘中是顺序存储的
   - 假如我们现在需要查询一个员工编号为114514的人，假如我们没有索引，那么我们就有可能是从1开始遍历，直到找到114514，这样效率极其低下；但是如果我们有了索引之后，我们就有了一个B + 树，我们就可以通过类似二分法的方式快速的得到数据存储地址块，将这个地址块读到内存中后再从中读取到真实数据的物理地址，从而快速读取数据
   - 数据库是将真实数据和数据的地址放置在磁盘中，将构建好的B+T放置在内存中，这样既可以通过内存速度快的优势快速定位数据，又可以利用磁盘存储空间大的优势来存储大量数据

3. 延展问题：为什么数据库的表越大，数据库操作越慢？

   1. 影响数据库增删改查的重要因素是数据库的索引维护，数据库表越大，维护索引的代价也就越大，因为我们有可能会在B+树中插入分支，这会使得B+树的结构发生变化，所以增删改会变慢
   2. 如果仅仅只是一次或少量查询，那么速度影响不大；如果是大量查询，那么会受到磁盘的带宽和IO瓶颈限制

#### 如何提升数据库的查询速度？

- 解决方案一：将整个数据库迁移到内存中去
  - 典型的数据库就是SAP的HANA数据库，这个数据库是整个构建在内存中的，所以它的速度极快，但是相应的，内存的价格奇高，如果想要搭建一个HANA数据库，那么HANA公司提供的一台拥有2T内存的服务器配合一系列的硬件软件价格可以达到2亿
  - 很明显，这个方案虽然速度快，但是价格太高，很多企业无法负担，这个时候就有了折中方案
- 解决方案二：缓存技术
  - 缓存技术就是将数据磁盘中的数据库中的一些数据迁移到内存中去，这些被迁移的数据往往是“热数据”，是那些访问频率极高的数据，这样就可以让数据库享受到内存级别的速度，使得查询速度大大提高
  - 缓存技术的典型就是当下非常火热的Redis技术

#### 初识Redis

- Redis的特点：
  - Redis 是一个开源（BSD许可）的，内存中的数据结构存储系统，它可以用作数据库、缓存和消息中间件。 它支持多种类型的数据结构，如 [字符串（strings）](http://www.redis.cn/topics/data-types-intro.html#strings)， [散列（hashes）](http://www.redis.cn/topics/data-types-intro.html#hashes)， [列表（lists）](http://www.redis.cn/topics/data-types-intro.html#lists)， [集合（sets）](http://www.redis.cn/topics/data-types-intro.html#sets)， [有序集合（sorted sets）](http://www.redis.cn/topics/data-types-intro.html#sorted-sets) 与范围查询， [bitmaps](http://www.redis.cn/topics/data-types-intro.html#bitmaps)， [hyperloglogs](http://www.redis.cn/topics/data-types-intro.html#hyperloglogs) 和 [地理空间（geospatial）](http://www.redis.cn/commands/geoadd.html) 索引半径查询。 Redis 内置了 [复制（replication）](http://www.redis.cn/topics/replication.html)，[LUA脚本（Lua scripting）](http://www.redis.cn/commands/eval.html)， [LRU驱动事件（LRU eviction）](http://www.redis.cn/topics/lru-cache.html)，[事务（transactions）](http://www.redis.cn/topics/transactions.html) 和不同级别的 [磁盘持久化（persistence）](http://www.redis.cn/topics/persistence.html)， 并通过 [Redis哨兵（Sentinel）](http://www.redis.cn/topics/sentinel.html)和自动 [分区（Cluster）](http://www.redis.cn/topics/cluster-tutorial.html)提供高可用性（high availability）。
- Redis学习网站：http://www.redis.cn/
- Redis和memcached的比较
  - 说起缓存技术，有两个值得说的，一个是Redis，还有一个是memcached
  - 这两个缓存技术从本质上来说，这两者都是基于K-V键值对存储的非关系型数据库，其实差距不大，但是Redis却在蚕食memcached的市场份额。
  - memcached和Redis的区别就在于，Redis的V带有数据类型，而memcached没有
  - 其实就算memcached的V没有数据类型，但是其实影响也不是特别大，因为V类型可以存储一个json，这样也照样可以存储字符串、对象、数组
  - Redis打败memcached的关键在于Redis对每一个数据类型都有自己单独的方法，这些方法可以使得client在向Redis发起请求的时候，Redis可以自己对数据进行简单的处理，之后再返回处理完毕的结果，例如自增、自减、截取字符串中的某些字符等等；但是memcached就不行，你必须自己先把完整数据拿到手，然后再自己将之读取出来，处理之后才可以，影响了效率
  - 这就是Redis相对于memcached的优势：计算向数据移动

### Redis安装

#### Redis安装流程

1. 首先，在root目录下面建立一个soft文件夹，用于存放redis

   ![image](\images\posts\Redis\2021-1-22-Redis系列一之初识Redis-2.jpg)

2. 使用wget命令下载redis

   ![image](\images\posts\Redis\2021-1-22-Redis系列一之初识Redis-3.jpg)

3. 下载、解压完毕后进入里面，首先应该大致浏览一下README文件，然后按照README的流程来进行操作

   ![image](\images\posts\Redis\2021-1-22-Redis系列一之初识Redis-4.jpg)

4. 编译完成，开始启动redis

   ![image](\images\posts\Redis\2021-1-22-Redis系列一之初识Redis-5.jpg)

   ![image](\images\posts\Redis\2021-1-22-Redis系列一之初识Redis-6.jpg)

5. 这个时候我们就已经启动了Redis了，但是这种启动方式很不方便，为了简单，我们将Redis变成一个开机自启动的服务

   ![image](\images\posts\Redis\2021-1-22-Redis系列一之初识Redis-9.jpg)

   首先安装，并且选择安装目录

   ![image](\images\posts\Redis\2021-1-22-Redis系列一之初识Redis-8.jpg)

   安装完毕后将刚才安装的路径添加成REDIS_HOME，添加好环境变量

   ![image](\images\posts\Redis\2021-1-22-Redis系列一之初识Redis-7.jpg)

   完毕后到根目录下的util下进行安装，将Redis变成一个开机自启动的程序

   ![image](\images\posts\Redis\2021-1-22-Redis系列一之初识Redis-10.jpg)

   以上是安装时候出现的各种信息内容

   ![image](\images\posts\Redis\2021-1-22-Redis系列一之初识Redis-11.jpg)

   安装成功
