# 三大日志(binlog、redo log和undo log)

`MySQL` 日志 主要包括错误日志、查询日志、慢查询日志、事务日志、二进制日志几大类。其中，比较重要的还要属二进制日志 `binlog`（归档日志）和事务日志 `redo log`（重做日志）和 `undo log`（回滚日志）。

- binlog是Server层面的，任何存储引擎对于数据库的更改都会产生，主要用来进行POING-IN-TIME（PIT）的恢复和主从复制
  - PIT就是归档，“有了归档日志可以恢复MySQL半个月内任何一秒时刻的状态”。
- redo log是InnoDB存储引擎层产生的，redo log让InnoDB存储引擎有了崩溃恢复的能力`crash safe`

<img src="https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220204161008210.png" alt="image-20220204161008210" style="zoom:67%;" />



## redo log

> 每条 redo 记录由“表空间号+数据页号+偏移量+修改数据长度+具体修改的数据”组成
>
> ib_logfile_0，ib_logfile_1

`redo log`（重做日志）是`InnoDB`存储引擎独有的，它让`MySQL`拥有了崩溃恢复能力。

比如 `MySQL` 实例挂了或宕机了，重启时，`InnoDB`存储引擎会使用`redo log`恢复数据，保证数据的持久性与完整性。

- ![image-20220724101954541](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220724101954541.png)

`MySQL` 中数据是以页为单位，你查询一条记录，会从硬盘把一页的数据加载出来，加载出来的数据叫数据页，会放入到 `Buffer Pool` 中。

后续的查询都是先从 `Buffer Pool` 中找，没有命中再去硬盘加载，减少硬盘 `IO` 开销，提升性能。

更新表数据的时候，也是如此，发现 `Buffer Pool` 里存在要更新的数据，就直接在 `Buffer Pool` 里更新。

然后会把“在某个数据页上做了什么修改”记录到重做日志缓存（`redo log buffer`）里，接着刷盘到 `redo log` 文件里。

![image-20220724102120241](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220724102120241.png)

理想情况，事务一提交就会进行刷盘操作，但实际上，刷盘的时机是根据策略来进行的。



### 刷盘时机

`InnoDB` 存储引擎为 `redo log` 的刷盘策略提供了 `innodb_flush_log_at_trx_commit` 参数，它支持三种策略：

- **0** ：设置为 0 的时候，表示每次事务提交时不进行刷盘操作
- **1** ：设置为 1 的时候，表示每次事务提交时都将进行刷盘操作（默认值）
- **2** ：设置为 2 的时候，表示每次事务提交时都只把 redo log buffer 内容写入 page cache

`innodb_flush_log_at_trx_commit` 参数默认为 1 ，也就是说当事务提交时会调用 `fsync` 对 redo log 进行刷盘

另外，`InnoDB` 存储引擎有一个后台线程，每隔`1` 秒，就会把 `redo log buffer` 中的内容写到文件系统缓存（`page cache`），然后调用 `fsync` 刷盘。

![image-20220724102340055](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220724102340055.png)

**也就是说，一个没有提交事务的 `redo log` 记录，也可能会刷盘。**

**为什么呢？**

因为在事务执行过程 `redo log` 记录是会写入`redo log buffer` 中，这些 `redo log` 记录会被后台线程刷盘。

![image-20220724102448736](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220724102448736.png)

除了后台线程每秒`1`次的轮询操作，还有一种情况，当 `redo log buffer` 占用的空间即将达到 `innodb_log_buffer_size` 一半的时候，后台线程会主动刷盘。



### 不同刷盘策略的流程图

####  innodb_flush_log_at_trx_commit=0

![image-20220724102557484](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220724102557484.png)

#### innodb_flush_log_at_trx_commit=1

![image-20220724102627481](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220724102627481.png)

为`1`时， 只要事务提交成功，`redo log`记录就一定在硬盘里，不会有任何数据丢失。

如果事务执行期间`MySQL`挂了或宕机，这部分日志丢了，但是事务并没有提交，所以日志丢了也不会有损失。



#### innodb_flush_log_at_trx_commit=2

![image-20220724102720515](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220724102720515.png)

为`2`时， 只要事务提交成功，`redo log buffer`中的内容只写入文件系统缓存（`page cache`）。

如果仅仅只是`MySQL`挂了不会有任何数据丢失，但是宕机可能会有`1`秒数据的丢失。



### 日志文件组

硬盘上存储的 `redo log` 日志文件不只一个，而是以一个**日志文件组**的形式出现的，每个的`redo`日志文件大小都是一样的。

比如可以配置为一组`4`个文件，每个文件的大小是 `1GB`，整个 `redo log` 日志文件组可以记录`4G`的内容。

它采用的是环形数组形式，从头开始写，写到末尾又回到头循环写，如下图所示。

<img src="https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220724102822847.png" alt="image-20220724102822847" style="zoom:50%;" />

在个**日志文件组**中还有两个重要的属性，分别是 `write pos、checkpoint`

- **write pos** 是当前记录的位置，一边写一边后移
- **checkpoint** 是当前要擦除的位置，也是往后推移

每次刷盘 `redo log` 记录到**日志文件组**中，`write pos` 位置就会后移更新。

每次 `MySQL` 加载**日志文件组**恢复数据时，会清空加载过的 `redo log` 记录，并把 `checkpoint` 后移更新。

`write pos之后` 和 `checkpoint之前` 之间的还空着的部分可以用来写入新的 `redo log` 记录。

如果 `write pos` 追上 `checkpoint` ，表示**日志文件组**满了，这时候不能再写入新的 `redo log` 记录，`MySQL` 得停下来，清空一些记录，把 `checkpoint` 推进一下。



### 崩溃恢复怎么做？ LSN

LSN是Log Sequence Number的缩写，代表的是日志序列号，在InnoDB存储引擎中，占8个字节，且单调递增。

LSN代表的含义：

- 重做日志写入的总量
- checkpoint的位置
- 页的版本

> 例如当前重做日志的LSN为1000，有一个事务T写入了100字节的重做日志，那么LSN就变为了1100
>
> LSN不仅记录在重做日志中，还存在于每个页中，在页中，LSN表示该页最后刷新时LSN的大小

**因为重做日志记录的是每个页的日志，因此页中的LSN用来判断页是否需要进行恢复操作。**

> 例如页P1的LSN为10000，而数据库启动时，InnoDB检测到写入重做日志中的LSN为13000，并且事务已经提交，那么数据库就需要执行恢复操作，将重做日志应用到P1页中，对于重做日志中LSN小于P1页的LSN，那么不需要进行重做，因为P1页中的LSN表示页已经被刷新到该位置。

InnoDB存储引擎在启动时不管上次数据库运行时是否正常关闭，都会尝试进行恢复操作，因为重做日志记录的是物理日志，因此恢复的速度比逻辑日志，如二进制日志快很多

**由于checkpoint表示已经刷新到磁盘页上的LSN,因此在恢复过程中仅需恢复checkpoint开始的日志部分。**



### redo log 小结

**只要每次把修改后的数据页直接刷盘不就好了，还有 `redo log` 什么事？**

它们不都是刷盘么？差别在哪里？

```bash
1 Byte = 8bit
1 KB = 1024 Byte
1 MB = 1024 KB
1 GB = 1024 MB
1 TB = 1024 GB
```

实际上，数据页大小是`16KB`，刷盘比较耗时，可能就修改了数据页里的几 `Byte` 数据，有必要把完整的数据页刷盘吗？

而且数据页刷盘是随机写，因为一个数据页对应的位置可能在硬盘文件的随机位置，所以性能是很差。

如果是写 `redo log`，一行记录可能就占几十 `Byte`，只包含表空间号、数据页号、磁盘文件偏移 量、更新值，再加上是顺序写，所以刷盘速度很快。

所以用 `redo log` 形式记录修改内容，性能会远远超过刷数据页的方式，这也**让数据库的并发能力更强**。

- 更重要的,`redo log`用来保证事务的持久性，做崩溃恢复



## binlog

`redo log` 它是物理日志，记录内容是“在某个数据页上做了什么修改”，属于 `InnoDB` 存储引擎。

而 `binlog` 是逻辑日志，记录内容是语句的原始逻辑，类似于“给 ID=2 这一行的 c 字段加 1”，属于`MySQL Server` 层。

不管用什么存储引擎，只要发生了表数据更新，都会产生 `binlog` 日志。

那 `binlog` 到底是用来干嘛的？

可以说`MySQL`数据库的**数据备份、主备、主主、主从**都离不开`binlog`，需要依靠`binlog`来同步数据，保证数据一致性。

![image-20220724103830437](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220724103830437.png)

`binlog`会记录所有涉及更新数据的逻辑操作，并且是顺序写。

### binlog三种格式对比

`binlog` 日志有三种格式，可以通过`binlog_format`参数指定。

- **statement**
- **row**
- **mixed**

```sql
mysql> CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `a` int(11) DEFAULT NULL,
  `t_modified` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  KEY `a` (`a`),
  KEY `t_modified`(`t_modified`)
) ENGINE=InnoDB;

insert into t values(1,1,'2018-11-13');
insert into t values(2,2,'2018-11-12');
insert into t values(3,3,'2018-11-11');
insert into t values(4,4,'2018-11-10');
insert into t values(5,5,'2018-11-09');
```

如果要在表中删除一行数据的话，我们来看看这个delete语句的binlog是怎么记录的。

注意，下面这个语句包含注释，如果你用MySQL客户端来做这个实验的话，要记得加-c参数，否则客户端会自动去掉注释。

```sql
mysql> delete from t /*comment*/  where a>=4 and t_modified<='2018-11-10' limit 1;
```

当binlog_format=statement时，binlog里面记录的就是SQL语句的原文。你可以用

```sql
mysql> show binlog events in 'master.000001';
```

命令看binlog中的内容。

![image-20220827173634661](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220827173634661.png)

- 第二行是一个BEGIN，跟第四行的commit对应，表示中间是一个事务；
- 第三行就是真实执行的语句了。可以看到，在真实执行的delete命令之前，还有一个“use ‘test’”命令。这条命令不是我们主动执行的，而是MySQL根据当前要操作的表所在的数据库，自行添加的。这样做可以保证日志传到备库去执行的时候，不论当前的工作线程在哪个库里，都能够正确地更新到test库的表t。 use 'test’命令之后的delete 语句，就是我们输入的SQL原文了。可以看到，**binlog“忠实”地记录了SQL命令**，甚至连注释也一并记录了。

**为了说明statement 和 row格式的区别，我们来看一下这条delete命令的执行效果图：**

![image-20220827174323135](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220827174323135.png)

可以看到，运行这条delete命令产生了一个warning，原因是当前binlog设置的是statement格式，并且语句中有limit，所以这个命令可能是unsafe的。

为什么这么说呢？这是因为delete 带limit，很可能会出现主备数据不一致的情况。比如上面这个例子：

1. 如果delete语句使用的是索引a，那么会根据索引a找到第一个满足条件的行，也就是说删除的是a=4这一行；

1. 但如果使用的是索引t_modified，那么删除的就是 t_modified='2018-11-09’也就是a=5这一行。

由于statement格式下，记录到binlog里的是语句原文，因此可能会出现这样一种情况：在主库执行这条SQL语句的时候，用的是索引a；而在备库执行这条SQL语句的时候，却使用了索引t_modified。因此，MySQL认为这样写是有风险的。

那么，如果我把binlog的格式改为binlog_format=‘row’， 是不是就没有这个问题了呢？我们先来看看这时候binog中的内容吧。

![image-20220827174438562](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220827174438562.png)

可以看到，与statement格式的binlog相比，前后的BEGIN和COMMIT是一样的。但是，row格式的binlog里没有了SQL语句的原文，而是替换成了两个event：Table_map和Delete_rows。

1. Table_map event，用于说明接下来要操作的表是test库的表t;
2. Delete_rows event，用于定义删除的行为。

其实，我们通过图5是看不到详细信息的，还需要借助mysqlbinlog工具，用下面这个命令解析和查看binlog中的内容。因为图5中的信息显示，这个事务的binlog是从8900这个位置开始的，所以可以用start-position参数来指定从这个位置的日志开始解析。

```sql
mysqlbinlog  -vv data/master.000001 --start-position=8900;
```

![image-20220827174557958](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220827174557958.png)

从这个图中，我们可以看到以下几个信息：

1. server id 1，表示这个事务是在server_id=1的这个库上执行的。
2. 每个event都有CRC32的值，这是因为我把参数binlog_checksum设置成了CRC32。
3. Table_map event跟在图5中看到的相同，显示了接下来要打开的表，map到数字226。现在我们这条SQL语句只操作了一张表，如果要操作多张表呢？每个表都有一个对应的Table_map event、都会map到一个单独的数字，用于区分对不同表的操作。
4. 我们在mysqlbinlog的命令中，使用了-vv参数是为了把内容都解析出来，所以从结果里面可以看到各个字段的值（比如，@1=4、 @2=4这些值）。
5. **binlog_row_image的默认配置是FULL，因此Delete_event里面，包含了删掉的行的所有字段的值。如果把binlog_row_image设置为MINIMAL，则只会记录必要的信息，在这个例子里，就是只会记录id=4这个信息。**
6. 最后的Xid event，用于表示事务被正确地提交了。

**你可以看到，当binlog_format使用row格式的时候，binlog里面记录了真实删除行的主键id，这样binlog传到备库去的时候，就肯定会删除id=4的行，不会有主备删除不同行的问题。**

#### 为什么会有mixed格式的binlog？

- 因为有些statement格式的binlog可能会导致主备不一致，所以要使用row格式。

- 但row格式的缺点是，很占空间。比如你用一个delete语句删掉10万行数据，用statement的话就是一个SQL语句被记录到binlog中，占用几十个字节的空间。但如果用row格式的binlog，就要把这10万条记录都写到binlog中。这样做，不仅会占用更大的空间，同时写binlog也要耗费IO资源，影响执行速度。

- 所以，MySQL就取了个折中方案，也就是有了mixed格式的binlog。mixed格式的意思是，MySQL自己会判断这条SQL语句是否可能引起主备不一致，如果有可能，就用row格式，否则就用statement格式。

也就是说，mixed格式可以利用statment格式的优点，同时又避免了数据不一致的风险。

因此，如果你的线上MySQL设置的binlog格式是statement的话，那基本上就可以认为这是一个不合理的设置。你至少应该把binlog的格式设置为mixed。

比如我们这个例子，设置为mixed后，就会记录为row格式；而如果执行的语句去掉limit 1，就会记录为statement格式。

当然我要说的是，现在越来越多的场景要求把MySQL的binlog格式设置成row。这么做的理由有很多，我来给你举一个可以直接看出来的好处：**恢复数据**。

接下来，我们就分别从delete、insert和update这三种SQL语句的角度，来看看数据恢复的问题。

通过图6你可以看出来，即使我执行的是**delete语句**，row格式的binlog也会把被删掉的行的整行信息保存起来。所以，如果你在执行完一条delete语句以后，发现删错数据了，可以直接把binlog中记录的delete语句转成insert，把被错删的数据插入回去就可以恢复了。

如果你是执行错了**insert语句**呢？那就更直接了。row格式下，insert语句的binlog里会记录所有的字段信息，这些信息可以用来精确定位刚刚被插入的那一行。这时，你直接把insert语句转成delete语句，删除掉这被误插入的一行数据就可以了。

如果执行的是**update语句**的话，binlog里面会记录修改前整行的数据和修改后的整行数据。所以，如果你误执行了update语句的话，只需要把这个event前后的两行信息对调一下，再去数据库里面执行，就能恢复这个更新操作了。

其实，由delete、insert或者update语句导致的数据操作错误，需要恢复到操作之前状态的情况，也时有发生。MariaDB的[Flashback](https://mariadb.com/kb/en/library/flashback/)工具就是基于上面介绍的原理来回滚数据的。

虽然mixed格式的binlog现在已经用得不多了，但这里我还是要再借用一下mixed格式来说明一个问题，来看一下这条SQL语句：

```sql
mysql> insert into t values(10,10, now());
```

如果我们把binlog格式设置为mixed，你觉得MySQL会把它记录为row格式还是statement格式呢？

![image-20220827175025132](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220827175025132.png)

可以看到，MySQL用的居然是statement格式。你一定会奇怪，如果这个binlog过了1分钟才传给备库的话，那主备的数据不就不一致了吗？

接下来，我们再用mysqlbinlog工具来看看：

![image-20220827175043410](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220827175043410.png)

从图中的结果可以看到，原来binlog在记录event的时候，多记了一条命令：SET TIMESTAMP=1546103491。它用 SET TIMESTAMP命令约定了接下来的now()函数的返回时间。

因此，不论这个binlog是1分钟之后被备库执行，还是3天后用来恢复这个库的备份，这个insert语句插入的行，值都是固定的。也就是说，通过这条SET TIMESTAMP命令，MySQL就确保了主备数据的一致性。

我之前看过有人在重放binlog数据的时候，是这么做的：用mysqlbinlog解析出日志，然后把里面的statement语句直接拷贝出来执行。

你现在知道了，这个方法是有风险的。因为有些语句的执行结果是依赖于上下文命令的，直接执行的结果很可能是错误的。

所以，用binlog来恢复数据的标准做法是，用 mysqlbinlog工具解析出来，然后把解析结果整个发给MySQL执行。类似下面的命令：

```Plaintext
mysqlbinlog master.000001  --start-position=2738 --stop-position=2973 | mysql -h127.0.0.1 -P13000 -u$user -p$pwd;
```

这个命令的意思是，将 master.000001 文件里面从第2738字节到第2973字节中间这段内容解析出来，放到MySQL去执行。



#### 读已提交（RC）下需要用row格式

那Mysql在5.0这个版本以前，binlog只支持`STATEMENT`这种格式！而这种格式在**读已提交(Read Commited)**这个隔离级别下主从复制是有bug的，因此Mysql将**可重复读(Repeatable Read)**作为默认的隔离级别！
接下来，就要说说当binlog为`STATEMENT`格式，且隔离级别为**读已提交(Read Commited)**时，有什么bug呢？如下图所示，在主(master)上执行如下事务

![image-20241019145043637](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20241019145043637.png)

此时在主(master)上执行下列语句

```cobol
select * from test；
```

输出如下

```cobol
+---+
| b |
+---+
| 3 |
+---+
1 row in set
```

但是，你在此时在从(slave)上执行该语句，得出输出如下

```vbscript
Empty set
```

这样，你就出现了主从不一致性的问题！原因其实很简单，就是在master上执行的顺序为先删后插！而此时binlog为STATEMENT格式，它记录的顺序为先插后删！从(slave)同步的是binglog，因此从机执行的顺序和主机不一致！就会出现主从不一致！

- Session2先提交，Session1再提交，因此binlog里面记录的是，先插后删，但是在RC下，Session1没有受到阻塞，它先删了，然后Session2也没有受到阻塞，它后插入
- 因此在主机上，先删后插，会有数据出现，在从机上，binlog读出来是先插后删，数据插入了但又被删除了，因此从机没有数据，造成了主从不一致。

*如何解决？*
解决方案有两种！
(1)隔离级别设为**可重复读(Repeatable Read)**,在该隔离级别下引入间隙锁。当`Session 1`执行delete语句时，会锁住间隙。那么，`Ssession 2`执行插入语句就会阻塞住！
(2)将binglog的格式修改为row格式，此时是基于行的复制，自然就不会出现sql执行顺序不一样的问题！奈何这个格式在mysql5.1版本开始才引入。因此由于历史原因，mysql将默认的隔离级别设为**可重复读(Repeatable Read)**，保证主从复制不出问题！



### 写入机制

`binlog`的写入时机也非常简单，事务执行过程中，先把日志写到`binlog cache`，事务提交的时候，再把`binlog cache`写到`binlog`文件中。

因为一个事务的`binlog`不能被拆开，无论这个事务多大，也要确保一次性写入，所以系统会给每个线程分配一个块内存作为`binlog cache`。

我们可以通过`binlog_cache_size`参数控制单个线程 binlog cache 大小，如果存储内容超过了这个参数，就要暂存到磁盘（`Swap`）。

`binlog`日志刷盘流程如下

![image-20220724105711925](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220724105711925.png)

- **上图的 write，是指把日志写入到文件系统的 page cache，并没有把数据持久化到磁盘，所以速度比较快**
- **上图的 fsync，才是将数据持久化到磁盘的操作**

`write`和`fsync`的时机，可以由参数`sync_binlog`控制，默认是`0`。

为`0`的时候，表示每次提交事务都只`write`，由系统自行判断什么时候执行`fsync`。

- 虽然性能得到提升，但是机器宕机，`page cache`里面的 binlog 会丢失。

为了安全起见，可以设置为`1`，表示每次提交事务都会执行`fsync`，就如同 **redo log 日志刷盘流程** 一样。

最后还有一种折中方式，可以设置为`N(N>1)`，表示每次提交事务都`write`，但累积`N`个事务后才`fsync`。

- 做备份的时候，
- 从库落后主库太多了，让从库的数据追上主库的时候,
- 在出现`IO`瓶颈的场景里
- 将`sync_binlog`设置成一个比较大的值，可以提升性能。同样的，如果机器宕机，会丢失最近`N`个事务的`binlog`日志



## undo log

我们知道如果想要保证事务的原子性，就需要在异常发生时，对已经执行的操作进行**回滚**，在 MySQL 中，恢复机制是通过 **回滚日志（undo log）** 实现的，所有事务进行的修改都会先记录到这个回滚日志中，然后再执行相关的操作。如果执行过程中遇到异常的话，我们直接利用 **回滚日志** 中的信息将数据回滚到修改之前的样子即可！并且，回滚日志会先于数据持久化到磁盘上。这样就保证了即使遇到数据库突然宕机等情况，当用户再次启动数据库的时候，数据库还能够通过查询回滚日志来回滚将之前未完成的事务。

undo是逻辑日志，只是将数据库逻辑地恢复到原来的样子。所有的修改都被逻辑地取消了，但是数据结构和页本身在回滚之后可能不相同

- 当InnoDB存储引擎回滚时，它实际上做的是与向前相反的工作
- 对于每个insert，InnoDB会完成一个delete
- 对于每个delete，InnoDB会完成一个insert
- 对于每个update，InnoDB会完成一个相反的update，将修改前的行放回去

另外，`MVCC` 的实现依赖于：**隐藏字段、Read View、undo log**。在内部实现中，`InnoDB` 通过数据行的 `DB_TRX_ID` 和 `Read View` 来判断数据的可见性，如不可见，则通过数据行的 `DB_ROLL_PTR` 找到 `undo log` 中的历史版本。每个事务读到的数据版本可能是不一样的，在同一个事务中，用户只能看到该事务创建 `Read View` 之前已经提交的修改和该事务本身做的修改



## 两阶段提交

`redo log`（重做日志）让`InnoDB`存储引擎拥有了崩溃恢复能力。（存储引擎层）

`binlog`（归档日志）保证了`MySQL`集群架构的数据一致性。（Server层）

虽然它们都属于持久化的保证，但是侧重点不同。

在执行更新语句过程，会记录`redo log`与`binlog`两块日志，以基本的事务为单位，`redo log`在事务执行过程中可以不断写入，而`binlog`只有在提交事务时才写入，所以`redo log`与`binlog`的写入时机不一样。

为了解决两份日志之间的逻辑一致问题，`InnoDB`存储引擎使用**两阶段提交**方案。

原理很简单，将`redo log`的写入拆成了两个步骤`prepare`和`commit`，这就是**两阶段提交**。

<img src="https://raw.githubusercontent.com/xuhaoyao/images/master/img/2e5bff4910ec189fe1ee6e2ecc7b4bbe.png" alt="2e5bff4910ec189fe1ee6e2ecc7b4bbe" style="zoom: 33%;" />

### 为什么要用两份日志？

为什么要用两份日志？`redo log 和 binlog.`

MySQL自带的引擎是MyISAM，它不支持事务，而InnoDB 引擎就是通过 redo log 来支持事务的，binlog只能用来归档（POINT-IN-TIME）以及主从复制高可用这些。



### MySQL怎么知道binlog是完整的?

一个事务的binlog是有完整格式的：

- statement格式的binlog，最后会有COMMIT；
  - ![image-20220724115143764](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220724115143764.png)

- row格式的binlog，最后会有一个XID event。
  - ![image-20220724115153862](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220724115153862.png)

另外，在MySQL 5.6.2版本以后，还引入了**binlog-checksum**参数【校验和】，用来验证binlog内容的正确性。对于binlog日志由于磁盘原因，可能会在日志中间出错的情况，MySQL可以通过校验checksum的结果来发现。所以，MySQL还是有办法验证事务binlog的完整性的。



### redo log 和 binlog是怎么关联起来的?

它们有一个共同的数据字段，叫XID。崩溃恢复的时候，会按顺序扫描redo log：

- 如果碰到既有prepare、又有commit的redo log，就直接提交；
- 如果碰到只有parepare、而没有commit的redo log，就拿着XID去binlog找对应的事务。



### 处于prepare阶段的redo log加上完整binlog，重启就能恢复，MySQL为什么要这么设计?

回答：其实，这个问题还是跟我们在反证法中说到的数据与备份的一致性有关。在时刻B，也就是binlog写完以后MySQL发生崩溃，这时候binlog已经写入了，之后就会被从库（或者用这个binlog恢复出来的库）使用。

所以，在主库上也要提交这个事务。采用这个策略，主库和备库的数据就保证了一致性。



### 如果这样的话，为什么还要两阶段提交呢？干脆先redo log写完，再写binlog。崩溃恢复的时候，必须得两个日志都完整才可以。是不是一样的逻辑？

回答：其实，两阶段提交是经典的分布式系统问题，并不是MySQL独有的。

如果必须要举一个场景，来说明这么做的必要性的话，那就是事务的持久性问题。

对于InnoDB引擎来说，如果redo log提交完成了，事务就不能回滚（如果这还允许回滚，就可能覆盖掉别的事务的更新）。而如果redo log直接提交，然后binlog写入的时候失败，InnoDB又回滚不了，数据和binlog日志又不一致了。

两阶段提交就是为了给所有人一个机会，当每个人都说“我ok”的时候，再一起提交。



### 不引入两个日志，也就没有两阶段提交的必要了。只用binlog来支持崩溃恢复，又能支持归档，不就可以了？

意思是，只保留binlog，然后可以把提交流程改成这样：… -> “数据更新到内存” -> “写 binlog” -> “提交事务”，是不是也可以提供崩溃恢复的能力？

答案是不可以。

历史原因：

- InnoDB并不是MySQL的原生存储引擎。MySQL的原生引擎是MyISAM，设计之初就有没有支持崩溃恢复。
- InnoDB在作为MySQL的插件加入MySQL引擎家族之前，就已经是一个提供了崩溃恢复和事务支持的引擎了。
- InnoDB接入了MySQL后，发现既然binlog没有崩溃恢复的能力，那就用InnoDB原有的redo log好了。

实现原因（binlog为什么不支持崩溃恢复？）：

- 看这个流程
  - prepare1 -> binlog1 -> commit1 -> prepare2 -> binlog2  -> commit2
  - 假设在binlog2写完后，还没有commit2，发生了crash
- 主要就是一个点：binlog没有能力恢复数据页
  - 重启后，引擎内部事务2会回滚，然后应用binlog2可以补回来；但是对于事务1来说，系统已经认为提交完成了，不会再应用一次binlog1。
  - 但是，InnoDB引擎使用的是WAL技术，执行事务的时候，写完内存和日志，事务就算完成了。如果之后崩溃，要依赖于日志来恢复数据页。
    - 也就是说在上述位置发生崩溃的话，事务1也是可能丢失了的，而且是数据页级的丢失【**事务1当时对应的数据页是脏页，还没有刷回磁盘，此时崩溃的话需要有崩溃恢复机制来找回数据**】。此时，binlog里面并没有记录数据页的更新细节，是补不回来的。
  - 你如果要说，那我优化一下binlog的内容，让它来记录数据页的更改可以吗？但，这其实就是又做了一个redo log出来。
  - 所以，至少现在的binlog能力，还不能支持崩溃恢复。



### 那能不能反过来，只用redo log，不要binlog？

如果只从崩溃恢复的角度来讲是可以的。你可以把binlog关掉，这样就没有两阶段提交了，但系统依然是crash-safe的。

但是，正式的生产库上，binlog都是开着的。因为binlog有着redo log无法替代的功能。

一个是归档。redo log是循环写，写到末尾是要回到开头继续写的。这样历史日志没法保留，redo log也就起不到归档的作用。

一个就是MySQL系统依赖于binlog。binlog作为MySQL一开始就有的功能，被用在了很多地方。其中，MySQL系统高可用的基础，就是binlog复制。

还有很多公司有异构系统（比如一些数据分析系统），这些系统就靠消费MySQL的binlog来更新自己的数据。关掉binlog的话，这些下游系统就没法输入了。

总之，由于现在包括MySQL高可用在内的很多系统机制都依赖于binlog，所以“鸠占鹊巢”redo log还做不到。你看，发展生态是多么重要。



### redo log一般设置多大？

redo log太小的话，会导致很快就被写满，然后不得不强行刷redo log，这样WAL机制的能力就发挥不出来了。

所以，如果是现在常见的几个TB的磁盘的话，就不要太小气了，直接将redo log设置为4个文件、每个文件1GB吧。



### 为什么日志需要两阶段提交？

为什么 redo log 要引入 prepare 预提交状态？这里我们用反证法来说明下为什么要这么做。

仍然用前面的update语句来做例子。假设当前ID=2的行，字段c的值是0，再假设执行update语句过程中在写完第一个日志后，第二个日志还没有写完期间发生了crash，会出现什么情况呢？

1、**先写redo log,后写binlog**：假设写完 redo log 后，机器挂了，binlog 日志没有被写入，那么机器重启后，这台机器会通过 redo log 恢复数据，但是这个时候 binlog 并没有记录该数据，后续进行机器备份的时候，就会丢失这一条数据，同时主从同步也会丢失这一条数据。 

2、**先写binlog,后写redo log：**假设写完binlog后，机器挂了，机器重启后不用恢复，但是下次进行备份的时候就会出现问题，“数据多了”，与原来产生不同

可以看到，如果不使用“两阶段提交”，那么数据库的状态就有可能和用它的日志恢复出来的库的状态不一致。



### 两阶段提交不同时刻的异常处理

<img src="https://raw.githubusercontent.com/xuhaoyao/images/master/img/ee9af616e05e4b853eba27048351f62a.jpg" alt="ee9af616e05e4b853eba27048351f62a" style="zoom:33%;" />



如果在时刻A发生崩溃，这时候还没有写入binlog,事务也没有commit状态，那么此次事务宣告失败，崩溃恢复的时候，事务就会回滚。

如果在时刻B发生崩溃，也就是binlog写完，redo log还没commit前发生crash，那崩溃恢复的时候MySQL会怎么处理？

我们先来看一下崩溃恢复时的判断规则。

1. 如果redo log里面的事务是完整的，也就是已经有了commit标识，则直接提交；
2. 如果redo log里面的事务只有完整的prepare，则**判断对应的事务binlog是否存在并完整**：
   a. 如果是，则提交事务；
   b. 否则，回滚事务。

这里，时刻B发生crash对应的就是2(a)的情况，崩溃恢复过程中事务会被提交。



## 缓冲池及LRU算法

InnoDB存储引擎是基于磁盘存储的，并将其中的记录按照页的方式进行管理。在数据库系统中，由于CPU速度与磁盘速度之间的鸿沟，基于磁盘的数据库系统通常使用缓冲池技术来提高数据库的整体性能。

- 在数据库读取页的操作，首先将从磁盘读到的页存放在缓冲池中，下一次再读相同的页时，首先判断该页是否再缓冲池中。若再缓冲池中，称该页再缓冲池被命中，直接读取该页，否则读取磁盘上的页，再放入缓冲池中
- 对于数据库中页的修改操作，则首先修改再缓冲池中的页，然后再以一定的频率刷新到磁盘上（checkpoint）
- 缓冲池中缓存的数据页类型：索引页、数据页、undo页、插入缓冲、Innodb存储的锁信息等

数据库中的缓冲池时通过LRU算法来进行管理的，即最频繁使用的页在LRU列表的前端，而最少使用的页在LRU列表的尾端。当缓冲池不能存放新读取到的页时，将首先释放LRU列表中尾端的页。

- 缓冲池中页的大小默认为16KB

**InnoDB存储引擎对传统LRU算法做了一些优化。**

- LRU列表加入了midpoint位置。新读取到的页，放入到midpoint位置
  - midpoint默认是37
  - innodb_old_blocks_pct=37
- 靠近LRU头部的5/8是yound区、后面3/8是old区，新读取的页插入到old区的头部，可以理解为yound区的都是最为活跃的数据。

**为什么要改造LRU算法？**

- 若直接将新读取到的页放入LRU的首部，那么某些SQL操作可能会使缓冲池中的页被刷出去很多（全表扫描），这类操作需要访问表中的很多页，但这些页可能仅仅是这次想查询的，并不是活跃的热点数据，如果新读取的页被放入LRU链表头部，非常可能将所需要的热点数据页从LRU列表中移除，而在下一次需要读取该页时，Innodb需要再次访问磁盘。
- innodb_old_blocks_time，用于表示页读取到midpoint位置后需要等待多久才会被加入到LRU列表的热端



## checkpoint

Checkpoint技术的目的是解决下面三个问题：

- 缩短数据库的恢复时间
  - 但数据库发生宕机时，数据库不需要重做所有日志，因为checkpoint之前的页都已经刷新回磁盘了。故数据库只需对Checkpoint后的重做日志进行恢复。这样就大大缩短了恢复的时间
- 缓冲池不够用时，将脏页刷新到磁盘
  - 根据LRU算法溢出最近最少使用的页，若此页为脏页，那么需要强制执行Checkpoint，将脏页也就是页的新版本刷回磁盘
- 重做日志不可用时，刷新脏页
  - redo log是循环使用的，并不是让其无限增大
  - 当重做日志写满了，此时必须强制checkpoint，将缓冲池中的页至少刷新到当前重做日志的位置。





## WAL

WAL的全称是Write-Ahead Logging，它的关键点就是先写日志，再写磁盘，具体来说，当有一条记录需要更新的时候，InnoDB引擎就会先把记录写到redo log里面，并更新内存，这个时候更新就算完成了。同时，InnoDB引擎会在适当的时候，将这个操作记录更新到磁盘里面，而这个更新往往是在系统比较空闲的时候做



## change buffer

当需要更新一个数据页时，如果数据页在内存中就直接更新，而如果这个数据页还没有在内存中的话，在不影响数据一致性的前提下，InooDB会将这些更新操作缓存在change buffer中，这样就不需要从磁盘中读入这个数据页了。在下次查询需要访问这个数据页的时候，将数据页读入内存，然后执行change buffer中与这个页有关的操作。通过这种方式就能保证这个数据逻辑的正确性。

需要说明的是，虽然名字叫作change buffer，实际上它是可以持久化的数据。也就是说，change buffer在内存中有拷贝，也会被写入到磁盘上。

**将change buffer中的操作应用到原数据页，得到最新结果的过程称为merge**。除了访问这个数据页会触发merge外，系统有后台线程会定期merge。在数据库正常关闭（shutdown）的过程中，也会执行merge操作。

> merge的执行流程是这样的：
>
> 1. 从磁盘读入数据页到内存（老版本的数据页）；
> 2. 从change buffer里找出这个数据页的change buffer 记录(可能有多个），依次应用，得到新版数据页；
> 3. 写redo log。这个redo log包含了数据的变更和change buffer的变更。
>
> 到这里merge过程就结束了。这时候，数据页和内存中change buffer对应的磁盘位置都还没有修改，属于脏页，之后各自刷回自己的物理数据，就是另外一个过程了。

显然，如果能够将更新操作先记录在change buffer，减少读磁盘，语句的执行速度会得到明显的提升。而且，数据读入内存是需要占用buffer pool的，所以这种方式还能够避免占用内存，提高内存利用率。

那么，**什么条件下可以使用change buffer呢？**

对于唯一索引来说，所有的更新操作都要先判断这个操作是否违反唯一性约束。比如，要插入(4,400)这个记录，就要先判断现在表中是否已经存在k=4的记录，而这必须要将数据页读入内存才能判断。如果都已经读入到内存了，那直接更新内存会更快，就没必要使用change buffer了。

**因此，唯一索引的更新就不能使用change buffer，实际上也只有普通索引可以使用。**

change buffer用的是buffer pool里的内存，因此不能无限增大。change buffer的大小，可以通过参数innodb_change_buffer_max_size来动态设置。这个参数设置为50的时候，表示change buffer的大小最多只能占用buffer pool的50%。

> **如果要在这张表中插入一个新记录(4,400)的话，InnoDB的处理流程是怎样的。**
>
> 第一种情况是，**这个记录要更新的目标页在内存中**。这时，InnoDB的处理流程如下：
>
> - 对于唯一索引来说，找到3和5之间的位置，判断到没有冲突，插入这个值，语句执行结束；
> - 对于普通索引来说，找到3和5之间的位置，插入这个值，语句执行结束。
>
> 这样看来，普通索引和唯一索引对更新语句性能影响的差别，只是一个判断，只会耗费微小的CPU时间。
>
> 但，这不是我们关注的重点。
>
> 第二种情况是，**这个记录要更新的目标页不在内存中**。这时，InnoDB的处理流程如下：
>
> - 对于唯一索引来说，需要将数据页读入内存，判断到没有冲突，插入这个值，语句执行结束；
> - 对于普通索引来说，则是将更新记录在change buffer，语句执行就结束了。

**将数据从磁盘读入内存涉及随机IO的访问，是数据库里面成本最高的操作之一。change buffer因为减少了随机磁盘访问，所以对更新性能的提升是会很明显的。**

**现在有一个问题就是：普通索引的所有场景，使用change buffer都可以起到加速作用吗？**

因为merge的时候是真正进行数据更新的时刻，而change buffer的主要目的就是将记录的变更动作缓存下来，所以在一个数据页做merge之前，change buffer记录的变更越多（也就是这个页面上要更新的次数越多），收益就越大。

因此，对于写多读少的业务来说，页面在写完以后马上被访问到的概率比较小，此时change buffer的使用效果最好。这种业务模型常见的就是账单类、日志类的系统。

反过来，假设一个业务的更新模式是写入之后马上会做查询，那么即使满足了条件，将更新先记录在change buffer，但之后由于马上要访问这个数据页，会立即触发merge过程。这样随机访问IO的次数不会减少，反而增加了change buffer的维护代价。所以，对于这种业务模式来说，change buffer反而起到了副作用。

> 如果所有的更新后面，都马上伴随着对这个记录的查询，那么你应该关闭change buffer。而在其他情况下，change buffer都能提升更新性能。



## change buffer 和 redo log

现在，我们要在表上执行这个插入语句：

```bash
mysql> insert into t(id,k) values(id1,k1),(id2,k2);
```

k1所在的数据页在内存(InnoDB buffer pool)中，k2所在的数据页不在内存中。如图2所示是带change buffer的更新状态图。

![image-20220724193357769](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220724193357769.png)

这条更新语句做了如下的操作（按照图中的数字顺序）：

1. Page 1在内存中，直接更新内存；
2. Page 2没有在内存中，就在内存的change buffer区域，记录下“我要往Page 2插入一行”这个信息
3. 将上述两个动作记入redo log中（图中3和4）。

做完上面这些，事务就可以完成了。所以，你会看到，执行这条更新语句的成本很低，就是写了两处内存，然后写了一处磁盘（两次操作合在一起写了一次磁盘），而且还是顺序写的。

同时，图中的两个虚线箭头，是后台操作，不影响更新的响应时间。

那在这之后的读请求，要怎么处理呢？

1. 读Page 1的时候，直接从内存返回。WAL之后如果读数据，是不是一定要读盘，是不是一定要从redo log里面把数据更新以后才可以返回？其实是不用的。虽然磁盘上还是之前的数据，但是这里直接从内存返回结果，结果是正确的。
2. 要读Page 2的时候，需要把Page 2从磁盘读入内存中，然后应用change buffer里面的操作日志(merge)，生成一个正确的版本并返回结果。

可以看到，直到需要读Page 2的时候，这个数据页才会被读入内存。

所以，如果要简单地对比这两个机制在提升更新性能上的收益的话，**redo log 主要节省的是随机写磁盘的IO消耗（转成顺序写），而change buffer主要节省的则是随机读磁盘的IO消耗。**

**change buffer一开始是写内存的，那么如果这个时候机器掉电重启，会不会导致change buffer丢失呢？change buffer丢失可不是小事儿，再从磁盘读入数据可就没有了merge过程，就等于是数据丢失了。会不会出现这种情况呢？**

- 虽然是只更新内存，但是在事务提交的时候，我们把change buffer的操作也记录到redo log里了，所以崩溃恢复的时候，change buffer也能找回来。



### 正常运行中的实例，数据写入后的最终落盘，是从redo log更新过来的还是从buffer pool更新过来的呢？

实际上，redo log并没有记录数据页的完整数据，所以它并没有能力自己去更新磁盘数据页，也就不存在“数据最终落盘，是由redo log更新过去”的情况。

1. 如果是正常运行的实例的话，数据页被修改以后，跟磁盘的数据页不一致，称为脏页。最终数据落盘，就是把内存中的数据页写盘。这个过程，甚至与redo log毫无关系。
2. 在崩溃恢复场景中，InnoDB如果判断到一个数据页可能在崩溃恢复的时候丢失了更新，就会将它读到内存，然后让redo log更新内存内容。更新完成后，内存页变成脏页，就回到了第一种情况的状态。





## 刷脏页

> 一条SQL语句，正常执行的时候特别快，但是有时也不知道怎么回事，它就会变得特别慢，并且这样的场景很难复现，它不只随机，而且持续时间还很短。
>
> 看上去，这就像是数据库“抖”了一下。
>
> - 这可能是MySQL正在刷脏页

当内存数据页跟磁盘数据页内容不一致的时候，我们称这个内存页为“脏页”。内存数据写入到磁盘后，内存和磁盘上的数据页的内容就一致了，称为“干净页”。

什么情况会引发数据库的flush过程呢？

- InnoDB的redo log写满了。这时候系统会停止所有更新操作，把checkpoint往前推进，redo log留出空间可以继续写。
  - 这种情况是InnoDB要尽量避免的。因为出现这种情况的时候，整个系统就不能再接受更新了，所有的更新都必须堵住。如果你从监控上看，这时候更新数会跌为0。
- 系统内存不足。当需要新的内存页，而内存不够用的时候，就要淘汰一些数据页，空出内存给别的数据页使用。如果淘汰的是“脏页”，就要先将脏页写到磁盘。
  - 这种情况其实是常态。
  - **InnoDB用缓冲池（buffer pool）管理内存，缓冲池中的内存页有三种状态：**
    - 第一种是，还没有使用的；
    - 第二种是，使用了并且是干净页；
    - 第三种是，使用了并且是脏页。
- MySQL认为系统“空闲”的时候，只要有机会就刷一点“脏页”。
- MySQL正常关闭的情况。这时候，MySQL会把内存的脏页都flush到磁盘上，这样下次MySQL启动的时候，就可以直接从磁盘上读数据，启动速度会很快。

当要读入的数据页没有在内存的时候，就必须到缓冲池中申请一个数据页。这时候只能把最久不使用的数据页从内存中淘汰掉：如果要淘汰的是一个干净页，就直接释放出来复用；但如果是脏页呢，就必须将脏页先刷到磁盘，变成干净页后才能复用。

所以，刷脏页虽然是常态，但是出现以下这两种情况，都是会明显影响性能的：

1. 一个查询要淘汰的脏页个数太多，会导致查询的响应时间明显变长；
2. 日志写满，更新全部堵住，写性能跌为0，这种情况对敏感业务来说，是不能接受的。

一旦一个查询请求需要在执行过程中先flush掉一个脏页时，这个查询就可能要比平时慢了。而MySQL中的一个机制，可能让你的查询会更慢：在准备刷一个脏页的时候，如果这个数据页旁边的数据页刚好是脏页，就会把这个“邻居”也带着一起刷掉；而且这个把“邻居”拖下水的逻辑还可以继续蔓延，也就是对于每个邻居数据页，如果跟它相邻的数据页也还是脏页的话，也会被放到一起刷。

在InnoDB中，innodb_flush_neighbors 参数就是用来控制这个行为的，值为1的时候会有上述的**“连坐”机制**，值为0时表示不找邻居，自己刷自己的。

找“邻居”这个优化在机械硬盘时代是很有意义的，可以减少很多随机IO。机械硬盘的随机IOPS一般只有几百，相同的逻辑操作减少随机IO就意味着系统性能的大幅度提升。

而如果使用的是SSD这类IOPS比较高的设备的话，我就建议你把innodb_flush_neighbors的值设置成0。因为这时候IOPS往往不是瓶颈，而“只刷自己”，就能更快地执行完必要的刷脏页操作，减少SQL语句响应时间。

在MySQL 8.0中，innodb_flush_neighbors参数的默认值已经是0了。



