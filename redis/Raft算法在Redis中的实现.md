# Raft算法在Redis中的实现

## Sentinel

Sentinel是Redis的高可用解决方案：由一个或多个Sentinel实例组成的Sentinel系统可以监视任意多个主服务器，以及这些主服务器的从服务器，并在被监视的主服务器进入下线状态时，自动将下线主服务器属下的某个从服务器升级为新的主服务器，然后由新的主服务器代替已下线的主服务器继续处理命令请求。

Sentinel系统选举领头Sentinel的方法是对Raft算法的领头选举方法的实现。

`vim sentinel.conf`

- 其中mymaster为监控对象起的服务器名称(就是监控服务器的别名，这里是6379的别名)
- 1 为至少有多少个哨兵同意迁移的数量(就是主机挂掉了，有1个哨兵监控到，就可以切换主机)

```bash
sentinel monitor mymaster 192.168.200.132 6379 1 #1是客观下线的标准
sentinel down-after-milliseconds master 30000   #默认30s,判断主服务器主观下线
```

### 检测主观下线

默认情况下，Sentinel会每秒一次向与它创建了命令连接的实例（主服务器、从服务器、其他Sentinel）发送PING命令，通过实例返回的PING命令回复判断实例是否在线。

Sentinel配置文件中的`down-after-milliseconds`选项指定了Sentinel判断实例进入主观下线所需的时间长度：如果一个实例在`down-after-milliseconds`毫秒内，连续向Sentinel返回无效回复，那么Sentinel会修改这个实例所对应的实例结构，在结构的flags属性中打开`SRI_S_DOWN`标识，以此来表示这个实例已经进入主观下线状态。

#### 多个Sentinel设置的主观下线时长可能不同

down-after-milliseconds选项另一个需要注意的地方是，对于监视同一个主服务器的多个Sentinel来说，这些Sentinel所设置的down-after-milliseconds选项的值也可能不同，因此，当一个Sentinel将主服务器判断为主观下线时，其他Sentinel可能仍然会认为主服务器处于在线状态。举个例子，如果Sentinel1载入了以下配置：

```bash
sentinel monitor mymaster 192.168.200.132 6379 2 
sentinel down-after-milliseconds master 50000   
```

而Sentinel2则载入了以下配置：

```bash
sentinel monitor mymaster 192.168.200.132 6379 2 
sentinel down-after-milliseconds master 10000
```

那么当master的断线时长超过10000毫秒之后，Sentinel2会将master判断为主观下线，而Sentinel1却认为master仍然在线。只有当master的断线时长超过50000毫秒之后，Sentinel1和Sentinel2才会都认为master进入了主观下线状态。

### 

### 检查客观下线

当Sentinel将一个主服务器判断为主观下线之后，为了确认这个主服务器是否真的下线了，**它会向同样监视这一主服务器的其他Sentinel进行询问，看它们是否也认为主服务器已经进入了下线状态**（可以是主观下线或者客观下线）。当Sentinel从其他Sentinel那里接收到足够数量的已下线判断之后，Sentinel就会将从服务器判定为客观下线，并对主服务器执行故障转移操作。

![image-20220807010155931](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220807010155931.png)

举个例子，如果被Sentinel判断为主观下线的主服务器的IP为127.0.0.1，端口号为6379，并且Sentinel当前的配置纪元为0，那么Sentinel将向其他Sentinel发送以下命令：

```bash
SENTINEL is-master-down-by-addr 127.0.0.1 6379 0 *
```

回复命令如下：

![image-20220807010408947](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220807010408947.png)

当认为主服务器已经进入下线状态的Sentinel的数量，超过Sentinel配置中设置的quorum参数的值，那么该Sentinel就会认为主服务器已经进入客观下线状态。比如说，如果Sentinel在启动时载入了以下配置：

```bash
sentinel monitor mymaster 192.168.200.132 6379 2 
```

那么包括当前Sentinel在内，只要总共有两个Sentinel认为主服务器已经进入下线状态，那么当前Sentinel就将主服务器判断为客观下线

#### 不同Sentinel判断客观下线的条件可能不同

对于监视同一个主服务器的多个Sentinel来说，它们将主服务器标判断为客观下线的条件可能也不同：当一个Sentinel将主服务器判断为客观下线时，其他Sentinel可能并不是那么认为的。比如说，对于监视同一个主服务器的五个Sentinel来说，如果Sentinel1在启动时载入了以下配置：

```bash
sentinel monitor mymaster 192.168.200.132 6379 2 
```

那么当五个Sentinel中有两个Sentinel认为主服务器已经下线时，Sentinel1就会将主服务器标判断为客观下线。

而对于载入了以下配置的Sentinel2来说：

```bash
sentinel monitor mymaster 192.168.200.132 6379 5 
```

仅有两个Sentinel认为主服务器已下线，并不会令Sentinel2将主服务器判断为客观下线。



### 选举领头Sentinel

当一个主服务器被判断为客观下线时，监视这个下线主服务器的各个Sentinel会进行协商，选举出一个领头Sentinel，并由领头Sentinel对下线主服务器执行故障转移操作。

- 所有在线的Sentinel都有被选为领头Sentinel的资格，换句话说，监视同一个主服务器的多个在线Sentinel中的任意一个都有可能成为领头Sentinel
- 每次进行领头Sentinel选举之后，不论选举是否成功，所有Sentinel的配置纪元（configuration epoch）的值都会自增一次。配置纪元实际上就是一个计数器，并没有什么特别的。
- 在一个配置纪元里面，所有Sentinel都有一次将某个Sentinel设置为局部领头Sentinel的机会，并且局部领头一旦设置，在这个配置纪元里面就不能再更改。
- 每个发现**主服务器进入客观下线**的Sentinel都会要求其他Sentinel将自己设置为局部领头Sentinel。
- 当一个Sentinel（源Sentinel）向另一个Sentinel（目标Sentinel）发送SENTINEL is-master-down-by-addr命令，并且命令中的runid参数不是*符号而是源Sentinel的运行ID时，这表示源Sentinel要求目标Sentinel将前者设置为后者的局部领头Sentinel。
- Sentinel设置局部领头Sentinel的**规则是先到先得**：最先向目标Sentinel发送设置要求的源Sentinel将成为目标Sentinel的局部领头Sentinel，而之后接收到的所有设置要求都会被目标Sentinel拒绝。
- 目标Sentinel在接收到SENTINEL is-master-down-by-addr命令之后，将向源Sentinel返回一条命令回复，回复中的leader_runid参数和leader_epoch参数分别记录了目标Sentinel的局部领头Sentinel的运行ID和配置纪元。
- 源Sentinel在接收到目标Sentinel返回的命令回复之后，会**检查回复中leader_epoch参数的值和自己的配置纪元是否相同**，如果相同的话，那么源Sentinel继续取出回复中的leader_runid参数，如果**leader_runid参数的值和源Sentinel的运行ID一致**，那么表示目标Sentinel将源Sentinel设置成了局部领头Sentinel。
- 如果有**某个Sentinel被半数以上的Sentinel设置成了局部领头Sentinel**，那么这个Sentinel成为领头Sentinel。举个例子，在一个由10个Sentinel组成的Sentinel系统里面，只要有大于等于10/2+1=6个Sentinel将某个Sentinel设置为局部领头Sentinel，那么被设置的那个Sentinel就会成为领头Sentinel。
- 因为领头Sentinel的产生需要半数以上Sentinel的支持，并且每个Sentinel在每个配置纪元里面只能设置一次局部领头Sentinel，所以在一个配置纪元里面，只会出现一个领头Sentinel。
- 如果在给定时限内，没有一个Sentinel被选举为领头Sentinel，那么各个Sentinel将在一段时间之后再次进行选举，直到选出领头Sentinel为止。



## 集群

Redis集群是Redis提供的分布式数据库方案，集群通过分片（sharding）来进行数据共享，并提供复制和故障转移功能。

Redis集群中的节点分为主节点（master）和从节点（slave），其中主节点用于处理槽，而从节点则用于复制某个主节点，并在被复制的主节点下线时，代替下线主节点继续处理命令请求。

### 故障检测

集群中的每个节点都会定期地向集群中的其他节点发送PING消息，以此来检测对方是否在线，如果接收PING消息的节点没有在规定的时间内，向发送PING消息的节点返回PONG消息，那么发送PING消息的节点就会将接收PING消息的节点标记为**疑似下线（probable fail，PFAIL）。**

![image-20220807131235567](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220807131235567.png)

当一个主节点A通过消息得知主节点B认为主节点C进入了疑似下线状态时，主节点A会在自己的clusterState.nodes字典中找到主节点C所对应的clusterNode结构，并将主节点B的下线报告（failure report）添加到clusterNode结构的fail_reports链表里面：

```c++
struct clusterNode{
    //一个链表，记录了所有其他节点对该节点的下线报告
    list *fail_reports;
}
```

每个下线报告如下：

```c++
struct clusterNodeFailReport{
    struct clusterNode *node;  //报告目标节点已经下线的节点
    mstime_t time;  //最后一次从node节点收到下线报告的时间
}
```

举个例子，如果主节点7001在收到主节点7002、主节点7003发送的消息后得知，主节点7002和主节点7003都认为主节点7000进入了疑似下线状态，那么主节点7001将为主节点7000创建图17-38所示的下线报告。

![image-20220807131839735](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220807131839735.png)

如果在一个集群里面，**半数以上负责处理槽的主节点都将某个主节点x报告为疑似下线**，那么这个主节点x将被标记为已下线（FAIL），将主节点x标记为已下线的节点会向集群广播一条关于主节点x的FAIL消息，所有收到这条FAIL消息的节点都会立即将主节点x标记为已下线。

举个例子，对于图17-38所示的下线报告来说，主节点7002和主节点7003都认为主节点7000进入了下线状态，并且主节点7001也认为主节点7000进入了疑似下线状态（代表主节点7000的结构打开了REDIS_NODE_PFAIL标识），综合起来，在集群四个负责处理槽的主节点里面，有三个都将主节点7000标记为下线，数量已经超过了半数，**所以主节点7001会将主节点7000标记为已下线，并向集群广播一条关于主节点7000的FAIL消息**

### 故障转移

当一个从节点发现自己正在复制的主节点进入了已下线状态时【收到了FAIL消息】，**从节点将开始对下线主节点进行故障转移**，以下是故障转移的执行步骤：

1）复制下线主节点的所有从节点里面，会有一个从节点被选中。

2）被选中的从节点会执行SLAVEOF no one命令，成为新的主节点。

3）新的主节点会撤销所有对已下线主节点的槽指派，并将这些槽全部指派给自己。

4）新的主节点向集群广播一条PONG消息，这条PONG消息可以让集群中的其他节点立即知道这个节点已经由从节点变成了主节点，并且这个主节点已经接管了原本由已下线节点负责处理的槽。

5）新的主节点开始接收和自己负责处理的槽有关的命令请求，故障转移完成。

### 选举新的主节点

以下是集群选举新的主节点的方法：

1）集群的配置纪元是一个自增计数器，它的初始值为0。

2）当集群里的某个节点开始一次故障转移操作时，集群配置纪元的值会被增一。

3）对于每个配置纪元，集群里每个负责处理槽的主节点都有一次投票的机会，而**第一个向主节点要求投票的从节点将获得主节点的投票**。

4）当从节点发现自己正在复制的主节点进入已下线状态时，从节点会向集群广播一条CLUSTERMSG_TYPE_FAILOVER_AUTH_REQUEST消息，要求所有收到这条消息、并且具有投票权的主节点向这个从节点投票。

5）如果一个主节点具有投票权（它正在负责处理槽），并且这个主节点尚未投票给其他从节点，那么主节点将向要求投票的从节点返回一条CLUSTERMSG_TYPE_FAILOVER_AUTH_ACK消息，表示这个主节点支持从节点成为新的主节点。

6）每个参与选举的从节点都会接收CLUSTERMSG_TYPE_FAILOVER_AUTH_ACK消息，并根据自己收到了多少条这种消息来统计自己获得了多少主节点的支持。

7）如果集群里有N个具有投票权的主节点，那么当一个从节点收集到大于等于N/2+1张支持票时，这个从节点就会当选为新的主节点

8）因为在每一个配置纪元里面，每个具有投票权的主节点只能投一次票，所以如果有N个主节点进行投票，那么具有大于等于N/2+1张支持票的从节点只会有一个，这确保了新的主节点只会有一个。

9）如果在一个配置纪元里面没有从节点能收集到足够多的支持票，那么集群进入一个新的配置纪元，并再次进行选举，直到选出新的主节点为止。