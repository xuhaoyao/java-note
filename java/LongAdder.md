# LongAdder

- AtomicLong使用内部变量value保存着实际的long值，所有的操作 都是针对该value变量进行的。也就是说，在高并发环境下，value变 量其实是一个热点，也就是N个线程竞争一个热点。重试线程越多，就 意味着CAS的失败概率更高，从而进入恶性CAS空自旋状态。

- LongAdder的基本思路是分散热点，将value值分散到一个数组 中，不同线程会命中到数组的不同槽(元素)中，各个线程只对自己 槽中的那个值进行CAS操作。这样热点就被分散了，冲突的概率就小很 多。

使用LongAdder，即使线程数再多也不必担心，各个线程会分配到 多个元素上去更新，增加元素个数，就可以降低value的“热度”， AtomicLong中的恶性CAS空自旋就解决了

如果要获得完整的LongAdder存储的值，只要将各个槽中的变量值 累加，返回最终累加之后的值即可。

LongAdder的实现思路与ConcurrentHashMap中分段锁的基本原理 非常相似，本质上都是不同的线程在不同的单元上进行操作，这样减 少了线程竞争，提高了并发效率。

LongAdder的设计体现了空间换时间的思想，不过在实际高并发场 景下，数组元素所消耗的空间可以忽略不计。

LongAdder继承于Striped64类，base值和cells数组都在 Striped64类中定义。基类Striped64内部三个重要的成员如下:

```java
/**
* 成员一:存放Cell的哈希表，大小为2的幂
*/
transient volatile Cell[] cells;
/**
* 成员二:基础值
* 1. 在没有竞争时会更新这个值
* 2. 在cells初始化时，cells不可用，也会尝试通过CAS操作值累加到base */
transient volatile long base;
/**
* 自旋锁，通过CAS操作加锁，为0表示cells数组没有处于创建、扩容阶段
* 为1表示正在创建或者扩展cells数组，不能进行新Cell元素的设置操作 */
transient volatile int cellsBusy;
```

Striped64内部包含一个base和一个Cell[]类型的cells数组， cells数组又叫哈希表。在没有竞争的情况下，要累加的数通过CAS累 加到base上;如果有竞争的话，会将要累加的数累加到cells数组中的 某个Cell元素里面。所以Striped64的整体值value为 base+∑[0~n)cells。

```java
public long longValue() {
  return sum();
}
// The returned value is NOT an atomic snapshot
// invocation in the absence of concurrent updates returns an accurate result(没有并发下会返回准确值)
// but concurrent updates that occur while the sum is being calculated might not be incorporated(纳入).
public long sum() {
  Cell[] as = cells; Cell a;
  long sum = base;
  if (as != null) {
    for (int i = 0; i < as.length; ++i) {
      if ((a = as[i]) != null)
        sum += a.value;
    }
  }
  return sum;
}
```

## 设计思路

Striped64的设计核心思路是通过内部的分散计算来避免竞争，以 空间换时间。LongAdder的base类似于AtomicInteger里面的value，在 没有竞争的情况下，cells数组为null，这时只使用base进行累加;而 一旦发生竞争，cells数组就上场了。

cells数组第一次初始化长度为2，以后每次扩容都变为原来的两 倍，一直到cells数组的长度大于等于当前服务器CPU的核数。为什么 呢?同一时刻能持有CPU时间片去并发操作同一个内存地址的最大线程 数最多也就是CPU的核数。

在存在线程争用的时候，每个线程被映射到 cells[threadLocalRandomProbe & cells.length]位置的Cell元素， 该线程对value所做的累加操作就执行在对应的Cell元素的值上，最终 相当于将线程绑定到cells中的某个Cell对象上。

## add

```java
    public void add(long x) {
        Cell[] as; long b, v; int m; Cell a;
      	//进入if的情况：cells数组不为空   cas失败
        if ((as = cells) != null || !casBase(b = base, b + x)) {
            boolean uncontended = true;
            if (as == null || (m = as.length - 1) < 0 ||
                (a = as[getProbe() & m]) == null ||
                !(uncontended = a.cas(v = a.value, v + x)))
              //调用longAccumulate的情况:
              //	1. 【as == null || (m = as.length - 1) < 0】 :  cells没有初始化
              //  2.  cells初始化了，但是哈希到的这个cell没有初始化过
              //  3.  拿到了一个具体的cell,但是cas失败了,此时uncontended=false
                longAccumulate(x, null, uncontended);
        }
    }

    final boolean casBase(long cmp, long val) {
        return UNSAFE.compareAndSwapLong(this, BASE, cmp, val);
    }
```



## longAccumulate

```java
/**
* 自旋锁，通过CAS操作加锁，为0表示cells数组没有处于创建、扩容阶段
* 为1表示正在创建或者扩展cells数组，不能进行新Cell元素的设置操作 */
		transient volatile int cellsBusy;

/*
casCellsBusy()方法相当于锁的功能:当线程需要cells数组初始化或扩容时，需要调用casCellsBusy()方法
通过CAS方式将 cellsBusy成员的值改为1
	如果修改失败，就表示其他的线程正在进行数组初始化或扩容的操作。
	只有CAS操作成功，cellsBusy成员的值被改为1，当前线程才能执行cells数组初始化或扩容的操作。
在cells 数组初始化或扩容的操作执行完成之后，cellsBusy成员的值被改为0，这时不需要进行CAS修改，直接修改即可，因为不存在争用。

当cellsBusy成员值为1时，表示cells数组正在被某个线程执行初始化或扩容操作，其他线程不能进行以下操作:
(1)对cells数组执行初始化。
(2)对cells数组执行扩容。
(3)如果cells数组中某个元素为null，就为该元素创建新的 Cell对象。因为数组的结构正在修改，所以其他线程不能创建新的 Cell对象。
*/
		final boolean casCellsBusy() {
        return UNSAFE.compareAndSwapInt(this, CELLSBUSY, 0, 1);
    }
```



```java
final void longAccumulate(long x, LongBinaryOperator fn,
                              boolean wasUncontended) {
        int h;
        if ((h = getProbe()) == 0) {
            ThreadLocalRandom.current(); // force initialization
            h = getProbe();
            wasUncontended = true;
        }
  			//扩容意向，collide=true可以扩容，collide=false不可扩容
        boolean collide = false;                // True if last slot nonempty
  			//自旋，一直到设值成功
        for (;;) {
          //as 表示cells引用
          //a  表示当前线程命中的Cell
          //n  表示cells数组长度
          //v. 表示期望值
            Cell[] as; Cell a; int n; long v;
            if ((as = cells) != null && (n = as.length) > 0) {
              //进入到这里，表示cells初始化过了，当前线程要将数据写入到一个cell中
                if ((a = as[(n - 1) & h]) == null) {
                    if (cellsBusy == 0) {       // Try to attach new Cell
                        Cell r = new Cell(x);   // Optimistically create
                        if (cellsBusy == 0 && casCellsBusy()) {
                            boolean created = false;
                            try {               // Recheck under lock
                                Cell[] rs; int m, j;
                                if ((rs = cells) != null &&
                                    (m = rs.length) > 0 &&
                                    rs[j = (m - 1) & h] == null) {
                                  //乐观思想，创建成功，此时可以break
                                    rs[j] = r;
                                    created = true;
                                }
                            } finally {
                                cellsBusy = 0;
                            }
                            if (created)
                                break;
                            continue;           // Slot is now non-empty(Slot指的应该是具体的cell?)
                        }
                    }
                    collide = false;
                }
              
              /*
           wasUncontended是add(...)方法传递进来的参数如果为 false，
           就表示cells已经被初始化，并且线程对应位置的Cell元素也已经被初始化,但是当前线程对Cell元素的竞争修改失败。
           如果wasUncontended为false，就需要重新计算prob的值，那么自旋操作进入下一轮循环
              */
                else if (!wasUncontended)       // CAS already known to fail
                    wasUncontended = true;      // Continue after rehash
       
              /*
              无论执行第一个if分支的哪个子条件，都会在末尾执行h=advanceProb()语句rehash出一个新哈希值
              然后命中新的Cell，如果新 命中的Cell不为空，在此分支进行CAS更新
              将Cell的值更新为 a.value+x，如果更新成功，就跳出自旋操作;否则还得继续自旋
              */
                else if (a.cas(v = a.value, ((fn == null) ? v + x :
                                             fn.applyAsLong(v, x))))
                    break;
              
              /*
              调整cells数组的扩容意向，然后进入下一轮循环。
              如果n≥NCPU条件成立，就表示cells数组大小已经大于等于CPU核数，扩容意向改为false，表示不扩容了;
              如果该条件不成立，就说明cells数组 还可以扩容，尽管如此
              如果cells != as为true，就表示其他线程已经扩容过了，也会将扩容意向改为false，表示当前循环不扩容了。
              当前线程跳到当前if分支的末尾执行rehash操作重新计算prob的值，然后进入下一轮循环。
              */
                else if (n >= NCPU || cells != as)
                    collide = false;            // At max size or stale
              //设置扩容意向为true，但是不一定真的发生扩容
                else if (!collide)
                    collide = true;
              	
              /*
              执行真正扩容的逻辑。
              其条件一cellsBusy==0为true表示当前cellsBusy的值为0(无锁状态)，当前线程可以去竞争这把锁;
              其条件二casCellsBusy()表示当前线程获取锁成功，CAS操作cellsBusy改为 0成功，可以执行扩容逻辑。
              */
                else if (cellsBusy == 0 && casCellsBusy()) {
                    try {
                        if (cells == as) {      // Expand table unless stale
                            Cell[] rs = new Cell[n << 1]; //double
                            for (int i = 0; i < n; ++i)
                                rs[i] = as[i];
                            cells = rs;
                        }
                    } finally {
                        cellsBusy = 0;	//释放锁，不用cas，因为只有一个线程能走到这
                    }
                    collide = false;
                    continue;                   // Retry with expanded table
                }
              	//第一个if分支，最后执行这个，能到达这里的话
                h = advanceProbe(h);	//rehash当前线程的Hash值
            }
          
            else if (cellsBusy == 0 && cells == as && casCellsBusy()) {
              // cells还未初始化(as为null)，本分支计划初始化cells， 
              // 在此之前开始执行cellsBusy加锁，并且要求cellsBusy加锁成功
                boolean init = false;
                try {                           // Initialize table
                    if (cells == as) {
                        Cell[] rs = new Cell[2];
                        rs[h & 1] = new Cell(x);
                        cells = rs;
                        init = true;
                    }
                } finally {
                    cellsBusy = 0;
                }
                if (init)
                    break;
            }
          //如果cellsBusy加锁失败，表示其他线程正在初始化 cells，所以当前线程将值累加到base上。
            else if (casBase(v = base, ((fn == null) ? v + x :
                                        fn.applyAsLong(v, x))))
                break;                          // Fall back on using base
        }
    }
```



## CAS的弊端

CAS操作的弊端主要有以下三点:

### ABA问题

使用CAS操作内存数据时，数据发生过变化也能更新成功，如操作 序列A==>B==>A时，最后一个CAS的预期数据A实际已经发生过更改，但 也能更新成功，这就产生了ABA问题。

ABA问题的解决思路是使用版本号。在变量前面追加上版本号，每 次变量更新的时候将版本号加1，那么操作序列A==>B==>A就会变成 A1==>B2==>A3，如果将A1当作A3的预期数据，就会操作失败。

JDK提供了两个类AtomicStampedReference和 AtomicMarkableReference来解决ABA问题。比较常用的是 AtomicStampedReference类，该类的compareAndSet()方法的作用是首 先检查当前引用是否等于预期引用，以及当前印戳是否等于预期印 戳，如果全部相等，就以原子方式将引用和印戳的值一同设置为新的值。



### 只能保证一个共享变量之间的原子性操作

当对一个共享变量执行操作时，我们可以使用循环CAS的方式来保 证原子操作，但是对多个共享变量操作时，CAS就无法保证操作的原子 性。

 一个比较简单的规避方法为:把多个共享变量合并成一个共享变量来操作。

JDK提供了AtomicReference类来保证引用对象之间的原子性，可以把多个变量放在一个AtomicReference实例后再进行CAS操作。比如 有两个共享变量i=1、j=2，可以将二者合并成一个对象，然后用CAS 来操作该合并对象的AtomicReference引用。



### 开销问题

自旋CAS如果长时间不成功(不成功就一直循环执行，直到成 功)，就会给CPU带来非常大的执行开销。

解决CAS恶性空自旋的有效方式之一是以空间换时间，较为常见的 方案为:

(1)分散操作热点，使用LongAdder替代基础原子类 AtomicLong，LongAdder将单个CAS热点(value值)分散到一个cells 数组中。

(2)使用队列削峰，将发生CAS争用的线程加入一个队列中排 队，降低CAS争用的激烈程度。JUC中非常重要的基础类AQS(抽象队列同步器)就是这么做的