---
post: layout
title: Redis系列(二)之list、set、sorted set、hash的简单运用
categories: Redis
description: Redis的list、set、sorted set、hash简单运用
keywords: Redis
---

### @list讲解

#### Redis中list的结构

- 在Redis中，list的结构是一个双向链表，并且list的key中还保有两个指针，一个指向list的开头，一个指向list的结尾

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-1.jpg)

#### push和pop

- Redis的list最基础操作有lpush、lpop、rpush、rpop，这些操作看名字就可以知道是什么

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-2.jpg)

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-3.jpg)

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-4.jpg)

- lpushx、rpushx

  如果你想要向一个list中push element，但是希望只有这个list存在才向里面push，如果不存在就不要新建list就可以使用这两个命令

  我们首先清空list，然后向一个不存在的list发起lpushx，然后我们检查这个库中所有的key，发现这个库中没有任何key，说明刚才没有创建任何一个key，接着我们再lpush k1 a，这样这个库中就存在了一个k1，这个k1的type是一个list，里面只有一个元素 ‘a’；接着我们再lpushx k1 b，然后我们再检查k1里面的元素，发现有‘a’，‘b’元素添加成功！

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-5.jpg)

- blpop、brpop

  b的意思是blocking，这个命令的意思是我将会弹出一个元素，如果这个list中没有元素的话，我就会一直等待，直到有元素为止或者我自己设定的等待时间过期，然后再将元素弹出

  我现在启动三个redis-cli -p 6379，来模拟blpop的使用场景，我在二号机和三号机上先后发起 blpop k1 100，这条命令的意思是：订阅 redis 6379上的k1，将最左边的元素弹出，等待时间最长100s；然后我们再在一号机上给k1push元素。我们最后发现，当我们push第一个的时候，先订阅的二号机就输出了结果，三号机继续等待订阅；然后我们再push一个元素进入k1，三号机也得到了这个元素

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-6.jpg)

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-7.jpg)

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-8.jpg)

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-9.jpg)

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-10.jpg)

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-11.jpg)

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-12.jpg)

  假如我们订阅的list到了截至时间都没有返回. 数据的话，那么我们就会返回一个空(nil)

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-13.jpg)

#### lindex、lrange、lset、lrem

- lindex

  lindex的意思是list的index，就是将一个list看作是一个数组，然后通过index查找特定位置的元素，很明显，这个方法也支持正反向索引，在redis中，只要是带索引的基本都支持正反向索引

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-14.jpg)

- lrange

  这个方法我们在上面学习push和pop的时候已经使用过了，这个方法相当于一个范围型的lindex方法，可以一次性得到多个元素，并且标注出下标来

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-15.jpg)

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-16.jpg)

- lset

  lset就是list的set方法，意思是往指定key的指定位置set元素

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-17.jpg)

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-18.jpg)

  如果我们lset的index超过了list的长度，那么会抛出 index out of range异常

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-19.jpg)

  如果我们lset的key不是list或者不存在，那么也会报错

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-20.jpg)

- lrem

  有set就有remove，lrem就是对特定list进行特定元素的remove，这个方法非常有意思

  lrem  key  count  value

  这里面，key指的是指定想要进行删除操作的key；

  count是想要删除的数量，count一共有三种：正数、零、负数，如果是正数，那么就从前往后删除这个数量的某个值，如果是0就不进行删除，如果是负数就从后往前删除这个数量的某个值。

  value是你想要删除哪一个数值

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-21.jpg)

  我们举一个例子，我们有一个list，里面的元素是  a   a   a   a   e   e   e   a   a   a

  我们首先执行命令 lrem k1 3 a，意思是我们要把k1中的a从前往后删除一共3个！得到结果是a   e   e   e   a   a   a

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-22.jpg)

  接下来我们执行命令lrem k1 -4 a，意思是我们要删除k1集合里面的4个a，从后往前删除，得到结果是e   e   e

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-23.jpg)

#### linsert、llen、ltrim

- linsert就是往list中插入元素，可以选择在指定元素的前面或者后面

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-24.jpg)

  要注意一件事情，linsert的时候是默认选择第一个找到的元素，后面的元素是无法进行操作的，我们的例子很好展示了，第一个a才被进行了insert操作，第二个就没有

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-25.jpg)

- llen

  llen就是计算长度，非常简单

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-26.jpg)

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-27.jpg)

- ltrim

  ltrim的用处就是给定一个范围，然后删除这个范围以外两头的元素

  我们给定一个list，然后执行命令  ltrim k1 1 -2，意思是把从index1到index-2的两头全部删掉

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-28.jpg)

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-29.jpg)

- rpoplpush、brpoplpush

  这两个方法见面知意，第二个无非是在第一个的基础上加上了blocking

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-30.jpg)

### @set讲解

#### Redis的set简介

- Redis中的set和Java语言中的set有异曲同工之妙，都是只能存放不重复的元素，并且没有顺序

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-31.jpg)

#### sadd、spop、srem、scard、smove

- sadd

  sadd没什么好说的，就是往set中添加元素

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-32.jpg)

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-33.jpg)

- spop

  spop就是指定key，再指定一个你想要弹出的成员总数，然后set就会随机弹出这个数量的set成员

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-34.jpg)

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-35.jpg)

  注意，如果你输入的数值是负数，那么会抛出index out of range异常；如果你输入的数大于set中元素的总数，那么就会将set中所有剩余的元素全部弹出

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-36.jpg)

- srem

  srem就是移除指定key的指定成员，如果移除成功，返回一个成功移除的总数，如果移除的成员不存在，那么返回一个0

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-37.jpg)

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-38.jpg)

- scard

  scard没什么好讲的，就是查看set中有多少个成员

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-39.jpg)

- smove

  smove也没有什么好说的，就是将一个set中的某个特定成员移动到另外一个set中

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-40.jpg)

  要注意的是，这个方法也是可以创建一个set的

#### sinter、sunion、sdiff

- sinter

  sinter就是选定数个set，并且将这些set全部进行交集操作，将交集得到的结果返回

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-41.jpg)

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-42.jpg)

- sunion

  sunion就是选定数个set进行并集操作，将结果返回

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-43.jpg)

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-44.jpg)

- sdiff

  sdiff的用处就是选定一个set，然后这个set不断减去其他set中与之相同的成员，最后得到结果，sdiff和sinter和sunion的有一个差距，就是sdiff中key的顺序对最后得到的结果有影响

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-45.jpg)

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-46.jpg)

- sinterstore、sunionstore、sdiffstore

  这三个方法和上面的方法非常相似，区别就在于这个前面三个方法会将得到的结果直接返回，但是不会存储；这三个方法不会直接返回结果，但是会将之存储在指定的key中

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-47.jpg)

### @hash

#### Redis的hash简介

- redis的hash是用来放置k-v的，这个hash有点像java中的hashmap

#### hset、hget、hmset、hmget、hkeys、hvals、hgetall、hdel

- hset和hget

  hset就是往key中添加map，请注意，设置的field是成对出现的，并且只能一次性设置一个field；hget需要指定key和field

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-48.jpg)

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-49.jpg)

- hmset和hmget

  hmset和hmget就是multiple set和multiple get

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-50.jpg)

- hkeys

  hkeys就是得到当前hash中所有的field name

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-51.jpg)

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-52.jpg)

- hvals

  hvals就是得到当前hash中所有的field val

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-53.jpg)

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-54.jpg)

- hgetall

  hgetall是得到当前hash中所有的hash，同时得到key和value

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-55.jpg)

- hdel

  hdel就是删除指定key中的指定field

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-56.jpg)

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-57.jpg)

#### hincrby、hincrbyfloat、hstrlen、hexists

- hincrby、hincrbyfloat

  这三个方法都是针对hash中的str类型，和一开始前面的@string中的内容很像，这里讲一个hincr即可

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-58.jpg)

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-59.jpg)

- hstrlen

  没啥好说的，和string的strlen基本没有太大区别

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-60.jpg)

- hexists

  这个没有什么好说的，看名字就知道什么意思

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-61.jpg)

### @sorted_set

#### sorted_set简介

- sorted_set的用处是放置一些带有排序的元素，使用的场景有很多，例如网易云的热度歌曲排行，这些歌曲的热度依靠点击量或者下载量来进行排序，那么就可以使用sorted_set来制作排行榜单
- sorted_set有非常多的方法，我们这里就展示最常用的几个方法

#### zadd

- zadd是sorted_set的添加操作，之所以前面是‘z’是因为‘s’已经被set的操作给使用了，于是就选择了英文字母中的最后一个字母‘z’来作为开头

- zadd在添加元素的时候，是要一起添加score的，用这个score来作为排位的证据，如果score相同的话，就按照value的字典排序

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-62.jpg)

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-63.jpg)

  细节补充：从zadd的介绍中我们可以看到，如果我们插入了两个score不同的同一成员的话，那么我们插入的第二个成员会直接覆盖第一个成员

#### zcount

- zcount的用处是指定一个sorted_set，然后再指定一个score的大小范围，计算出在这个范围内的所有成员的数量

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-64.jpg)

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-65.jpg)

#### zrange、zrevrange

- zrange就是选定一个index区间，然后讲这个区间里面所有的members按照score从小到大输出，后面还可以接参数‘withscores’来显示每一个成员的score

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-66.jpg)

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-67.jpg)

- zrevrange就是选定一个index区间，将这个区间中的所有成员按照score的从大到小排序输出

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-68.jpg)

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-69.jpg)

#### zrangebyscore、zrevrangebyscore

- zrangebyscore的用处是指定一个sorted_set，然后给一个从最小值到最大值的区间，最后返回score在这个区间内的所有成员，这些成员将会按照score从小到大排序，可以附带参数withscores

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-70.jpg)

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-71.jpg)

- zrevrangebyscore的用处是指定一个sorted_set，然后给定一个从最大值到最小值的区间，最后返回在这个区间内的所有成员，这些成员会按照score从大到小的排序，可以附带参数withscores

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-72.jpg)

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-73.jpg)

#### zrank、zscore

- zrank

  zrank的用处是得到某个成员的排序，并且这个排位是按照从小到大排序的，最小的值为0

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-74.jpg)

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-75.jpg)

  注，zrevrank就是将score从大到小排序，score最大的排位0

- zscore

  zscore的用处就是得到某一个成员的score

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-76.jpg)

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-77.jpg)

#### zincrby、zcard

- zincrby

  zincrby的用处就是将某个sorted_set中的某个成员的score值进行改变，注意的是，这个incrby和string中有不同的地方，这里的incrby可以增加double，这和string中的incrbyfloat非常像

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-78.jpg)

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-79.jpg)

- zcard

  zcard就是得到某个sorted_set中的所有成员的数量

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-80.jpg)

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-81.jpg)

  

#### zinterstore、zunionstore

- zinterstore

  zinterstore的用处就是将某些key的成员进行交集操作，交集完毕后得到的成员的score可以根据一开始给key分配的权重来进行  和、选最大、选最小

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-82.jpg)

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-83.jpg)

  使用zinterstore interk 2 math english命令，我们最后可以发现，每个key默认的权值是1，默认的score处理方式是sum

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-84.jpg)

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-85.jpg)

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-86.jpg)

- zunionstore

  zunionstore和zinterstore基本没有太大区别，主要区别在于一个是做交集，一个是做并集

#### sorted_set的数据结构

- 在sorted_set中所有的成员在计算机中的物理存储方式是从小到大排序，不论如何操作，物理的存储方式是不会改变的

- sorted_set的底层原理就是 跳表，以下以图片方式展示sorted_set的底层原理

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-87.jpg)

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-88.jpg)

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-89.jpg)

  ![image](\images\posts\Redis\2021-1-25-Redis系列三之Redis的list、set、sorted set、hash-90.jpg)

  sorted_set底层跳表的探讨

  1. 跳表的查询和插入速度不一定每一个元素的查询和插入都是最快的，但是可以保证平均速度是最快的
  2. 跳表的索引层数不是固定的，索引层数太高也会影响效率
  3. 跳表其实要在sorted_set中存在大量元素的时候才会有更好的效果
  4. 跳表其实就是一种类平衡树

#### 







