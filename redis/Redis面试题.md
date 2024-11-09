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



## Redis事务

Redis 可以通过 **`MULTI`，`EXEC`，`DISCARD` 和 `WATCH`** 等命令来实现事务(Transaction)功能。

**Redis 事务提供了一种将多个命令请求打包的功能。然后，再按顺序执行打包的所有命令，并且不会被中途打断。**

你可以通过[`WATCH`](https://redis.io/commands/watch) 命令监听指定的 Key，当调用 `EXEC` 命令执行事务时，如果一个被 `WATCH` 命令监视的 Key 被 **其他客户端/Session** 修改的话，整个事务都不会被执行。

```sql
# 客户端 1
SET PROJECT "RustGuide"
OK
WATCH PROJECT
OK
MULTI
OK
SET PROJECT "JavaGuide"
QUEUED

# 客户端 2
# 在客户端 1 执行 EXEC 命令提交事务之前修改 PROJECT 的值
SET PROJECT "GoGuide"

# 客户端 1
# 修改失败，因为 PROJECT 的值被客户端2修改了
EXEC
(nil)
GET PROJECT
"GoGuide"
```

Redis 事务在运行错误的情况下，除了执行过程中出现错误的命令外，其他命令都能正常执行。并且，Redis 事务是不支持回滚（roll back）操作的。因此，Redis 事务其实是不满足原子性的。

Redis 官网也解释了自己为啥不支持回滚。简单来说就是 Redis 开发者们觉得没必要支持回滚，这样更简单便捷并且性能更好。Redis 开发者觉得即使命令执行错误也应该在开发过程中就被发现而不是生产过程中。

**语法错误**（在事务执行前就可以检测到的错误）

- 如果事务队列中的某个命令存在语法错误，比如拼写错误或参数不正确，Redis 会在 `EXEC` 执行时返回错误，并且 **整个事务不会执行**。
- 这是因为语法错误是在 `MULTI` 阶段检测到的，Redis 在执行事务前已经知道该命令不可执行。

**运行时错误**（在 `EXEC` 阶段检测到的错误）

- 如果事务队列中的命令在 `EXEC` 阶段出现了运行时错误（例如操作一个不存在的 key），Redis **不会回滚**已经成功执行的命令，错误命令会返回异常，后续命令仍然会继续执行。
- 比如一个key是string的，但是你用了hset key a b, 那么这个命令会执行错误，但是后续命令可以继续执行

**如何解决Redis事务的缺陷？**

我们可以利用 Lua 脚本来批量执行多条 Redis 命令，这些 Redis 命令会被提交到 Redis 服务器一次性执行完成，大幅减小了网络开销。

一段 Lua 脚本可以视作一条命令执行，一段 Lua 脚本执行过程中不会有其他脚本或 Redis 命令同时执行，保证了操作不会被其他指令插入或打扰。

不过，如果 Lua 脚本运行时出错并中途结束，出错之后的命令是不会被执行的。并且，出错之前执行的命令是无法被撤销的，无法实现类似关系型数据库执行失败可以回滚的那种原子性效果。因此， **严格来说的话，通过 Lua 脚本来批量执行 Redis 命令实际也是不完全满足原子性的。**

如果想要让 Lua 脚本中的命令全部执行，必须保证语句语法和命令都是对的。

