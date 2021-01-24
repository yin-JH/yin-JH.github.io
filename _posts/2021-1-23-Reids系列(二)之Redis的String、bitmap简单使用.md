---
layout: post
title: Reids系列(二)之Redis的String、bitmap简单使用
categories: Redis
description: Redis的String、bitmap使用
keywords: Redis
---

本篇中讲述Redis的基本命令、Redis String的详细讲解以及Redis在现实环境中的一些运用


### Redis的启动

- 当所有的安装都完毕后我们可以启动Redis的Server

  ![image](\images\posts\Redis\2021-1-23-Redis系列二之Redis的String、bitmap简单使用-1.jpg)

- 服务器起来了，我们还需要启动redis的客户端来向Redis的服务器发出请求

  ![image](\images\posts\Redis\2021-1-23-Redis系列二之Redis的String、bitmap简单使用-2.jpg)
  
  我们除了可以默认启动以外，还可以加上一些配置，我们可以查看redis提供的帮助文档，非常的详细和具体
  
  ![image](\images\posts\Redis\2021-1-23-Redis系列二之Redis的String、bitmap简单使用-3.jpg)

### Redis的基本结构和一些简单命令

![image](\images\posts\Redis\2021-1-23-Redis系列二之Redis的String、bitmap简单使用-4.jpg)

- 在Redis的一个进程中，Redis是被分为了16个区域的，每一个区域中的内容互相不共享

  ![image](\images\posts\Redis\2021-1-23-Redis系列二之Redis的String、bitmap简单使用-5.jpg)

### @String讲解

- 讲解String组时，我们可以先查看String的帮助文档

  ![image](\images\posts\Redis\2021-1-23-Redis系列二之Redis的String、bitmap简单使用-6.jpg)

#### set和get

- set命令是向库中插入一个“type”为string的value

- set命令还可以跟nx和xx

  - set key value nx：当这个key不存在时，才可以set成功

    ![image](\images\posts\Redis\2021-1-23-Redis系列二之Redis的String、bitmap简单使用-7.jpg)

  - set key value xx：当这个key存在时，才可以set成功

    ![image](\images\posts\Redis\2021-1-23-Redis系列二之Redis的String、bitmap简单使用-8.jpg)

  - set后面跟nx和xx的运用一般在于一个分布式并发系统上，如果有很多个客户端同时向redis server发起创建某个key的命令，这个时候可以在后面跟上一个nx，确保只有当key不存在的时候才创建

- mget和mset就是set和get的复数操作

  ![image](\images\posts\Redis\2021-1-23-Redis系列二之Redis的String、bitmap简单使用-9.jpg)

- mset后面如果跟上一个nx或者xx，那么一个mset就是原子的

  ![image](\images\posts\Redis\2021-1-23-Redis系列二之Redis的String、bitmap简单使用-10.jpg)

  我们可以发现，当仓库中存在k1这个key的时候，我们使用mset nx会使得创建的三个key都失败，也就是说会被回滚

#### append命令

- append key “value”

  ![image](\images\posts\Redis\2021-1-23-Redis系列二之Redis的String、bitmap简单使用-11.jpg)

  我们可以看到，如果一个key没有的话，我们仍然使用append会重新创建一个key

#### getrange和setrange

- getrange的使用

  ![image](\images\posts\Redis\2021-1-23-Redis系列二之Redis的String、bitmap简单使用-12.jpg)

  ![image](\images\posts\Redis\2021-1-23-Redis系列二之Redis的String、bitmap简单使用-13.jpg)

- serange的使用

  ![image](\images\posts\Redis\2021-1-23-Redis系列二之Redis的String、bitmap简单使用-14.jpg)

  ![image](\images\posts\Redis\2021-1-23-Redis系列二之Redis的String、bitmap简单使用-15.jpg)

  我们发现，如果setrange的长度超过了后面覆盖的长度，那么会额外添加上去；如果set的长度低于后面可覆盖的长度，那么后面没有被覆盖到的会原样不变

#### Redis的正反向索引

- 我们发现，在getrange和setrange的时候要一个一个的数，非常的麻烦，这里提出一个redis作者提供的正反向索引

  ![image](\images\posts\Redis\2021-1-23-Redis系列二之Redis的String、bitmap简单使用-16.jpg)

  我们发现，原本的getrange k1 0 8变成了getrange k1 0 -1，这就是正反向索引

  ![image](\images\posts\Redis\2021-1-23-Redis系列二之Redis的String、bitmap简单使用-17.jpg)

  在Redis中，所有的索引都有正向和反向的

#### strlen与Redis的二进制安全

![image](\images\posts\Redis\2021-1-23-Redis系列二之Redis的String、bitmap简单使用-18.jpg)

![image](\images\posts\Redis\2021-1-23-Redis系列二之Redis的String、bitmap简单使用-19.jpg)

![image](\images\posts\Redis\2021-1-23-Redis系列二之Redis的String、bitmap简单使用-28.jpg)

从上面两张图我们可以看到，在redis中每一个字符都被算做一个单位的长度，虽然redis的string中有的数字是可以进行加减的，但是这些数字仍然被划分成了一个一个的字符，并不是一个整体

特例：

![image](\images\posts\Redis\2021-1-23-Redis系列二之Redis的String、bitmap简单使用-29.jpg)

这个例子很有意思，我们存入了一个中文  ‘尹’  ，这个字符在utf-8的编码下有三个字节，虽然是一个字符，但是却被strlen认定为3，这是因为redis算长度其实算的是字节的长度，数字‘999’为什么会被算成长度3？这是因为每一个‘9’都被切分出来，放在一个字节里面；如果一个字符的长度用一个字节装不下，那么就会导致一个字符有多个字节，进而导致一个字符被算作 >1 的长度

特例延展探讨：

首先，我们重启客户端，并且在启动命令的最后面添加上命令   --raw ，这个命令可以让我们的redis在get   value的时候直接将编码转换为当前客户端的编码，我们可以看到，通过这个方式，我们一开始 set 的 尹不再是一堆的编码了，而是utf-8下被转换的中文    ‘尹’

![image](\images\posts\Redis\2021-1-23-Redis系列二之Redis的String、bitmap简单使用-30.jpg)

接下来，我们改变一下客户端的编码，将之变成gbk，然后我们再次set一个尹，set完毕后我们再get k1和k2，我们发现，一开始在utf-8下面的  ‘尹’  变成了  ‘灏’  。

![image](\images\posts\Redis\2021-1-23-Redis系列二之Redis的String、bitmap简单使用-31.jpg)

![image](\images\posts\Redis\2021-1-23-Redis系列二之Redis的String、bitmap简单使用-32.jpg)

然后，我们再把客户端编码改回来，然后再mget k1 k2

![image](\images\posts\Redis\2021-1-23-Redis系列二之Redis的String、bitmap简单使用-33.jpg)

![image](\images\posts\Redis\2021-1-23-Redis系列二之Redis的String、bitmap简单使用-34.jpg)

我们发现，得到的结果又发生了变化

这是因为，redis只会记录这个字符的二进制编码，并不会记录客户端传来时的编码，也就是说，redis中只会记录一堆的二进制，而怎么编码这个二进制，这个要交给客户端来完成！

这就是Redis的二进制安全，通过这个方式，Redis可以保留最真实的数据，作为一个重要的中间件，这个特性是非常重要的

#### String的type和encoding

- 当我们set一个value的时候，它的type一定是string，但是它的encoding不一定

  ![image](\images\posts\Redis\2021-1-23-Redis系列二之Redis的String、bitmap简单使用-20.jpg)

  我们仔细看一下：我们set的k1是  hello  ，这是一个string type，embstr encoding；

  我们set的k2是  99  ，这是一个string type，int encoding

  type和encoding的区别在哪里？

  1. type是指的类型，整个redis中类型是固定的，分别是string，hashes，lists，sets，sorted sets

  2. encoding是值得其编码，这个是redis作者自己添加的，这样方便了用户使用string

  3. 当用户set key value的时候，这个key下的value就已经固定是String了，不会改变，但是encoding是有可能发生变化的——

     - 当你set的一瞬间，redis就会自动判定你这个value的encoding

       ![image](\images\posts\Redis\2021-1-23-Redis系列二之Redis的String、bitmap简单使用-21.jpg)

     - 每一次对这个value的操作都有可能会使得这个value的encoding发生变化，这些变化会被redis自己记录

       ![image](\images\posts\Redis\2021-1-23-Redis系列二之Redis的String、bitmap简单使用-22.jpg)

- string 的encoding有什么用？

  1. string中提供了很多关于int的操作，这些操作都要求string的encoding是‘int’

     ![image](\images\posts\Redis\2021-1-23-Redis系列二之Redis的String、bitmap简单使用-23.jpg)

     如果这个string的encoding是int，那么就可以使用incr、incrby、decr、decrby这些数值操作命令

     如果这个string的encoding不是int，那么就不行

     ![image](\images\posts\Redis\2021-1-23-Redis系列二之Redis的String、bitmap简单使用-24.jpg)

  2. 通过encoding对string的区分，可以使得redis可用性提高，更加人性化，方便用户，还可以提高效率

  3. 补充一个关于embstr的细节的例子：

     ![image](\images\posts\Redis\2021-1-23-Redis系列二之Redis的String、bitmap简单使用-25.jpg)

     我们可以看到，我们对一个‘int’ encoding的string通过 incrbyfloat的方式变成了一个embstr

     ![image](\images\posts\Redis\2021-1-23-Redis系列二之Redis的String、bitmap简单使用-26.jpg)

     这个时候我们发现，我们仍然可以通过incr来操作这个encoding为embstr的value，这是因为，embstr中也是分为数值和非数值的，redis的encoding中没有double，所以double就被划分到了embstr中

     但是，这种情况仅限于原本一开始是int  encoding的value，通过incrbyfloat变成embstr，如果一开始就是embstr，那么就会出现问题

     ![image](\images\posts\Redis\2021-1-23-Redis系列二之Redis的String、bitmap简单使用-27.jpg)

#### setbit、bitcount、bitpos、bitop

- setbit详解

  - 在redis中，二进制被看成是一串连续的01，我们可以直接操作二进制

    ![image](\images\posts\Redis\2021-1-23-Redis系列二之Redis的String、bitmap简单使用-35.jpg)

    我们可以看到，我们直接setbit k1 1 1，进行二进制操作，此时，我们得到的value其实是：0100 0000

    这串二进制的值是64，可是为什么展示出来是 ‘@’ 呢？

    - 这里补充知识：ascii编码下的64其实就是 ‘@’

      ![image](\images\posts\Redis\2021-1-23-Redis系列二之Redis的String、bitmap简单使用-36.jpg)

    - 关于ascii编码集和其他编码集：

      对于计算机来说，只有两种编码，一种是ascii编码，一种是非ascii编码，ascii编码有二个特点——

      1. 所有的ascii编码只占一个字节
      2. 所有的ascii编码的字节，第一位都是0

      也就是说，所有的ascii的结构都是0XXX XXXX

      当我们有一个程序试图解码一串二进制时，我们首先会拿出第一个字节，看这个字节的第一位bit是不是0，如果是0，那么就将这一个字节当作一个字符，转换成ascii编码对应下的字符然后输出；如果第一位、第二位、第三位都是1，那么就将这个字节后面包括这个字节的一共3个字节拿出来，单独作为一个字符，试图用人们预设好的编码集来得到字符，例如

      我们将先setbit k1 1 1，此时k1是：0100 0000；然后我们再 setbit k1 9 1，此时k1是：0100 0000 0100 0000。redis在解码的时候，首先读取第一个字节，发现是0bit开头，于是将之解析为ascii中的‘@’；接着再读取第二个字节，发现这个字节还是0bit开头，也就是ascii的编码，于是又一次进行解码，将之解释为ascii的‘@’

      ![image](\images\posts\Redis\2021-1-23-Redis系列二之Redis的String、bitmap简单使用-37.jpg)

      接着我们进行setbit k1 0 1，此时k1是：1100 0000 0100 0000，如果redis在试图解码的时候，发现第一个字节的前两位都是1，于是就将这个字节和这个字节后面一个字节一起读出来，试图用客户端的编码，也就是utf-8来解码，最后得到一个乱码

      ![image](\images\posts\Redis\2021-1-23-Redis系列二之Redis的String、bitmap简单使用-38.jpg)

      最后我们再执行一个命令：setbit k1 17 1，此时k1是：1100 0000 0100 0000 0100 0000

      最后我们得到的就是一个乱码和一个@，这是因为，redis首先读取第一个字节，发现第一个字节的开头两位都是1，于是将第一个字节和第二个字节都拿出来，试图用utf-8来解码，解码完成后输出；然后redis又读取第三个字节，发现这个字节的第一位bit是0，于是使用ascii来解码，最后得到‘@’

      ![image](\images\posts\Redis\2021-1-23-Redis系列二之Redis的String、bitmap简单使用-39.jpg)

  - setbit在使用的时候要注意：

    1. setbit的时候，setbit  key  offset  value， offset是指的二进制位的索引，不是字节的索引，value也只有两种，一种0，一种1
    2. 当你setbit的长度超过这个value的长度时，redis会自动补全到这个长度，并且在后面填充0，以达到8的倍数

- bitcount详解

  bitcount是统计某个value在特定字节之间的1bit位的个数

  - 首先，我们setbit一个  0100 0000 1100 0000；我们可以看到，在第一个字节里面，一共有1bit位是1；在第二个字节里面，一共有2bit位是1；在第一个字节到第二个字节里面，一共有3bit位是1，这符合结果

    ![image](\images\posts\Redis\2021-1-23-Redis系列二之Redis的String、bitmap简单使用-40.jpg)

- bitpos详解

  - bitpos就是找到某个指定value的指定范围的字节中第一个出现的bit，可以找第一个1，也可以找第一个0

    ![image](\images\posts\Redis\2021-1-23-Redis系列二之Redis的String、bitmap简单使用-41.jpg)

  - 我们首先设置出一个  0100 0000    0001 0000    1000 0000

  - 执行bitpos  k1  1  0  0：在k1的第一个字节中找到第一个  ‘1’，返回它的二进制索引位置

  - 执行bitpos  k1  1  1  1：在k1的第二个字节中找到第一个  ‘1’，返回它的二进制索引位置

  - 执行bitpos  k1  1  2  2：在k1的第三个字节中找到第一个  ‘1’，返回它的二进制索引位置

  - 执行bitpos  k1  1  0  -1：在k1的第一个字节到最后一个字节中找到第一个  ‘1’，返回它的二进制索引位置

  - 执行bitpos  k1    0  -1：在k1的第一个字节到最后一个字节中找到第一个  ‘2’，返回它的二进制索引位置，因为二进制只有0和1，所以直接报错

    ![image](\images\posts\Redis\2021-1-23-Redis系列二之Redis的String、bitmap简单使用-42.jpg)

- bitop详解

  - bitop就是选定两个key，将这两个key中的二进制要么进行与操作，要么进行或操作，将操作的结果放到一个指定的key中

    ![image](\images\posts\Redis\2021-1-23-Redis系列二之Redis的String、bitmap简单使用-44.jpg)

  - 我们先得到两个key

    k1：0100 0000

    k2：0000 0001

    然后我们让这k1和k2进行或操作，将得到的结果放到k3，这样，k3就是：0100 0001，转换成ascii就是‘A’

    ![image](\images\posts\Redis\2021-1-23-Redis系列二之Redis的String、bitmap简单使用-43.jpg)



### Redis现实的使用场景

#### 记录不同用户的每日登陆情况

现在有一个场景，天猫需要统计所有用户全年的登陆情况，方便🐎安排任务，比如：在双11前7天一直登陆淘宝的用户可以得到奖励；双12后12天一直登陆淘宝的用户可以得到优惠券；他老婆生日当天登陆的用户可以得到优惠券。这个时候，如果我们使用传统的mysql进行统计的话，那么我们就需要专门建表，这个表里面放每个用户的登陆情况，如果用户id是4个字节，日期记录也是4个字节，那么如果想要记录每一个用户每天的登陆就需要 8 * 年数（366） * 用户数，那么这个空间代价是相当昂贵的，我们假设一共有5亿用户，那么我们就需要开辟 （8 * 10 * 5e）/  1024 / 1024 /1024=  1,396,179T  的存储空间来存储，这个代价实在是太昂贵了；但是如果我们使用redis来记录，代价会大大降低

假如我们有一个用户 yin，当这个用户新年第一天登陆淘宝的时候，我们就可以：setbit yin 0 1；假如yin在新年的第80天后登陆了淘宝，我们就可以：setbit yin 79 1。假设一年有366天，那么我们只需要  366 / 8 = 45.75M就可以存储一个用户一年的登陆情况，5e客户只需要22,875,000,000M，大约21,815T，远远小于1,396,179T

而且二进制的操作速度非常快，远远高于访问mysql然后遍历库计算得到结果来的快

#### 记录用户的活跃情况

现在有一个场景，Steam要找到十二月份每天都活跃的用户，给他赠送一款游戏，我们可以通过redis来轻松解决，只要用日期作为key，给不同的用户分配不同的bit位，假如用户yin是bit位3，如果想要表示yin在1月23日登陆了Steam，那么我只需要 setbit  2021-1-23  2  1，这样就可以表示yin在1月23日登陆了Steam，当需要统计月活跃用户的时候，只需要把这个月每一天都依次进行一遍：bitop and andResultKey key1 key2，就可以得到每天都活跃的用户，代价远低于mysql遍历累加的操作

#### 购物平台的秒杀环节

京东有一个冰箱正处在秒杀阶段，一共10000台，有可能在2s内就被抢完，这样光是这一个冰箱购买接口就会造成大约5000的qps，这个时候就可以使用redis，每一个人抢购，就向redis服务器发起一个decr命令，这样就可以避免高并发环境下对mysql操作导致的触发事务，从而使得秒杀环节会变得非常慢，影响效率

