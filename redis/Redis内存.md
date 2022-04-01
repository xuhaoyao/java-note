# Redis内存

- redis.conf中搜maxmemory查看，或者`info memory`
- 若没有设置最大内存或者最大内存为0，64位操作系统不限制内存大小，32位操作系统最多使用3GB内存
- 生产环境一般设置内存大小为最大物理内存的四分之三

- 配置文件配置：配置100MB的最大内存如下所示，单位是字节【1024 * 1024 * 100】

![image-20220126000704793](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220126000704793.png)

- 命令行设置：`config set maxmemory xxx`，不是永久配置，下次重启会失效。

![image-20220126001308088](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220126001308088.png)



## 打满内存OOM

故意将最大内存设为1（一个字节），可以看到，还是可以取数据，但是存数据的时候就会报OOM的错误

```bash
127.0.0.1:6379> config set maxmemory 1
OK
127.0.0.1:6379> keys *
1) "goods:001"
2) "good_fail"
127.0.0.1:6379> get good_fail
"31"
127.0.0.1:6379> set k1 v1
(error) OOM command not allowed when used memory > 'maxmemory'.
```

若设置了maxmemory，假如redis内存达到上限，没有加上过期时间就会导致数据写满，为了避免这种情况，就有了内存淘汰策略。



## 内存淘汰策略

**键的过期时间与删除策略**：先看看《Redis设计与实现》100-111的内容

一共有八种策略，如下图

![image-20220126005743944](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220126005743944.png)

**noeviction是默认的淘汰策略**

```java
127.0.0.1:6379> config get maxmemory-policy
1) "maxmemory-policy"
2) "noeviction"
```

**两个维度：**过期键中筛选和所有键中筛选

**四个方面：**LRU,LFU,,ttl

一般要选择淘汰策略的话，选**allkeys-lru**

![image-20220126010235149](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220126010235149.png)

![image-20220126010301912](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220126010301912.png)

- 每一个redisObject都有一个属性为lru,该属性记录了对象最后一个被命令程序访问的时间。

- `obejct idletime`命令可以打印出给定键的空转时长，就是通过将当前系统时间减去对象的lru时间计算得出。
- 如果服务器打开了maxmemory选项，并且服务器用于回收内存的算法为volatile-lru或者allkeys-lru,那么**当服务器占用的内存数超过了maxmemory选项所设置的上限值**的时候，空转时长较高的那部分键就会优先被服务器释放，从而回收内存。



## 如何保证Redis中的数据都是热点数据？

> 相关问题：MySQL 里有 2000w 数据，Redis 中只存 20w 的数据，如何保证 Redis 中的数据都是热点数据?

根据内存淘汰策略，当set值的时候发现内存不够用了，会进行数据淘汰，这时候，如果选用`allkeys-lru`算法，就能做到。

- 1、保证Redis 中的 20w 数据都是热点数据 说明是 被频繁访问的数据，并且要保证Redis的内存能够存放20w数据，要计算出Redis内存的大小。
- 2、假设一个key平均占25个字节，一个value平均占175个字节，那么一个键值对平均占200字节，20w的数据
  - 200 * 20 0000 = 4 * 10^7B，大约是40MB
- 3、`命令行中config set maxmemory ...`或者`配置文件中 maxmemory ...`设置为40MB即可
- 4、这样当内存不足以容纳新写入数据时，在键空间中，移除最近最少使用的key