# Redis主从复制

## 搭建一主二从

**一个虚拟机配置的话，需要设置好端口号，以及防止RDB和AOF文件的重名**

1、创建/myredis文件夹

2、cp /etc/redis.conf /myredis

3、配置一主两从，创建三个配置文件

- redis6379.conf
- redis6380.conf
- redis6381.conf
- `include /myredis/redis.conf` ：引入公共部分，为了方便，**关闭AOF或者更改aof文件的名字**

```bash
include /myredis/redis.conf
pidfile /var/run/redis_6379.pid
port 6379
dbfilename dump6379.rdb

include /myredis/redis.conf
pidfile /var/run/redis_6380.pid
port 6380
dbfilename dump6380.rdb

include /myredis/redis.conf
pidfile /var/run/redis_6381.pid
port 6381
dbfilename dump6381.rdb
```

4、启动三个服务

![image-20220221192900857](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220221192900857.png)

5、查看各个服务的情况`info replication`

```bash
[root@VarerLeet2 bin]# redis-cli -p 6379
127.0.0.1:6379> info replication
# Replication
role:master
connected_slaves:0
master_failover_state:no-failover
master_replid:33946f60d7c5a33d254af9c78b9af928f59856c4
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:0
second_repl_offset:-1
repl_backlog_active:0
repl_backlog_size:1048576
repl_backlog_first_byte_offset:0
repl_backlog_histlen:0
```

6、配从库，不配主库

`slaveof ip port`

```bash
127.0.0.1:6380> slaveof 192.168.200.132 6379
OK
127.0.0.1:6381> slaveof 192.168.200.132 6379
OK
```

**从机的情况**

```bash
127.0.0.1:6380> info replication
# Replication
role:slave
master_host:192.168.200.132
master_port:6379
master_link_status:up
master_last_io_seconds_ago:9
master_sync_in_progress:0
slave_repl_offset:42
slave_priority:100
slave_read_only:1
connected_slaves:0
master_failover_state:no-failover
master_replid:c652c6e5e637e9f36eed0ca0510aa3ef2b8821ff
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:42
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:42
```

**主机的情况**

```bash
127.0.0.1:6379> info replication
# Replication
role:master
connected_slaves:2
slave0:ip=192.168.200.132,port=6380,state=online,offset=84,lag=0
slave1:ip=192.168.200.132,port=6381,state=online,offset=84,lag=1
master_failover_state:no-failover
master_replid:c652c6e5e637e9f36eed0ca0510aa3ef2b8821ff
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:84
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:84
```

7、主机可读可写，从机写报错

```bash
127.0.0.1:6381> set ke v2
(error) READONLY You can't write against a read only replica.
```

8、主机挂掉，重启之后从机还在

```bash
[root@VarerLeet2 bin]# ps -ef | grep redis
root      17000      1  0 19:28 ?        00:00:01 redis-server *:6379
root      17006      1  0 19:28 ?        00:00:00 redis-server *:6380
root      17012      1  0 19:28 ?        00:00:01 redis-server *:6381
root      17066  13715  0 19:31 pts/1    00:00:00 redis-cli -p 6380
root      17129  17073  0 19:32 pts/2    00:00:00 redis-cli -p 6381
root      17189  12839  0 19:39 pts/0    00:00:00 grep --color=auto redis
[root@VarerLeet2 bin]# kill 17000
[root@VarerLeet2 bin]# redis-server /myredis/redis6379.conf 
[root@VarerLeet2 bin]# redis-cli -p 6379
127.0.0.1:6379> info replication
# Replication
role:master
connected_slaves:2
slave0:ip=192.168.200.132,port=6380,state=online,offset=14,lag=0
slave1:ip=192.168.200.132,port=6381,state=online,offset=14,lag=0
master_failover_state:no-failover
master_replid:5f638bc470f5c23ec2c5b2a1306699368c293346
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:14
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:14
```

9、从机挂掉，需要重新设置主机的信息，若从机挂掉的时候，主机添加了几个kv，重新连接到主机后，这几个kv也会自动得到



## 主从复制原理

### 命令传播

- 每当主服务器执行客户端发送的写命令，主从数据库就会产生不一致的状态，主服务器会对从服务器执行命令传播，主服务器会给从服务器也发送相同的写命令，使数据库回到一致状态。

### 复制

**PSNYC命令具有完整重同步和部分重同步两种模式**

- 完整重同步处理初次复制情况

  - 主服务器执行BGSAVE命令来生成RDB文件，发送给从服务器，然后再发送缓冲区保存的所有写命令（生成RDB文件时新进来的命令也会写在缓冲区里），完成同步
- 部分重同步用于处理断线后的情况

  - 当从服务器断线重连主服务器时，条件允许的话（见下面），主服务器可以将主从服务器连接断开期间执行的写命令发送给从服务器，从服务器只需要接受并执行这些写命令即可完成同步，而不需要载入全量的RDB文件，提升了性能（旧版复制没有部分重同步，十分低效）
- 从服务器第一次执行复制 ？ (PSYNC ? -1) : (PSYNC <runid> <offset>)

  - 主服务器返回+FULLRESYNC <runid> <offset> 就是完全重同步
- 主服务器返回+CONTINUE就是部分重同步

#### 复制延迟问题

由于所有的写操作都是先在Master上操作，然后同步更新到Slave上，所以从Master同步到Slave机器有一定的延迟，当系统很繁忙的时候，延迟问题会更加严重，Slave机器数量的增加也会使这个问题更加严重。



### 部分重同步

由三部分构成：复制偏移量，主服务器的复制积压缓冲区和服务器的运行ID



#### 复制偏移量

执行复制的双方，主从服务器会分别维护一个复制偏移量。

- 主服务器每次向从服务器传播N个字节的数据时候，就将自己的复制偏移量+N
- 从服务器每次收到N个字节的数据，就将自己的复制偏移量+N
- **对比复制偏移量就可以知道主从服务器是否处于一致的状态**
- `info replication`可以看到复制偏移量



#### 主服务器的复制积压缓冲区

主服务器应该对从服务器执行完整重同步还是部分重同步？若执行部分重同步的话，如何补偿从服务器A在断线期间丢失的那部分数据呢？这和复制积压缓冲区有关。

> 复制积压缓冲区是由主服务器维护的一个固定长度的先进先出队列，默认大小为1MB

当主服务器进行命令传播的时候，不仅将写命令发送给所有从服务器，还将写命令入队到复制积压缓冲区里面。因此，**主服务器的复制积压缓冲区保存着一部分最近传播的写命令，并且复制积压缓冲区会为队列中的每个字节记录相应的复制偏移量**



因此，从服务器重新连上主机后，通过PSYNC命令将自己的复制偏移量offset发给主服务器

- 若offset偏移量之后的数据还在复制积压缓冲区，那么部分重同步，否则完整重同步
- 因为是FIFO队列，偏移量小的还在复制积压缓冲区，那么所需要的命令肯定都还在。



#### 服务器运行ID

运行ID在服务器运行时自动生成，40个随机的十六进制字符。

- 当从服务器对主服务器进行初次复制的时候，主服务器会将自己的运行ID传给从服务器，而从服务器会将这个ID存起来
  - `info replication就可以查看到`
  - 若ID不同，那么执行完全重同步，ID相同的话，再看看复制积压缓冲区能否允许部分重同步。

![image-20220221203346736](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220221203346736.png)



## 心跳检测

在命令传播阶段，从服务器默认每秒一次的频率，向主服务器发送命令

`REPLCONF ACK <replication_offset>`

- replication_offset 就是从服务器当前的复制偏移量

发送REPLCONF ACK 对主从服务器有三个作用：

- 检测主从服务器的网络连接状态
- 辅助实现min-slaves选项
- **检测命令丢失**



### 检测主从服务器的网络连接状态

lag一栏中，可以看到相应从服务器最后一次向主服务器发送REPLCONF ACK命令距离现在过了几秒

![image-20220221204315231](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220221204315231.png)



### 辅助实现min-slaves选项

可以防止主从服务器在不安全的情况下执行写命令

如果给主服务器配置如下：

```bash
min-slaves-to-write 3
min-slaves-max-lag 10
```

那么在从服务器的数量少于3，或者三个从服务器的延迟(lag)值都大于等于10，主服务器会拒绝执行写命令



### 检测命令丢失

`REPLCONF ACK <replication_offset>`，由于发送复制偏移量，一旦发现主从的复制偏移量不一致的话，主服务器就会根据这个偏移量找到复制积压缓冲区里面丢失的那些命令，发给从服务器，因此偶然的一次命令传播丢失了也没关系，可以恢复

- REPLCONF ACK和复制积压缓冲区都是Redis2.8版本增加的，因此选用主从架构的话，版本至少要在2.8以上



## 薪火相传

上一个Slave可以是下一个slave的Master，Slave同样可以接收其他 slaves的连接和同步请求，那么该slave作为了链条中下一个的master, 可以有效减轻master的写压力,去中心化降低风险。

- 风险是一旦某个slave宕机，后面的slave都没法备份
- 主机挂了，从机还是从机，无法写数据了

![image-20220222144341800](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220222144341800.png)

1、首先，6379是主机，6380和6381是6379的从机，然后让6381成为6380的从机

```bash
127.0.0.1:6381> slaveof 192.168.200.132 6379
OK
127.0.0.1:6381> slaveof 192.168.200.132 6380
OK
```

2、主机写命令，所有从机都能收到，包括6381

```bash
127.0.0.1:6379> set ccc ddd
OK
127.0.0.1:6381> get ccc
"ddd"
```

3、如果这时候6380挂掉了，会怎样？

```bash
127.0.0.1:6380> shutdown
not connected>
127.0.0.1:6379> set abcd abcd
OK
127.0.0.1:6381> get abcd  #这时候6381就是一个孤立节点了,无法接受到写命令
(nil)

```



## 反客为主

`slaveof no one`：当一个master宕机后，后面的slave可以立刻升为master，其后面的slave不用做任何修改。

还是上面那个例子，这时候6381由于是从机，无法写，可以让它变成主机，6381若原来有从服务器的话，那么就会变成主从关系，6381的写命令可以传播给它的从服务器

```bash
127.0.0.1:6381> set aa bb
(error) READONLY You can't write against a read only replica.
127.0.0.1:6381> slaveof no one
OK
127.0.0.1:6381> set aa bb
OK
```



## 哨兵模式（sentinel）

### demo

**反客为主的自动版**，能够后台监控主机是否故障，如果故障了根据投票数自动将从库转换为主库

1、调整为一主二从的模式，6379是主机

2、自定义的/myredis目录下新建sentinel.conf文件

3、vim sentinel.conf

- 其中mymaster为监控对象起的服务器名称(就是监控服务器的别名，这里是6379的别名)
- 1 为至少有多少个哨兵同意迁移的数量(就是主机挂掉了，有1个哨兵监控到，就可以切换主机)

```bash
sentinel monitor mymaster 192.168.200.132 6379 1
sentinel down-after-milliseconds master 30000   #默认30s,判断主服务器主观下线
```

4、启动,有一个命令就是redis-sentinel

![image-20220222000841414](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220222000841414.png)

5、看一下主机挂掉之后，哨兵这边会怎样

```bash
127.0.0.1:6379> shutdown
not connected> 

```

![image-20220222000745291](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220222000745291.png)

6、原主机重启后会变为从机。

![image-20220222000950359](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220222000950359.png)



### 选举领头羊

`sentinel monitor master 127.0.0.1 6379 2`

`sentinel down-after-milliseconds master 10000`

对于上述sentinel的配置，当master断线超过10s，**sentinel会将master判断为主观下线**，为了确认这个master是否真的下线了，它会向同样监视这个master的其他sentinel询问。

**若有2个或者2个以上(包括自己)判定master下线,那个我这个sentinel会认为master是客观下线**

一旦判定为客观下线，那么就开始进行故障转移。

**当一个主服务器被判断为客观下线，监视这个下线主服务器的各个Sentinel会进行协商，选举出一个领头Sentinel，并由这个领头Sentinel对下线主服务器执行故障转移操作。**

选举领头的过程是和Raft算法相似的。

- 若一个sentinel发现了客观下线的主机，那么它就开始成为candidate，发起投票这样子。



### 故障转移规则

![image-20220222001530739](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220222001530739.png)

优先级：有一个参数是replica-priority，默认值是100，**看注释**

![image-20220222001339985](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220222001339985.png)

复制偏移量选择最大的，因为它获得原主机数据更多，损失的data就越少。

每个redis实例启动后都会随机生成一个40位的runid



**当新的主服务器出现后，领头Sentinel会让已下线主服务器的从服务器都去复制这个新的主服务器，通过salveof命令实现。**

最后，会让已下线的主服务器成为新主服务器的从服务器。



## 搭建Redis集群

**Redis 集群实现了对Redis的水平扩容，即启动N个redis节点，将整个数据库分布存储在这N个节点中，每个节点存储总数据的1/N。**

Redis 集群通过分区（partition）来提供一定程度的可用性（availability）： 即使集群中有一部分节点失效或者无法进行通讯， 集群也可以继续处理命令请求。

> 连接各个节点的工作可以使用cluster meet <ip> <port> 来完成
>
> 16384个槽分配完毕，集群进入上线状态。
>
> unsigned char slots[16384 / 8];
>
> int numslots;
>
> **一个节点除了会将自己负责处理的槽记录在clusterNode结构的slots属性和numslots属性之外，还会将自己的slots数组通过消息发送给集群中的其他节点，以此来告知其他节点自己目前负责处理哪些槽，因此集群中每个节点都会知道16384个槽分别指派给了集群中的哪些节点。**

### 1、删除持久化数据rdb,aof

### 2、制作**6379,6380,6381,6389,6390,6391**

cluster-enabled yes  打开集群模式

cluster-config-file nodes-6379.conf 设定节点配置文件名

cluster-node-timeout 15000  设定节点失联时间，超过该时间（毫秒），集群自动进行主从切换。

```bash
[root@VarerLeet2 myredis]# cat redis6379.conf
include /myredis/redis.conf
pidfile "/var/run/redis_6379.pid"
port 6379
dbfilename "dump6379.rdb"
cluster-enabled yes
cluster-config-file nodes-6379.conf
cluster-node-timeout 15000
其他五个文件,用%s/6379/63xx替换
:%s/6379/6380  
```

### 3、启动六个服务，确保nodes文件生成了

![image-20220222132616937](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220222132616937.png)

### 4、将六个节点合成一个集群,合成集群前，确保集群都是空的，即没有存任何数据，不然会报错

```bash
[root@VarerLeet2 src]# pwd   #来到redis的src文件
/opt/redis-6.2.1/src

此处不要用127.0.0.1， 用真实IP地址
--replicas 1 采用最简单的方式配置集群，一台主机，一台从机，正好三组。

redis-cli --cluster create --cluster-replicas 1 192.168.200.132:6379 192.168.200.132:6380 192.168.200.132:6381 192.168.200.132:6389 192.168.200.132:6390 192.168.200.132:6391
```

```bash
[root@VarerLeet2 src]# redis-cli --cluster create --cluster-replicas 1 192.168.200.132:6379 192.168.200.132:6380 192.168.200.132:6381 192.168.200.132:6389 192.168.200.132:6390 192.168.200.132:6391
>>> Performing hash slots allocation on 6 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
Adding replica 192.168.200.132:6390 to 192.168.200.132:6379
Adding replica 192.168.200.132:6391 to 192.168.200.132:6380
Adding replica 192.168.200.132:6389 to 192.168.200.132:6381
>>> Trying to optimize slaves allocation for anti-affinity
[WARNING] Some slaves are in the same host as their master
M: 91d2c0abf142c0e8a2c53d7d652e07a8188728e1 192.168.200.132:6379
   slots:[0-5460] (5461 slots) master
M: b8102ebd9198731bb179f44d17b074e43b4f865f 192.168.200.132:6380
   slots:[5461-10922] (5462 slots) master
M: 1f3f5df05dd4c115910197724f1307e3f8ceb04b 192.168.200.132:6381
   slots:[10923-16383] (5461 slots) master
S: ce1bfb0d80815de0727eba74075e051656e93bcb 192.168.200.132:6389
   replicates 91d2c0abf142c0e8a2c53d7d652e07a8188728e1
S: 75081c72ac296e16adf50af05e9d227023c25809 192.168.200.132:6390
   replicates b8102ebd9198731bb179f44d17b074e43b4f865f
S: 8e479c28fa22b3c1273f193230e4229f45c25595 192.168.200.132:6391
   replicates 1f3f5df05dd4c115910197724f1307e3f8ceb04b
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join
.
>>> Performing Cluster Check (using node 192.168.200.132:6379)
M: 91d2c0abf142c0e8a2c53d7d652e07a8188728e1 192.168.200.132:6379
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
S: 8e479c28fa22b3c1273f193230e4229f45c25595 192.168.200.132:6391
   slots: (0 slots) slave
   replicates 1f3f5df05dd4c115910197724f1307e3f8ceb04b
S: 75081c72ac296e16adf50af05e9d227023c25809 192.168.200.132:6390
   slots: (0 slots) slave
   replicates b8102ebd9198731bb179f44d17b074e43b4f865f
M: b8102ebd9198731bb179f44d17b074e43b4f865f 192.168.200.132:6380
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
M: 1f3f5df05dd4c115910197724f1307e3f8ceb04b 192.168.200.132:6381
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
S: ce1bfb0d80815de0727eba74075e051656e93bcb 192.168.200.132:6389
   slots: (0 slots) slave
   replicates 91d2c0abf142c0e8a2c53d7d652e07a8188728e1
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

### 5、连接

普通连接方式，可能直接进入读主机，存储数据时，会出现MOVED重定向操作。所以，应该以集群方式登录。

```bash
[root@VarerLeet2 src]# redis-cli -p 6379
127.0.0.1:6379> set k1 v1
(error) MOVED 12706 192.168.200.132:6381
127.0.0.1:6379> keys *
(empty array)
127.0.0.1:6379> set k2 v2
OK
127.0.0.1:6379> keys *
1) "k2"

```

-c 采用集群策略连接，设置数据会自动切换到相应的写主机

```bash
[root@VarerLeet2 src]# redis-cli -c -p 6379
127.0.0.1:6379> set k2 v2
OK
127.0.0.1:6379> set k1 v1
-> Redirected to slot [12706] located at 192.168.200.132:6381
OK
192.168.200.132:6381> 

```

### 6、通过cluster nodes命令查看集群信息

**看到myself就可以知道自己的节点信息**

```bash
192.168.200.132:6381> cluster nodes
91d2c0abf142c0e8a2c53d7d652e07a8188728e1 192.168.200.132:6379@16379 master - 0 1645508410058 1 connected 0-5460
b8102ebd9198731bb179f44d17b074e43b4f865f 192.168.200.132:6380@16380 master - 0 1645508409000 2 connected 5461-10922
ce1bfb0d80815de0727eba74075e051656e93bcb 192.168.200.132:6389@16389 slave 91d2c0abf142c0e8a2c53d7d652e07a8188728e1 0 1645508407000 1 connected
8e479c28fa22b3c1273f193230e4229f45c25595 192.168.200.132:6391@16391 slave 1f3f5df05dd4c115910197724f1307e3f8ceb04b 0 1645508411065 3 connected
75081c72ac296e16adf50af05e9d227023c25809 192.168.200.132:6390@16390 slave b8102ebd9198731bb179f44d17b074e43b4f865f 0 1645508409052 2 connected
1f3f5df05dd4c115910197724f1307e3f8ceb04b 192.168.200.132:6381@16381 myself,master - 0 1645508410000 3 connected 10923-16383

```

### 7、一个集群至少要有三个主节点。

选项 --cluster-replicas 1 表示我们希望为集群中的每个主节点创建一个从节点。

![image-20220222134414388](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220222134414388.png)

### 8、slots

**[OK] All nodes agree about slots configuration.**

**>>> Check for open slots...**

**>>> Check slots coverage...**

**[OK] All 16384 slots covered.**

一个 Redis 集群包含 16384 个插槽（hash slot）， 数据库中的每个键都属于这 16384 个插槽的其中一个， 

集群使用公式 CRC16(key) % 16384 来计算键 key 属于哪个槽， 其中 CRC16(key) 语句用于计算键 key 的 CRC16 校验和 。

集群中的每个节点负责处理一部分插槽。 举个例子， 如果一个集群可以有主节点， 其中：

节点 A 负责处理 0 号至 5460 号插槽。

节点 B 负责处理 5461 号至 10922 号插槽。

节点 C 负责处理 10923 号至 16383 号插槽。

**cluster nodes命令可以查看插槽的范围**

### 9、集群中录入值

在redis-cli每次录入、查询键值，redis都会计算出该key应该送往的插槽，如果不是该客户端对应服务器的插槽，redis会报错，并告知应前往的redis实例地址和端口。

**redis-cli客户端提供了 –c 参数实现自动重定向。**

如 redis-cli -c –p 6379 登入后，再录入、查询键值对可以自动重定向。

不在一个slot下的键值，是不能使用mget,mset等多键操作。

```bash
[root@VarerLeet2 src]# redis-cli -c -p 6379
127.0.0.1:6379> mset k1 v1 k2 v2 k3 v3
(error) CROSSSLOT Keys in request don't hash to the same slot
```

**可以通过{}来定义组的概念，从而使key中{}内相同内容的键值对放到一个slot中去。**

```bash
127.0.0.1:6379> mset k1{g} v1 k2{g} v2
-> Redirected to slot [7233] located at 192.168.200.132:6380
OK
192.168.200.132:6380> keys *
1) "k1{g}"
2) "k2{g}"
```

### 10、查询集群中的值

CLUSTER GETKEYSINSLOT <slot><count> 返回 count 个 slot 槽中的键。

```bash
192.168.200.132:6380> get k1
-> Redirected to slot [12706] located at 192.168.200.132:6381
"v1"
192.168.200.132:6381> mget k1{g} k2{g}
-> Redirected to slot [7233] located at 192.168.200.132:6380
1) "v1"
2) "v2"
192.168.200.132:6380> cluster countkeysinslot 7233
(integer) 2
192.168.200.132:6380> cluster getkeysinslot 7233 2
1) "k1{g}"
2) "k2{g}"
```

### 11、故障恢复

如果主节点下线？从节点能否自动升为主节点？15秒后，从机变主机，主机重启后变成从机

如果所有某一段插槽的主从节点都宕掉，redis服务是否还能继续?

如果某一段插槽的主从都挂掉，而cluster-require-full-coverage 为yes ，那么 ，整个集群都挂掉

如果某一段插槽的主从都挂掉，而cluster-require-full-coverage 为no ，那么，该插槽数据全都不能使用，也无法存储。

redis.conf中的参数 cluster-require-full-coverage，默认是yes，整个集群挂掉`(error) CLUSTERDOWN The cluster is down`

![image-20220222140824208](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220222140824208.png)



### 12、为什么是16384？

Redis 集群有16384个哈希槽，每个key通过CRC16校验后对16384取模来决定放置哪个槽，集群的每个节点负责一部分hash槽。

在redis节点发送心跳包时需要把所有的槽放到这个心跳包里，以便让节点知道当前集群信息，16384=16k，在发送心跳包时使用`char`进行bitmap压缩后是2k（`2 * 8 (8 bit) * 1024(1k) = 16K`），也就是说使用2k的空间创建了16k的槽数。

**简而言之，发送心跳包要传输数据，网络传输是需要代价的。**

虽然使用CRC16算法最多可以分配65535（2^16-1）个槽位，65535=65k，压缩后就是8k（`8 * 8 (8 bit) * 1024(1k) =65K`），也就是说需要需要8k的心跳包，作者认为这样做不太值得；并且一般情况下一个redis集群不会有超过**1000个master**节点，所以16k的槽位是个比较合适的选择。

> unsigned char slots[16384 / 8];
>
> C语言,每一个char占用1个字节，那么一个char有8位，这8位通过位运算，也可以很快速的存储0或1。
