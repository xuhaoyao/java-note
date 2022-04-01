# 有序集合底层实现？

有两种编码方式：ziplist和skiplist。

> ziplist是Redis为了节约内存而开发的，由一系列特殊编码的连续内存块组成的顺序型数据结构、
>
> - 查找数据，从尾部往前找，每一个entry有一个previous_entry_length记录了前一项的长度，而ziplist有一个属性字段zltail，可以得到尾部的地址，因此可以从后面一直遍历到前面
> - 由于存放在连续内存，若其中某一个entry修改了，使它占用的字节变大，可能造成连锁更新
> - zadd price 8.5 apple 5.0 banana 6.0 cherry
> - 有序集合保存的元素成员个数小于128，或者成员的大小都小于64字节的时候，使用ziplist编码，否则使用skiplist编码。



```bash
127.0.0.1:6379> zadd price 3.0 apple 2.0 banana 7.0 cherry
(integer) 3
127.0.0.1:6379> zrank price apple
(integer) 1
127.0.0.1:6379> zrank price banana
(integer) 0
127.0.0.1:6379> zrange price 0 1
1) "banana"
2) "apple"
127.0.0.1:6379> zrange price 0 2
1) "banana"
2) "apple"
3) "cherry"
127.0.0.1:6379> zrange price 0 3
1) "banana"
2) "apple"
3) "cherry"
127.0.0.1:6379> zrange price 0 -1
1) "banana"
2) "apple"
3) "cherry"
127.0.0.1:6379> zscore price cherry
"7"
```





## skiplist

skiplist编码的有序集合对象使用zset结构作为底层实现，一个zset结构同时包含一个字典和跳跃表。

**跳表插入和删除**
https://www.jianshu.com/p/9d8296562806

```c
typedef struct zset{
    zskiplist *zsl;
    dict *dict;
}zset;
```

- zsl跳跃表按分值从小到大保存了所有集合元素，每个跳跃表节点都保存了一个集合元素，跳跃表节点的object属性保存了元素的成员，而跳跃表节点的score属性则保存了元素的分值。通过这个跳跃表，可以对有序集合进行范围型操作，比如zrank,zrange等。
- dict字典创建了**成员到分值**的映射。zscore通过O(1)时间查找给定成员的分值。

> 跳跃表是一种有序数据结构，它通过在每个节点中维持多个指向其他节点的指针，从而达到快速访问节点的目的。
>
> ```c
> typedef struct zskiplistNode {
> 
>     // member 对象
>     robj *obj;
> 
>     // 分值
>     double score;
> 
>     // 后退指针
>     struct zskiplistNode *backward;
> 
>     // 层
>     struct zskiplistLevel {
> 
>         // 前进指针
>         struct zskiplistNode *forward;  //遍历就通过前进指针
> 
>         // 这个层跨越的节点数量,用来计算排位
>         unsigned int span;
> 
>     } level[]; //幂次定律生成层,[1,32]越大的数,得到的概率越小,期望是1/(1-p),Redis把p设置成0.25,平均是1.33层
> 
> } zskiplistNode;
> ```
>
> 



## 为什么有序集合需要同时使用跳跃表和字典来实现？

- 若只使用字典来实现有序集合，可以O（1）查找成员的分值，但字典以无序的方式来保存集合元素，所以每次在执行范围型操作如zrank,zrange等命令时,要怎么做？可能的做法就是对所有元素进行快排 O（NLogN），以及额外的O(N)空间复杂度。
- 若只使用跳跃表，zrank和zrange可以O（LogN），但是没了字典，zscore根据成员查找分值的操作就需要从O（1）变成O（LogN）了。
- 因此Redis组合了两种数据结构的特点，同时选择跳跃表和字典来实现有序集合。
- 字典和跳跃表会共享元素的成员和分值，并不会造成任何的数据重复，不会浪费任何内存。



## Redis为什么用跳跃表而不是红黑树？

- 首先，红黑树是一种很高级的数据结构，它实现起来比较复杂，而跳表实现起来相对简单一些。它们增删改查的效率都是O（logn）

- 范围查找的话，两者如何做?
  - 跳跃表只需要找到最小的节点，然后在第一层沿着前向指针顺序遍历就可以了。
  - 红黑树实际上也是一颗二叉树，二叉树范围查找，中序遍历，如何中序遍历？如果不维护一个最左边的节点的话，就需要通过递归或者迭代的方式来中序遍历，而这时间复杂度是O（N），空间复杂度也是O（N）。若维护最左边的节点，又很复杂，红黑树要保持平衡，最左边的节点就没那么容易维护。
- 插入删除的话，两者如何做？
  - 跳跃表只需要修改相邻节点的指针就好了，操作简单快速。
  - 红黑树实际上也是一颗平衡树，插入删除可能引起子树的调整。逻辑复杂。
- 内存占用情况？
  - 跳跃表每个节点包含的层级平均是1/(1−*p*)，p在Redis里面设置成了0.25,故每个节点平均1.33个指针
  - 而红黑树有左孩子和右孩子，包含两个指针