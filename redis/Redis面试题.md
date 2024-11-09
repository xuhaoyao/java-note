# Redis面试题

## 简单介绍一下Redis

《Redis实战》：起初Redis作者在开发某个应用的时候，使用数据库做缓存，然后不管他怎么优化，QPS都上不去，只能达到几千，（磁盘I/O的瓶颈），所以他就决定开发一款基于内存的数据库，Redis就此诞生，他开发那款应用的QPS也是瞬间达到了几万级别，Redis是一款使用C语言编写的数据库，数据都在内存里面，读写速度非常快，**Redis因此也被广泛应用在缓存方向**。

- Redis除了做缓存之外，也可以拿来做分布式锁，计数器
- **支持事务、持久化**



## 说一下Redis和Memcached的区别

分布式缓存的话，使用的比较多的主要是 **Memcached** 和 **Redis**。

共同点：

- 都是基于内存的数据库，一般都是拿来做缓存。
- 都有数据的过期策略

区别：

- Redis提供的数据类型更加丰富，有string、list、set、zset、hash等,而Memcached只支持最简单的k/v数据类型
- Redis支持持久化，可以将内存中的数据保存在磁盘中，而Memcached把数据全都存在内存里面。
  - 所以可以看出Redis有灾难恢复机制，因为可以把缓存中的数据放到磁盘上。
- Redis服务器在内存使用完了之后可以有对应的数据淘汰策略，而Memcached是直接抛异常
- Redis对过期数据有定期删除和惰性删除两种策略，而Memcached只有惰性删除
- Redis是单线程的I/O多路复用模型，而Memcached是多线程，非阻塞I/O复用模型



## 缓存数据的处理流程

1、用户请求某个数据

2、先从缓存中查，查到了直接返回

3、若缓存中没有，去数据库中查

4、若在数据库中查到了，顺便放入缓存，并把数据返回

5、若没在数据库中查到，那么返回null，需要的话也可以缓存null值

<img src="C:/Users/Administrator/AppData/Roaming/Typora/typora-user-images/image-20220305094941149.png" alt="image-20220305094941149" style="zoom: 67%;" />



## Redis有哪些数据类型？

五种基本的数据类型：

- string、list、set、zset、hash

三种后来引入的比较特殊的：

- bitmaps（位图）、hyperloglog（计数器）、GEO（地理空间）

**对于redis，使用keys * 可以查到目前未过期的所有键,用type key就可以知道对应的key是什么数据类型的，用object encoding key就可以知道这种对应的key的数据结构**

```bash
127.0.0.1:6379> lpush a bb bb bb
(integer) 3
127.0.0.1:6379> type a
list
127.0.0.1:6379> object encoding a
"quicklist"
```



### redisObject

Redis使用对象来表示数据库中的键和值，每次当我们在Redis数据库中新创建一个键值对的时候，都会至少创建两个对象

- 键对象和值对象
- 键对象总是string，值对象就可以是所有数据类型

```c
typedef struct redisObject{
    unsigned type:4;	//这个object是什么类型 type key
    
    unsigned encoding:4; //编码 object encoding key可以查看
    
    void *ptr;		  //指向底层实现数据结构的指针
    
    int refcount;	  //引用计数
    
    unsigned lru:22;  //记录了对象最后一个被命令访问的时间
    
}robj;
```





### string

#### 数据结构：SDS

string是最简单的key-value类型，做缓存一般都用string。Redis没有使用原始的C语言的字符串，而是自己构建了**简单动态字符串（simple dynamic string，SDS）**，将SDS用作Redis的默认字符串表示。

```c
struct sdshdr{
    //记录buf中已使用的字节数量,即可以在O(1)时间获取SDS所保存的字符串的长度
    //而C语言的strlen需要在O(n)时间才能拿到一个字符串的长度
    int len;
    
    //记录buf中未使用的字节数量
    int free;
    
    //字节数组,用来保存字符串
    char buf[];
}
```

- **常数复杂度获取字符串长度**
- **杜绝缓冲区溢出**，每次对SDS的修改，先检查空间是否满足（len+free即可检查）
- **空间预分配**：当SDS的修改需要对SDS进行空间分配
  - 若修改之后SDS的长度小于1MB，程序分配和len属性同样大小的未使用空间，用free记录起来
  - 若大于1MB，会分配1MB的未使用空间
  - 通过空间预分配，减少了修改字符串时带来的内存重分配次数

- **惰性空间释放**：当SDS需要缩小的时候，不立即内存重分配，而是简单的用free字段记录删除的大小，O（1）删除

- **二进制安全**:SDS的API都是二进制安全的，所有SDS的API都会以处理二进制的方式来处理SDS存放在buf数组里的数据



#### string的编码

string的编码有三种，raw, embstr ,int。

- 存的是整数值,就是int
- 字符串长度大于39个字节，就是raw
- 字符串长度小于等于39个字节，就是embstr

**raw:两次内存分配：分配SDS和stringObject**

**embstr:一次内存分配，空间中依次包含redisObject和SDS**

embstr没有修改的API，它是只读的，任何修改操作都会转化成raw



#### 应用场景

缓存、计数（用户的访问次数、热点文章的点赞转发数量）



### list

列表对象编码是quicklist，数据结构就是quicklist是一个双向链表，每个节点有前后指针的quickListNode，每个quickListNode封装了一个ziplist。

应用场景：发布与订阅、消息队列、慢查询



### hash

哈希对象编码可以是ziplist和hashtable。

#### 数据结构：字典

总体思路和JDK1.7的HashMap一样。

```
hash = hashFunction(key)
index = hash & sizemask
然后将这个index放到对应的底层数组里
Redis采用链地址法解决冲突
若index对应的项不为空，说明存在了hash冲突,在这条链上没找到key的话,那么头插法,找到的话,修改key对应的value
```

##### rehash:

- ![image-20241109223239008](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20241109223239008.png)
- ![image-20241109223503607](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20241109223503607.png)

##### 渐进式rehash

扩展或收缩哈希表需要将ht[0]的数据迁移到ht[1]，这个动作不是一次性完成的，因为ht[0]里面可能有很多数据，会影响速度，而是分批次rehash.

![image-20241109223655594](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20241109223655594.png)

![image-20241109223711962](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20241109223711962.png)



## Redis实现排行榜

Redis 中有一个叫做 `Sorted Set` （有序集合）的数据类型经常被用在各种排行榜的场景，比如直播间送礼物的排行榜、朋友圈的微信步数排行榜、王者荣耀中的段位排行榜、话题热度排行榜等等。

相关的一些 Redis 命令: `ZRANGE` (从小到大排序)、 `ZREVRANGE` （从大到小排序）、`ZREVRANK` (指定元素排名)。

在ZSet中，每个元素都会关联一个分数，分数可以重复，但元素不能重复。这使得ZSet非常适合用于实现排行榜等场景。

每个直播间都有粉丝的排行榜，可以通过key+直播间id来作为redis的key。例如broadcast:20210108231。每个直播间的观众按照点赞数排序。则观众刚刚进入直播间即可通过ZADD添加排行榜。

```text
张三 观众进入直播间。
李四 进入直播间
ZADD [key] [score] [value]
```

> ZADD broadcast:20240108231 1 zhangsan
> ZADD broadcast:20240108231 1 lisi

```text
`李四` 送了直播间两颗小红心。李四分值`加2`。
`ZINCRBY [key] increment [member]
```

> ZINCRBY broadcast:20240108231 2 lisi

展示榜单
通过如上指令对直播间分值进行设置之后，得到redis的value如下：

```text
127.0.0.1:6379> ZRANGE broadcast:20240108231 0 -1 WITHSCORES
1) "zhaoliu"
2) "2"
3) "wangwu"
4) "5"
5) "lisi"
6) "8"
7) "zhangsan"
8) "10"
```

查看直播间人数

`ZCARD key` 返回集合数量。

```text
127.0.0.1:6379> zcard  broadcast:20240108231
(integer) 4
```

离开直播间
张三离开直播间，则删除对应key。ZREM [key] [value]

```text
127.0.0.1:6379>  ZREM broadcast:20240108231 zhangsan
(integer) 1
127.0.0.1:6379>  ZREVRANGE broadcast:20240108231 0 2
1) "lisi"
2) "wangwu"
3) "zhaoliu"
```