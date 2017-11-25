---
layout: post
title: Redis深入学习系列（一）
description: Redis作为一种NoSQL数据库，不仅是一种同Memcache类似的数据缓存工具，同时在队列应用、发布订阅、计数器、排行榜、资源锁等方面也有着较好的应用。
category: blog
date: 2017-11-15
---

## 1、Redis应用场景

### Redis缓存
> Redis作为一般的数据缓存，类似于Memcache。作为一种数据缓存，Redis还具有持久化机制，可以定期将内存中的数据持久化到硬盘中，而Memcache断电后数据就会丢失。

### 队列应用
> 非实时业务中，如发放积分或需要削峰降流的秒杀等场景，都会用到队列。

### 发布订阅
> 可以构造具备发布订阅的功能。

### 排行榜
> 利用有序集合实现Redis排行榜，微博的热搜榜就是很好的例子。

### 资源锁
> 秒杀时往往也会用到，防止超卖等现象发生，同时在防止并发风险的同时保证效率性。


## 2、Redis使用

### Redis安装（Windows环境）

- 首先下载release版本，可以msi安装或.zip安装；
- 然后进入bin目录，cmd启动脚本：
	redis-server.exe redis.windows.conf
- 使用Redis，bin目录下运行下面命令
	redis-cli.exe

### 3、Redis数据结构
#### 3.1 数据结构介绍
redis是一个key/value型数据库，其中每个key和value都是使用对象表示的。

比如，执行下面语句，其中key是message是一个包含字符串“message”的对象；而value是一个包含“hello redis”的对象。
    
    redis>set message "hello redis"
    
Redis有5种数据类型：

| 类型常量 | 对象的名称 |
|:--------|---------:|:-------:|
|REDIS_STRING|字符串对象
|REDIS_LIST|列表对象|
|REDIS_HASH|哈希对象|
|REDIS_SET|集合对象|
|REDIS_ZSET|有序集合对象|

#### 3.2 Redis数据结构（Redis Object：ROBJ）

    typedef struct redisObject {
        unsigned type:4;        //类型
        unsigned encoding:4;    // 编码
        unsigned lru:REDIS_LRU_BITS;  // 对象最后一次被访问的时间
        int refcount;           // 引用计数
        void *ptr;           //ptr指向实现对象的数据结构
    }
下面我们会ROBJ的字段进行介绍，其中我比较关注的是类型和编码，能够提供多种数据结构的`构造方式`。

    问题：如何构造不同种类的数据结构？？

##### 1）type对象类型
数据结构中type属性记录了对象的类型，分为string、hash、list、set、zset。type属性占用4bit，值和取值类型字典如下：

    /* Object types */
    #define REDIS_STRING 0
    #define REDIS_LIST 1
    #define REDIS_SET 2
    #define REDIS_ZSET 3
    #define REDIS_HASH 4

##### 2）encoding对象的编码
ptr指针指向对象实现的数据结构，而数据结构采用的编码方式由encoding属性标识。encoding属性占用4bit，其取值和对应类型如下：

    \#define OBJ_ENCODING_RAW 0     / Raw representation /
    \#define OBJ_ENCODING_INT 1     / Encoded as integer /
    \#define OBJ_ENCODING_HT 2      / Encoded as hash table /
    \#define OBJ_ENCODING_ZIPMAP 3  / Encoded as zipmap / // 已废弃
    \#define OBJ_ENCODING_LINKEDLIST 4 / Encoded as regular linked list /
    \#define OBJ_ENCODING_ZIPLIST 5 / Encoded as ziplist /
    \#define OBJ_ENCODING_INTSET 6  / Encoded as intset /
    \#define OBJ_ENCODING_SKIPLIST 7  / Encoded as skiplist /
    \#define OBJ_ENCODING_EMBSTR 8  / Embedded sds string encoding /
    \#define OBJ_ENCODING_QUICKLIST 9 / Encoded as linked list of ziplists /

|编码方式|简单介绍|
|--|--|
| OBJ_ENCODING_RAW | 编码方式使用简单动态字符串来保存字符串对象 |
| OBJ_ENCODING_INT | 编码方式以整数保存字符串数据，仅限能用long类型值表达的字符串 |
| OBJ_ENCODING_EMBSTR编码方式（即embedded string） | 长度小于OBJ_ENCODING_EMBSTR_SIZE_LIMIT的字符串将以EMBSTR方式存储。采用该方式可以减少内存分配的次数，提高内存分配的效率，降低内存碎片率。 |
| OBJ_ENCODING_ZIPLIST（压缩列表） | 在list、hash、zset的元素较少、元素值较小时都会采用ZIPLIST编码方式进行存储。“较少”和“较小”的标准可以进行配置。压缩列表就是一系列连续的内存数据块，其内存利用率很高，但增删改查效率较低。 |
| OBJ_ENCODING_LINKDEDLIST / OBJ_ENCODING_QUICKLIST | 在Redis 3.2版本之前，一般的链表使用LINKDEDLIST编码。Redis 3.2版本开始，所有的链表都是用QUICKLIST编码。两者都是使用基本的双端链表数据结构，区别是QUICKLIST每个节点的值都是使用ZIPLIST进行存储的。 |
| OBJ_ENCODING_INTSET | 整数集合(intset)是集合键的底层实现之一：当一个集合只包含整数值元素， 并且这个集合的元素数量不多时， Redis 就会使用整数集合作为集合键的底层实现。 |
| OBJ_ENCODING_HT | 字典是Redis中存在最广泛的一种数据结构不仅在哈希对象，集合对象和有序结合对象中都有使用，而且Redis所有的Key,Value都是存在db->dict这张字典中的。Redis 的字典使用哈希表作为底层实现。 |


一个哈希表里面可以有多个哈希表节点，而每个哈希表节点就保存了字典中的一个键值对。
跳跃表(SKIPLIST)编码方式为有序集合对象专用，有序集合对象采用了字典+跳跃表的方式实现。


##### 3)访问时间LRU
lru属性（占24bit）表示对象最后一次被访问的时间，根据lru判断对象是否应该被释放。

##### 4）引用计数refcount
C语言并不具备内存回收机制，所以redis通过refcount记录robj共享的次数，当refcount为0是，表示该对象应该被释放，回收内存。

### redis对象的作用
> 1.为5种不同的数据类型提供了统一的表达方式，类似于一个统一的接口；

> 2.通过使用redis对象，针对不同的使用场景，同一种数据类型可以使用不同的数据结构实现，既提升了redis的灵活性，同时可以优化对象在某一场景下的效率。

> 3.采用引用计数和对象共享机制，节约内存

#### 3.2、Hash Table设计
1、在Redis中，所有key-value对都存储在一个hash table中。
> Hash table是一个二维结构。包括一个一维固定长度的数组，每个槽位上保存一个dictEntry对象。key计算hash值后按照这个定长数组求模，结果相同的key-value通过链表保存在同一个槽位上，这样形成一个二维结构。

> Hash table中这个固定长度的数组能够根据key-value数量动态调整大小。

redis Hash table结构：
![redis Hash table-结构](http://i1.bvimg.com/620264/202c28cedf933743.png "redis Hash table结构")

2、Redis哈希表设计思想

> 在Redis中，hash表被称为字典（dictionary），采用了典型的链式解决冲突方法，即：当有多个key/value的key的映射值（每对key/value保存之前，会先通过类似HASH(key) MOD N的方法计算一个值，以便确定其对应的hash table的位置）相同时，会将这些value以单链表的形式保存；同时为了控制哈希表所占内存大小，redis采用了双哈希表(ht[2])结构，并逐步扩大哈希表容量（桶的大小）的策略，即：刚开始，哈希表ht[0]的桶大小为4，哈希表ht[1]的桶大小为0，待冲突严重（redis有一定的判断条件）后，ht[1]中桶的大小增为ht[0]的两倍，并逐步(注意这个词：”逐步”)将哈希表ht[0]中元素迁移（称为“再次Hash”）到ht[1]，待ht[0]中所有元素全部迁移到ht[1]后，再将ht[1]交给ht[0]（这里仅仅是C语言地址交换），之后重复上面的过程。

3、Redis哈希表实现

这里主要是分析Redis的重哈希过程：哈希表在一定情况下会触发rehash 操作，即将哈希表ht[0]的数据逐步转移到ht[1]中。

1）触发条件

当哈希表ht[0]的元素数目大于桶数目，且元素数目与桶数目比值大于5时，Hash表就会扩张为旧表的2倍。

2）转移策略

为了避免一次性转移带来的开销（？？？1.转移期间Redis的使用将会受到影响；2.Redis数据过大时一次性复制效率较低），Redis采用平摊平销的策略，即：将转移代价平摊到每个基本操作中。每一次的添加、更新、检索等操作都会触发一个桶中元素的迁移（如何迁移？迁移多少？）；同时，Redis每隔一个周期会迁移100个桶。

> - 每次对字典执行添加、删除、查找或者更新操作时，程序除了执行指定的操作以外，还会顺带将 ht[0] 哈希表在 rehashidx 索引上的所有键值对 rehash 到 ht[1] ， 并将 rehashidx 的值增一;
> - 直到整个 ht[0] 全部完成 rehash 后，rehashindex设为-1，释放 ht[0] , ht[1]置为 ht[0], 在 ht[1] 创建一个新的空白表。




[redis-hash-table-结构]:http://i1.bvimg.com/620264/202c28cedf933743.png "redis-hash-table-结构"



[Liuwei]:    https://lv093.github.io/Liuwei/  "Liuwei"
