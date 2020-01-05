<!-- GFM-TOC -->
* [一、概述](#概述)
* [二、数据类型](#数据类型)
    * [STRING](#string)
    * [HASH](#hash)
    * [LIST](#list)
    * [SET](#set)
    * [ZSet](#ZSet)
* [三、数据结构](#三数据结构)
    * [跳跃表](#跳跃表)
* [四、Guava、Memcached 和 Redis](#四Guava-Memcached-Redis)
* [五、使用场景](#五使用场景)
<!-- GFM-TOC -->

## 一、概述


## 二、数据类型

Redis 的数据架构，可以参考下图：

![Redis 的数据架构](https://github.com/CodeDaShu/JavaNotes/blob/master/img/Redis/Redis-structure.jpg)

Redis中的所有 value 都是以 Object 的形式存在的，其通用结构如下： 

```
typedef struct redisObject {
    unsigned [type] 4;
    unsigned [encoding] 4;
    unsigned [lru] REDIS_LRU_BITS;
    int refcount;
    void *ptr;
} robj;
```

*   type：指类型，String、Hash、List、Set、ZSet；
*   encoding：类型具体的实现方式；比如 Set 是用 hashTable 实现还是 intSet 实现；
*   lru：最后一次被访问的信息，其实一看到 LRU 估计也就和淘汰策略有关；
*   refcount：对象引用计数；
*   ptr：指向实际实现者的地址；

### String

Redis 中的 String 不仅仅表示 字符串，还可以表示 整型、浮点型。

String 的编码可以是 **int、raw 或者 embstr**；单说普通的字符串，就有 raw 和 embstr 两种实现方式，embstr 是 Redis 3.0 新增的数据结构：

字符串长度小于 39 字节，就用 embstr 对象，否则用传统的raw对象（Redis 3.2 版本之后，这里变成了以 44 字节为分界）。

embstr 的优势在于创建时少分配一次空间（RedisObject 和 sds 是连续的），删除时少释放一次空间，以及对象的所有数据连在一起，寻找方便；

当然缺点也非常明显，如果字符串的长度增加，需要重新分配内存的时候，整个 RedisObject 和 sds 都需要重新分配空间。

修改 embstr 对象的时候，Redis 会将其转换成 raw 格式再进行修改，所以 embstr 对象修改之后的对象，一定是 raw 的。

![SDS](https://github.com/CodeDaShu/JavaNotes/blob/master/img/Redis/Redis-String-data-structure.jpg)

应用场景：常规计数都可以使用，可用作缓存、计数、限速等等，比如商品剩余数量，字典表信息，长度不能超过 512MB。

### Hash

Hash 对象的底层实现可以是 **ziplist 或者 hashtable**。

**ziplist：**在这个数据结构中，是按照 key1, value1, key2, value2 这样的顺序存放来存储的；

**hashTable：**是由 dict 这个结构来实现的。（这个结构比较复杂，后面单写一篇来说）

**应用场景：**Hash 适用于存储结构化的对象，可以直接修改这个对象中的某个字段的值；比如用户信息。

### List

List 对象的编码可以是 **ziplist 或者 linkedlist**，从名字上也能看出来两种结构都是啥。

**ziplist：**是一种压缩链表，它存储数据都是连续地放在内存区域当中。

**linkedlist：**是一种双向链表。

**应用场景：**通常网站上的消息列表，可以使用 List 来进行存储；另外 lrange 命令，从某个元素开始，读取多少个元素，可以看做是分页查询，比如很多网站上那种不断下拉，不断分页的效果。

### Set

Set 相对于 List 来说，Set 是可以自动排重的；它的编码可以是 **intset 或者 hashtable **。

**intset：**是一个整数集合，支持三种长度的整数：int16_t、int32_t、int64_t；集合中的数据长度必须是一致的，比如一个 int16_t 长度的 Set，当插入了一条 int32_t 长度的数据，那么所有的数据都会转成 int32_t 长度（不支持降级）。

**hashTable：**对于 Set 来说，hashTable 的 value 永远为 NULL。

**应用场景：**如果要存储一个列表，同时又需要做数据排重的时候，可以使用 set ；另外，Redis 还为 Set 提供了求交集、并集、差集等操作，比如微博上面的【共同关注】这个功能，就可以用 Set 实现。

### ZSet

和 Set 相比，ZSet 增加了一个参数 score，集合中的元素按照 score 进行有序排列。

有序集合的编码可能两种，一种是 ziplist，另一种是 skipList 与 hashTable 的结合。

**ziplist：**和 Hash 类似，元素 和 score 都是按顺序存放的；比较适合用于元素内容不大的场景。

**skipList + hashTable：**是一种添加，移除，更新元素等操作更高效的数据结构，这个跳跃表的数据结构，我近期会发一篇文章单独介绍。

**应用场景：**有序 + 排重的场景，比如经常玩游戏的同学，应该不会陌生各种排行榜，就可以使用 ZSet 来实现。


## 三、数据结构

### 跳跃表

先让我们看一个问题：如果要存一组有序的 int 型数据集合，我们可以如何实现？

1. 数组

可能大多数同学最先想到的是用数据实现，将有序的数据集合存放在数据中，可以使用二分法进行查找，效率比较高，但是对于新增和删除的操作并不友好，因为这些操作都需要移动后面位置的元素。

2. 链表

那么有没有什么数据结构可以让查询、新增、删除操作都变得很快呢？这就是我们今天要介绍的跳跃表了，让我们看几张图，很容易理解。

#### 跳跃表概述
![跳跃表-底层数据链路](https://github.com/CodeDaShu/JavaNotes/blob/master/img/dataStructure/SkipList-1.jpg)

如上图，一个有序的链表，如果要找值为 50 的节点，需要从第一个节点开始遍历，查询最后才能找到值为 50 的节点。

我们给这个链表加一层索引，如图：

![跳跃表-一级索引](https://github.com/CodeDaShu/JavaNotes/blob/master/img/dataStructure/SkipList-2.jpg)

我们按照一级索引来查询（橙色查询路线），可以发现我们至少可以少遍历一半的节点。

还觉得有些慢？，那么再增加一层，再加一层，如图：

![跳跃表-二级索引](https://github.com/CodeDaShu/JavaNotes/blob/master/img/dataStructure/SkipList-3.jpg)

![跳跃表-三级索引](https://github.com/CodeDaShu/JavaNotes/blob/master/img/dataStructure/SkipList-4.jpg)

是不是更快地找到我们需要的节点了，当然这里的节点数量不够多，如果节点数量非常多，查找效率提升会更加明显。

如果需要找中间的某个节点，比如寻找 42 ，过程大概是这样的：

![跳跃表-三级索引-寻找42](https://github.com/CodeDaShu/JavaNotes/blob/master/img/dataStructure/SkipList-5.jpg)

#### 插入节点

看懂了跳跃表的数据结构，那么就很容易理解节点的插入操作了，基本上两步操作就可以实现：在最底层的数据链表中插入数据，然后调整索引；

其中每一层的索引链表中是否需要增加新增的节点，其实并没有什么标准答案，我们尽量做到索引的平均分布即可，常用的就是【随机判断】决定是否需要新增或调整索引，当有新节点插入的时候，通过概率算法判断这个节点需要插入到几级节点中。

比如：

底层数据链表有 N 个元素，随机选择 N/2 个元素作为 1 级索引，随机选择 N/4 个元素作为 2 级索引...一直到顶层索引；

新插入数据节点，1/2 概率不插入任何一级索引，1/4 概率返回需要插入 1 级索引，1/8 概率返回需要插入到 2 级索引，以此类推；

这里要注意一点，插入 2 级索引的时候，同时也需要插入 1 级索引；也就是插入 n 级索引的时候，同时也要插入 1~( n-1 ) 级索引。

![跳跃表-三级索引-插入节点.jpg](https://github.com/CodeDaShu/JavaNotes/blob/master/img/dataStructure/SkipList-6.jpg)

#### 删除节点

跳跃表删除节点就更简单了，删除数据节点，并删除每一层的索引节点（如果有的话）。

总结来说：

1.  可以把跳跃表看成多个有序链表（最底层的数据链表+多层索引链表）；

2.  查找的过程中，从最长层开始查找，找到对应的区间再到下一层查找 ；

3.  每个节点都有两个指针，分别指向右边和下边；

4.  插入新节点时，随机判断该节点是否要插入索引，最高插入几级索引；

5.  插入 n 级索引的时候，同时也要插入 1~(n-1) 级索引；

6.  跳跃表和红黑树等平衡树相比，更容易实现，并且不需要维护平衡性；

7.  Redis 中的 ZSet 的一种实现方式是 skipList 与 hashTable 的结合；

8.  Google 的 LevelDB 、Facebook 的 RocksDB ，它们都是使用了跳跃表这种数据结构。

## 四、Guava、Memcached 和 Redis

### 为什么要使用 Redis
软件架构中引入 Redis ，是因为它“又快又强”。

1. 快，是指性能高

计算机硬件的速度由低到高：硬盘-网络-内存-CPU；

在传统的数据库中，如果第一次访问数据库中的某条数据，通常是比较慢的，因为数据库需要从硬盘上读取数据；而 Redis 中的数据保存在了内存中，所以速度会比从磁盘中读取数据快得多。

所以我们经常把 Redis 当做缓存：第一次从数据库中读取数据，并放入 Redis ，后面直接访问 Redis 就可以了。

2. 强，是指高并发场景下的稳定性（高可用）

在高并发的场景下，Redis 能够承受的访问极限，是远远大于数据库的，所以我们可以考虑把需要高并发读的数据放到 Redis 中；

比如秒杀功能，短短几秒内可能就会有数十万笔的访问，如果直接操作数据库的话，数据库可能瞬间就被击垮了。

### 哪些场景不适合放入 Redis

当然，也不是说所有的场景、所有的数据都适合放进 Redis 中，通常我们需要考虑以下几点：

*   数据查询的命中率高么？如果缓存的命中率很低，没有必要放入到 Redis 中；
*   数据读写操作多么？如果数据会被频繁写入（增、改、删），设置写操作次数大于读操作次数，那么也没有必要使用 Redis ；
*   业务数据大小如何？如果要储存文件，那完全没有必要放入到 Redis 中。

### 本地缓存 or Redis

缓存分为本地缓存和分布式缓存：

1. 本地缓存

比如 Guava、Ehcache，甚至把缓存保存到 Map 中，这些都是本地缓存；

本地缓存的特点是轻量、实现简单，生命周期随着 JVM 的销毁而结束；但是如果程序存在多个实例（程序部署多套），每个实例中的缓存不具有一致性。

2. 分布式缓存

Redis 被称作分布式缓存，如果程序存在多个实例，各个实例可以共用 Redis 中的缓存数据，但同时因为引入了 Redis ，那么需要保证 Redis 的高可用，架构上更为复杂。

### Redis or Memcached

Memcached 也经常被用作缓存，也是分布式缓存的一种，那么它和 Redis 有什么区别呢？

*   Redis 支持更丰富的数据类型，Memcache 支持简单的数据类型String；

*   Redis 支持数据的持久化，可以将内存中的数据保存到硬盘中，重启之后把数据加载到内存中，而 Memcache  只是把数据保存在内存中 ；

*   Redis 目前支持集群模式，而 Memcached 没有原生的集群模式，需要使用方自己实现；

*   Redis 使用单线程的多路 IO 复用模型（Redis 在最新的 6.0 版本中开始支持多线程）；Memcached 使用的是多非阻塞IO复用的网络模型。

![Redis 和 Memcache  的区别](https://github.com/CodeDaShu/JavaNotes/blob/master/img/Redis/Redis-Memcached-2.jpg)

最后再强调一点，**是否要引入 Redis？使用本地缓存还是分布式缓存？都需从项目的实际情况出发**；Redis 丰富的数据类型和对持久化的支持，会更加适合我们的项目。

# 五、使用场景