## MySQL为什么用B+树

**核心就是利用B+树存储千万级别的数据，树高也就三层，一般我们让树的第一层常驻内存，这样每次读取数据行的话，只需要两次磁盘I/O，效率很高**

- Hash索引(散列表)：如果只查询单个值的话，hash 索引的效率非常高，时间复杂度为O(1)。但是 hash 索引有几个问题：1）不支持范围查询；2）不支持索引值的排序操作；3）不支持联合索引的最左匹配规则。;4）可能有hash冲突。
- 平衡二叉树（红黑树）：查询性能也好，时间复杂度O(logn)，中序遍历可以得到一个从小到大有序的数据序列，但不支持区间查找。**而且由于是二叉树，当数据量很大时树的层数就会很高**，从树的根结点向下寻找的过程，每读1个节点，都相当于一次IO操作，因此他的I/O操作会比B+树多的多。
- B树索引：**B+树的中间节点存的是索引，不存储数据**，数据都保存在叶子节点中，而**B树的所有节点都能存放数据**。所以B+树 磁盘读写的代价比B树低，因为中间节点不放数据，所以**相同的磁盘块能存放更多的节点**，一次性读入内存的节点数量也就越多，所以IO读写次数就降低了。
- 跳表：是一种链表加多层索引的结构，时间复杂度O(logn)，支持区间查找，而B+树是一种多叉树，可以让每个节点大小等于操作系统每次读取页的大小，从而使读取节点时只需要进行一次IO即可。而且同数量级的数据，跳表索引的高度会比 B+ 树的高，导致 IO 读取次数多，影响查询性能。
  - 跳表，虽然可以跳跃式的查询，但是归根到底它还是类似于一种链表，它在最坏情况下可能达到O（N），N次的磁盘I/O可就太慢了，而MySQL对于千万级别的数据，也可以保持稳定的磁盘I/O



### B+树一个节点有多大？一千万条数据，B+树多高？

- **B+树一个节点的大小设为一页或页的倍数最为合适**。因为如果一个节点的大小 < 1页，那么读取这个节点的时候其实读取的还是一页，这样就造成了资源的浪费。
- 在 MySQL 中 B+ 树的一个节点大小为**“1页”，也就是16k**。之所以设置为一页，是因为对于大部分业务，一页就足够了：
- 对于叶子节点，如果一行数据大小为1k，那么一页就能存16条数据；对于非叶子节点，如果key使用的是bigint，则为8字节，指针在mysql中为6字节，一共是14字节，则16k能存放 16 * 1024 / 14 = 1170 个索引指针。于是可以算出，对于一颗高度为2的B+树，根节点存储索引指针节点，那么它有1170个叶子节点存储数据，每个叶子节点可以存储16条数据，一共 1170 x 16 = 18720 条数据。而对于高度为3的B+树，就可以存放 1170 x 1170 x 16 = 21902400 条数据（两千多万条数据），也就是对于两千多万条的数据，我们只需要高度为3的B+树就可以完成，通过主键查询只需要3次IO操作就能查到对应数据。所以在 InnoDB 中B+树高度一般为3层时，就能满足千万级的数据存储，所以一个节点为1页，也就是16k是比较合理的。	