# Raft

Raft 是一种分布式一致性协议，主要是用来竞选主节点。它依靠 **状态机** 和 **主从同步** 的方式，在各个节点之间实现数据的一致性。

Raft 算法是一种分布式共识算法。

为了优雅地应对任意网络分区和服务器故障问题，Raft要求集群中的**半数以上服务器**是正常启动的，而且在任意指定时刻都可以为领导者所用。如果有3台服务器，Raft可以允许1台机器故障，对于5台服务器的集群，可以允许2台机器故障； 对于`2N+1`台服务器，可以允许`N`台服务器出现故障。

## 节点类型

一个 Raft 集群包括若干服务器，在任意的时间，每个服务器一定会处于以下三个状态中的一个：

- `Leader`：负责发起心跳，响应客户端，创建日志，同步日志。
- `Candidate`：Leader 选举过程中的临时角色，由 Follower 转化而来，发起投票参与竞选。
- `Follower`：接受 Leader 的心跳和日志同步数据，投票给 Candidate。

Leader 会周期性的发送心跳包给 Follower。每个 Follower 都设置了一个随机的竞选超时时间，一般为 150ms~300ms，如果在这个时间内没有收到 Leader 的心跳包，就会变成 Candidate，进入竞选阶段。

![image-20220326153106210](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220326153106210.png)



## 任期

![image-20220326153244742](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220326153244742.png)

raft 算法将时间划分为任意长度的任期（term），任期用连续的数字表示，看作当前 term 号。**每一个任期的开始都是一次选举**，在选举开始时，一个或多个 Candidate 会尝试成为 Leader。如果一个 Candidate 赢得了选举，它就会在该任期内担任 Leader。如果没有选出 Leader，将会开启另一个任期，并立刻开始下一次选举。raft 算法保证在给定的一个任期最少要有一个 Leader。

每个节点都会存储当前的 term 号，当服务器之间进行通信时会交换当前的 term 号；如果有服务器发现自己的 term 号比其他人小，那么他会更新到较大的 term 值。如果一个 Candidate 或者 Leader 发现自己的 term 过期了，他会立即退回成 Follower。如果一台服务器收到的请求的 term 号是过期的，那么它会拒绝此次请求。



## 日志

- `entry`：每一个事件成为 entry，只有 Leader 可以创建 entry。entry 的内容为`<term,index,cmd>`其中 cmd 是可以应用到状态机的操作。
- `log`：由 entry 构成的数组，每一个 entry 都有一个表明自己在 log 中的 index。只有 Leader 才可以改变其他节点的 log。entry 总是先被 Leader 添加到自己的 log 数组中，然后再发起共识请求，获得同意后才会被 Leader 提交给状态机。Follower 只能从 Leader 获取新日志和当前的 commitIndex，然后把对应的 entry 应用到自己的状态机中。



## 单个 Candidate 的竞选

- 下图是一个分布式系统的最初阶段，此时只有 Follower 没有 Leader。
- Node A 等待一个随机的竞选超时时间之后（150ms and 300ms），没收到 Leader 发来的心跳包，因此进入竞选阶段，成为candidate

![image-20220326165152444](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220326165152444.png)

- 此时 Node A 发送投票请求给其它所有节点。

![68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f3131313532313131383434353533382e676966](https://raw.githubusercontent.com/xuhaoyao/images/master/img/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f3131313532313131383434353533382e676966.gif)

- 其它节点会对请求进行回复，如果超过一半的节点回复了，那么该 Candidate 就会变成 Leader。

![68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f3131313532313131383438333033392e676966](https://raw.githubusercontent.com/xuhaoyao/images/master/img/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f3131313532313131383438333033392e676966.gif)

- 之后 Leader 会周期性地发送心跳包给 Follower，Follower 接收到心跳包，会重新开始计时。

![68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f3131313532313131383634303733382e676966](https://raw.githubusercontent.com/xuhaoyao/images/master/img/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f3131313532313131383634303733382e676966.gif)



## 多个 Candidate 竞选

- 如果有多个 Follower 成为 Candidate，并且所获得票数相同，那么就需要重新开始投票。例如下图中 Node B 和 Node D 都获得两票，需要重新开始投票。

![68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f3131313532313131393230333334372e676966](https://raw.githubusercontent.com/xuhaoyao/images/master/img/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f3131313532313131393230333334372e676966.gif)

- 由于**每个节点设置的随机竞选超时时间不同**，因此下一次再次出现多个 Candidate 并获得同样票数的概率很低。

![68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f3131313532313131393336383731342e676966](https://raw.githubusercontent.com/xuhaoyao/images/master/img/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f3131313532313131393336383731342e676966.gif)



## 数据同步

- 来自客户端的修改都会被传入 Leader。注意该修改还未被提交，只是写入日志中。

![68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f37313535303431343130373537362e676966](https://raw.githubusercontent.com/xuhaoyao/images/master/img/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f37313535303431343130373537362e676966.gif)

- Leader 会把修改复制到所有 Follower。

![68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f39313535303431343133313333312e676966](https://raw.githubusercontent.com/xuhaoyao/images/master/img/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f39313535303431343133313333312e676966.gif)

- Leader 会等待大多数的 Follower 也进行了修改，然后才将修改提交，同时回复给客户端。

![68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f3130313535303431343135313938332e676966](https://raw.githubusercontent.com/xuhaoyao/images/master/img/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f3130313535303431343135313938332e676966.gif)

- 此时 Leader 会通知的所有 Follower 让它们也提交修改，此时所有节点的值达成一致。

![68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f3131313535303431343138323633382e676966](https://raw.githubusercontent.com/xuhaoyao/images/master/img/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f3131313535303431343138323633382e676966.gif)



## 网络分区情况

1、首先，健康状况下，这时候leader是B

<img src="https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220326170339170.png" alt="image-20220326170339170" style="zoom:67%;" />

2、出现网络分区

<img src="https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220326170414435.png" alt="image-20220326170414435" style="zoom:67%;" />



3、网络分区使得出现了两个leader，注意这里，**C D E是新一轮选举，它们的term这时候比A B的大了**

<img src="https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220326170443694.png" alt="image-20220326170443694" style="zoom:67%;" />



4、因为下面的分区，不满足超过半数条件，因此客户端的命令无法成功执行

<img src="https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220326170635432.png" alt="image-20220326170635432" style="zoom:67%;" />

5、因为上面的分区，满足超过半数条件，客户端的命令可以执行

<img src="https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220326170719951.png" alt="image-20220326170719951" style="zoom:67%;" />



6、若这时候恢复网络分区，因为上面的term比较大，所以这时候C是leader，这时候集群又恢复了数据一致的情况

<img src="https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220326170907780.png" alt="image-20220326170907780" style="zoom:67%;" />

![image-20220326170924203](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220326170924203.png)