# Redis分布式锁

## 配置

### pom文件

```xml
2.3.3.RELEASE
	<dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-actuator -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>

        <!-- https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-data-redis -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>

        <!-- https://mvnrepository.com/artifact/org.apache.commons/commons-pool2 -->
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-pool2</artifactId>
        </dependency>

        <!-- https://mvnrepository.com/artifact/redis.clients/jedis -->
        <dependency>
            <groupId>redis.clients</groupId>
            <artifactId>jedis</artifactId>
            <version>3.1.0</version>
        </dependency>

        <!-- https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-aop -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-aop</artifactId>
        </dependency>

        <!-- https://mvnrepository.com/artifact/org.redisson/redisson -->
        <dependency>
            <groupId>org.redisson</groupId>
            <artifactId>redisson</artifactId>
            <version>3.13.4</version>
        </dependency>


        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>

        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.12</version>
        </dependency>
    </dependencies>
```

### 配置文件

两个引用，就这里不同，一个端口号是1111，一个是2222

```bash
server.port=2222  

spring.redis.database=0
spring.redis.host=192.168.200.132
spring.redis.port=6379
#连接池最大连接数（使用负值表示没有限制）默认8
spring.redis.lettuce.pool.max-active=8
#连接池最大阻塞等待时间（使用负值表示没有限制）默认-1
spring.redis.lettuce.pool.max-wait=-1
#连接池中的最大空闲连接默认8
spring.redis.lettuce.pool.max-idle=8
#连接池中的最小空闲连接默认0
spring.redis.lettuce.pool.min-idle=0
```

### 配置类

```java
@Configuration
public class RedisConfig {

    @Bean
    public RedisTemplate<String, Serializable> redisTemplate(LettuceConnectionFactory connectionFactory){
        // 新建 RedisTemplate 对象，key 为 String 对象，value 为 Serializable（可序列化的）对象
        RedisTemplate<String, Serializable> redisTemplate = new RedisTemplate<>();
        // key 值使用字符串序列化器
        redisTemplate.setKeySerializer(new StringRedisSerializer());
        // value 值使用 json 序列化器
        redisTemplate.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        // 传入连接工厂
        redisTemplate.setConnectionFactory(connectionFactory);
        // 返回 redisTemplate 对象
        return redisTemplate;
    }

}
```

### nginx配置

```bash
upstream redisBoot {
    server localhost:1111 weight=1;
    server localhost:2222 weight=1;
}

server {
    listen       80;
    server_name  localhost;

    location / {
       proxy_pass http://redisBoot;
       index index.html index.htm;
       #root   html;
    }
}
```



## 单机锁例子分析

- synchronized ，没抢到锁就一直等待
- ReentrantLock，若有抢锁的时间要求，可以尝试tryLock，一定时间内没抢到就放弃。
- 用哪个看具体要求。
- 但是单机锁**只能锁住一个对象**，分布式下，有多台机器，它们锁的是不同对象，肯定会有问题

```java
@RestController
public class GoodController {

    @Autowired
    StringRedisTemplate stringRedisTemplate;

    @Value("${server.port}")
    private String serverPort;

    @GetMapping("/buy_goods")
    public String buy_Goods(){
        synchronized (this) {
            String result = stringRedisTemplate.opsForValue().get("goods:001");
            int goodNumber = result == null ? 0 : Integer.parseInt(result);
            String info = null;
            if(goodNumber > 0){
                int realNumber = goodNumber - 1;
                stringRedisTemplate.opsForValue().set("goods:001",String.valueOf(realNumber));
                info = "成功买到商品,库存还剩下" + realNumber + "件\t服务提供端口:" + serverPort;
            }
            else{
                info = "商品售罄...\t服务提供端口:" + serverPort;
            }
            System.out.println(info);
            return info;
        }

}

```

### jmeter压测

1秒内100个线程

![image-20220124230015979](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220124230015979.png)

http请求

![image-20220124230241117](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220124230241117.png)

100个商品

![image-20220124230220613](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220124230220613.png)

测试结果如下：

100个商品。模拟100个用户一起抢，结果是每次用户都抢到了，但是商品还有27件？**说明出现了超卖，有些用户买到了一样的商品。**

![image-20220124230303457](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220124230303457.png)

## Redis的setnx例子分析 --- key过期的问题

```java
public static final String REDIS_LOCK = "good_lock";
@GetMapping("/buy_goods")
public String buy_Goods(){
    String value = UUID.randomUUID().toString() + Thread.currentThread().getName();
    try {
        //设置key的过期时间,防止宕机之后无法删除key
        //必须在setnx中设置过期时间,保证原子性,若用setnx + expire的话还是会出现宕机导致无法删除key的问题
        Boolean flag = stringRedisTemplate.opsForValue().setIfAbsent(REDIS_LOCK, value,5L, TimeUnit.SECONDS);  //setnx
        if(!flag) return "抢购失败";
        String result = stringRedisTemplate.opsForValue().get("goods:001");
        int goodNumber = result == null ? 0 : Integer.parseInt(result);
        String info = null;
        if(goodNumber > 0){
            int realNumber = goodNumber - 1;
            stringRedisTemplate.opsForValue().set("goods:001",String.valueOf(realNumber));
            info = "成功买到商品,库存还剩下" + realNumber + "件\t服务提供端口:" + serverPort;
        }
        else{
            info = "商品售罄...\t服务提供端口:" + serverPort;
        }
        System.out.println(info);
        return info;
    }
    finally {
        //出异常的话,可能无法释放锁，必须要在代码层面finally释放锁
        stringRedisTemplate.delete(REDIS_LOCK);
    }
}
```

​	若业务一切正常的话，基本上可以锁住所有机器了。但是考虑这种情况，线程A执行5秒还没有完成，这时候key就过期了！B线程又setnx成功，这时候key是B拿到的，但是A过了一会执行完成，是会执行`del key`把key删除的，这时候又出现了问题。

![image-20220125000457430](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220125000457430.png)

**如何解决乱删除的问题？自己的key只有自己能删**

```java
String value = UUID.randomUUID().toString() + Thread.currentThread().getName();
if(value.equals(stringRedisTemplate.opsForValue().get("goods:001"))) {
    //出异常的话,可能无法释放锁，必须要在代码层面finally释放锁
    stringRedisTemplate.delete(REDIS_LOCK);
}
```

**但是还有一个问题，假如判断的时候，key还是自己的，返回了true，在这个时刻，key就过期了，又有另一个线程设置了key，这时候又删错了。因此必须保证判断和删除的原子性！**

**redis里面，多操作的原子性，可以考虑lua脚本（推荐）和redis事务**



## lua脚本例子分析(推荐)

https://redis.io/commands/set : 官网给了lua脚本。

An example of unlock script would be similar to the following:

```lua
if redis.call("get",KEYS[1]) == ARGV[1]
then
    return redis.call("del",KEYS[1])
else
    return 0
end
```

```java
    public static final String REDIS_LOCK = "good_lock";

    public static final String REDIS_FAIL = "good_fail";

    @GetMapping("/buy_goods")
    public String buy_Goods(){
        String value = UUID.randomUUID().toString() + Thread.currentThread().getName();
        try {
            //设置key的过期时间,防止宕机之后无法删除key
            //必须在setnx中设置过期时间,保证原子性,若用setnx + expire的话还是会出现宕机导致无法删除key的问题
            Boolean flag = stringRedisTemplate.opsForValue().setIfAbsent(REDIS_LOCK, value,5L, TimeUnit.SECONDS);  //setnx
            if(!flag) {
                stringRedisTemplate.opsForValue().increment(REDIS_FAIL);
                return "抢购失败";
            }
            String result = stringRedisTemplate.opsForValue().get("goods:001");
            int goodNumber = result == null ? 0 : Integer.parseInt(result);
            String info = null;
            if(goodNumber > 0){
                int realNumber = goodNumber - 1;
                stringRedisTemplate.opsForValue().set("goods:001",String.valueOf(realNumber));
                info = "成功买到商品,库存还剩下" + realNumber + "件\t服务提供端口:" + serverPort;
            }
            else{
                info = "商品售罄...\t服务提供端口:" + serverPort;
            }
            System.out.println(info);
            return info;
        }
        finally {
            String script = "if redis.call(\"get\",KEYS[1]) == ARGV[1]\n" +
                    "then\n" +
                    "    return redis.call(\"del\",KEYS[1])\n" +
                    "else\n" +
                    "    return 0\n" +
                    "end";
            //lua脚本 -> 获取值成功+对比成功删除 -> 原子删除
            stringRedisTemplate.execute(
                new DefaultRedisScript<>(script,Long.class) , Arrays.asList(REDIS_LOCK),value);
        }
    }
```

注意，记得指定返回值，Long.class，不然会抛异常`lettuce.core.RedisException: java.lang.IllegalStateException] with root cause`

### 测试

还是100个线程1秒内发请求。这里加了一个，抢购失败的计数器。可以看到，100个商品，还剩下37个，说明100个线程有37个抢失败了，计数器的值也是正确的。

![image-20220125110447817](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220125110447817.png)



## Redis事务例子分析(lua还未出现时的做法)

先看下相关概念

![image-20220125001849661](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220125001849661.png)

![image-20220125002120778](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220125002120778.png)

![image-20220125002108004](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220125002108004.png)

会话1执行下面操作

```java
127.0.0.1:6379> watch k1
OK
127.0.0.1:6379> multi
OK
127.0.0.1:6379(TX)> set k1 v111
QUEUED
127.0.0.1:6379(TX)> set k2 v222
QUEUED
127.0.0.1:6379(TX)> exec   在exec执行前,会话2修改了k1
(nil)
127.0.0.1:6379> unwatch
OK
```

会话2执行下面操作

```bash
127.0.0.1:6379> set k1 v2222
OK
```

WATCH命令是一个乐观锁，它可以在EXEC命令执行之前，监视任意数量的数据库键，并在EXEC命令执行时，检查被监视的键是否至少有一个已经被修改过了，如果有的话，服务器拒绝执行事务，并向客户端返回代表事务执行失败的空回复。

### watch底层原理

每个redis客户端都有一个状态属性flags,而每个redis服务器都有一个`dict* watched_keys,保存每个客户端正在被监视的key`对数据库修改的所有命令【包括这个key过期，expire】，执行完之后都会对这个字典检查，看看是否有客户端正在监视被修改的key，如果有的话，修改客户端的状态

```python
def touchWatchKey(db,key):
    # 如果key存在于数据库的watched_keys字典中
    # 那么说明至少有一个客户端在监视这个Key
    if key in db.watched_keys:
        #遍历所有监视key的客户端
        for client in db.watched_keys:
            #打开标识
            client.flags |= REDIS_DIRTY_CAS
```

### redis事务解决分布式锁的问题

```java
    public static final String REDIS_LOCK = "good_lock";

    public static final String REDIS_FAIL = "good_fail";

    @GetMapping("/buy_goods")
    public String buy_Goods(){
        String value = UUID.randomUUID().toString() + Thread.currentThread().getName();
        try {
            //设置key的过期时间,防止宕机之后无法删除key
            //必须在setnx中设置过期时间,保证原子性,若用setnx + expire的话还是会出现宕机导致无法删除key的问题
            Boolean flag = stringRedisTemplate.opsForValue().setIfAbsent(REDIS_LOCK, value,5L, TimeUnit.SECONDS);  //setnx
            if(!flag) {
                stringRedisTemplate.opsForValue().increment(REDIS_FAIL);
                return "抢购失败";
            }
            String result = stringRedisTemplate.opsForValue().get("goods:001");
            int goodNumber = result == null ? 0 : Integer.parseInt(result);
            String info = null;
            if(goodNumber > 0){
                int realNumber = goodNumber - 1;
                stringRedisTemplate.opsForValue().set("goods:001",String.valueOf(realNumber));
                info = "成功买到商品,库存还剩下" + realNumber + "件\t服务提供端口:" + serverPort;
            }
            else{
                info = "商品售罄...\t服务提供端口:" + serverPort;
            }
            System.out.println(info);
            return info;
        }
        finally {
            while (true) {
                stringRedisTemplate.watch(REDIS_LOCK);
                if (value.equals(stringRedisTemplate.opsForValue().get(REDIS_LOCK))) {
                    stringRedisTemplate.setEnableTransactionSupport(true);
                    stringRedisTemplate.multi();
                    //出异常的话,可能无法释放锁，必须要在代码层面finally释放锁
                    stringRedisTemplate.delete(REDIS_LOCK);
                    List<Object> result = stringRedisTemplate.exec();
                    if(result.isEmpty()){  //事务执行失败底层会返回null,然后new 一个EmptyList返回
                        continue;
                    }
                }
                stringRedisTemplate.unwatch();
                break;
            }
        }
    }
```

- 这段代码可以参考《Redis实战》121页。
- A：看最糟糕的情况，watch 之前 key就过期了，它watch的是别人的，那么if条件不成立，unwatch然后结束
- B：若进入了if条件之后key才过期，一过期，watch就检查到了，事务执行必然返回null，那么继续while循环，就变成了情况A
- C：一切正常，if进入，delete成功的返回OK，然后unwatch，break，正常结束。

#### 测试

还是100个线程1秒内发请求。这里加了一个，抢购失败的计数器。可以看到，100个商品，还剩下21个，说明100个线程有21个抢失败了，计数器的值也是正确的。

![image-20220125105218695](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220125105218695.png)





## 对比事务和lua

lua是Redis2.6之后才出现的，再次之前，解锁的代码都是通过Redis事务的那种方式来实现的，当lua出现后，就用lua的写法代替了事务的写法，在《Redis实战》中256页有测试，Lua版本的锁实现性能比事务的要快很多。



## Redis分布式锁如何续期的问题

要确保锁的时间大于业务执行时间才行。

### 集群CAP对比Zookeeper

**redis**

- redis 异步复制造成的锁丢失， 比如：主节点没来的及把刚刚 set 进来这条数据给从节点，就挂了，此时从节点上位，成为新的master，但是它是没有set的那个数据的，那么主节点和从节点的数据就不一致。此时如果集群模式下，就得上 Redisson 来解决

**zookeeper**

- zookeeper 保持强一致性原则，对于集群中所有节点来说，要么同时更新成功，要么失败，因此使用 zookeeper 集群并不存在主从节点数据丢失的问题，但丢失了速度方面的性能



## Redisson

- 官网原话：This page is an attempt to provide a more canonical（规范） algorithm to implement distributed locks with Redis. We propose an algorithm, called **Redlock**
- Redisson是Redlock算法的具体实现

```java
    @Bean
    public Redisson redisson(){
        Config config = new Config();
        config.useSingleServer().setAddress("redis://192.168.200.132:6379").setDatabase(0);
        return (Redisson) Redisson.create(config);
    }

    @Autowired
    private Redisson redisson;

    @GetMapping("/buy_goods")
    public String buy_Goods(){
        RLock lock = redisson.getLock(REDIS_LOCK);
        lock.lock();
        String info = null;
        try {
            String result = stringRedisTemplate.opsForValue().get("goods:001");
            int goodNumber = result == null ? 0 : Integer.parseInt(result);
            if(goodNumber > 0){
                int realNumber = goodNumber - 1;
                stringRedisTemplate.opsForValue().set("goods:001",String.valueOf(realNumber));
                info = "成功买到商品,库存还剩下" + realNumber + "件\t服务提供端口:" + serverPort;
            }
            else{
                info = "商品售罄...\t服务提供端口:" + serverPort;
            }
            System.out.println(info);
            return info;
        }
        finally {
            //标准写法
            if(lock.isLocked() && lock.isHeldByCurrentThread())
                lock.unlock();
        }
    }
```

**在finally解锁之前,要判断这个锁是不是自己的，否则极小概率出现下面的异常**

****

![image-20220125113719544](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220125113719544.png)



### watchDog:看门狗及最佳写法

- `lock.lock(30L,TimeUnit.SECONDS);`如果传递了锁的超时时间，就给redis发送lua脚本，进行占锁，默认超时时间就是指定的时间

- `lock.lock();`如果未指定，就使用看门狗的默认时间`this.lockWatchdogTimeout = 30000L;`【30秒】

  - 只要占锁成功，就启动一个定时任务`TimerTask`【重新给锁设置过期时间，就是锁续期，新的过期时间就是看门狗的默认时间】
  - 定时任务每隔`this.internalLockLeaseTime【看门狗时间】 / 3L`就进行续期
    - 比如看门狗时间是30s,那么执行了10s之后进行续期，又续到30s
    - 每隔1/3个看门狗时间都会再次续期，时间续满。

- 一般都可以不用看门狗，锁会过期【超过了业务时间】，我们让他不过期就好了【设置过期时间大一些，此时间点业务绝对可以执行完毕，否则就是有异常了，需要排查】

- 因此使用Redisson推荐，每次都指定过期时间

  - 最佳实战`lock.lock(30L,TimeUnit.SECONDS);`，手动解锁

  - ```java
    //标准写法如下
    RLock lock = redisson.getLock(REDIS_LOCK);
    lock.lock(30L,TimeUnit.SECONDS);
    try{
        serviceMethod();
    }
    finally{
        if(lock.isLocked() && lock.isHeldByCurrentThread())
            lock.unlock();
    }
    ```





## 例子总结

- Synchronized单机版ok,此时上分布式
- nginx分布式微服务，单机锁不行了
- 取消单机锁，上redis分布式锁setnx
- setnx只加了锁，没有释放锁，出异常的话可能无法释放锁，必须在代码层面finally释放锁
- 宕机了，部署了微服务代码层面没有走到finally块，无法保证解锁，这个key无法删除，因此需要给key一个过期时间
- 为redis分布式锁key增加过期时间，setnx+过期时间（写在一起）
- 为了防止锁过期了，删掉了别人的锁，因此自己的锁要自己删，lua原子删除或者用redis的乐观锁事务
- 完美解决：Redisson