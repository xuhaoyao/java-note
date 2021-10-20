- [AQS](#aqs)
  - [可重入锁](#可重入锁)
  - [LockSupport](#locksupport)
    - [三种让线程等待和唤醒的方式](#三种让线程等待和唤醒的方式)
      - [Object的wait和notify](#object的wait和notify)
    - [Condition的await和signal](#condition的await和signal)
    - [LockSupport的park(等待)和unpark(唤醒)](#locksupport的park等待和unpark唤醒)
      - [什么是LockSupport?](#什么是locksupport)
    - [阻塞方法](#阻塞方法)
    - [唤醒方法](#唤醒方法)
    - [例子演示](#例子演示)
    - [原理说明](#原理说明)
    - [面试题](#面试题)
      - [为什么可以先唤醒线程后阻塞线程？](#为什么可以先唤醒线程后阻塞线程)
      - [为什么唤醒两次后阻塞两次，但最终结果还会阻塞线程？](#为什么唤醒两次后阻塞两次但最终结果还会阻塞线程)
  - [AQS](#aqs-1)
    - [AQS为什么是JUC内容中最重要的基石](#aqs为什么是juc内容中最重要的基石)
    - [锁和同步器的关系](#锁和同步器的关系)
    - [AQS能做什么?](#aqs能做什么)
    - [AQS初步](#aqs初步)
      - [AQS同步队列的基本结构](#aqs同步队列的基本结构)
      - [AQS内部体系架构](#aqs内部体系架构)
    - [ReentrantLock解读AQS源码[非公平锁为例]](#reentrantlock解读aqs源码非公平锁为例)
      - [从最简单的lock方法看看公平和非公平](#从最简单的lock方法看看公平和非公平)
      - [lock()](#lock)
      - [acquire()](#acquire)
      - [不公平锁与公平锁](#不公平锁与公平锁)
      - [addWaiter()细节 ---CAS--- 保证了LCH队列的线程安全](#addwaiter细节----cas----保证了lch队列的线程安全)
      - [acquireQueued()](#acquirequeued)
      - [unlock()](#unlock)

# AQS

## 可重入锁

![image-20211020092515440](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20211020092515440.png)

Synchronized的重入的实现机制

![image-20211020093426926](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20211020093426926.png)



## LockSupport

LockSupport是用来创建锁和其他同步类的基本线程阻塞原语。

LockSupport中的park()和unpark()的作用分别是阻塞线程和解除阻塞线程，可以理解为是wait()和notify()的加强版



### 三种让线程等待和唤醒的方式

![image-20211020094909231](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20211020094909231.png)



#### Object的wait和notify

- 必须在synchronized同步代码块中调用，否则会抛出`IllegalMonitorStateException`
- 将notify放在wait方法前面，程序将无法执行，因为线程没法被唤醒

**wait和notify方法必须在同步块中或同步方法里成对出现使用,做到顺序是先wait后notify**



### Condition的await和signal

- 必须在lock()和unLock()之间调用,否则会抛出`IllegalMonitorStateException`
- signal放在await前面，程序将无法执行，因为线程没法被唤醒



### LockSupport的park(等待)和unpark(唤醒)

#### 什么是LockSupport?

- 通过park()和unpark(thread)方法来实现阻塞和唤醒线程的操作
-  LockSupport是一个线程阻塞工具类,所有的方法都是静态方法,可以让线程在任意位置阻塞,阻塞之后也有对应的唤醒方法。归根结底,LockSupport调用的Unsafe中的native代码
- 官网解释:
  - LockSupport是用来创建锁和其他同步类的基本线程阻塞原语
  - LockSupport类使用了一种名为Permit(许可)的概念来做到阻塞和唤醒线程的功能,每个线程都有一个许可(permit),permit只有两个值1和零,默认是零
  - 可以把许可看成是一种(0,1)信号量(Semaphore),但与Semaphore不同的是,许可的累加上限1



### 阻塞方法

```java
public static void park() {
    UNSAFE.park(false, 0L);
}
```

- permit默认是0,所以一开始调用park()方法,当前线程就会阻塞,直到别的线程将当前线程的permit设置为**1时, park方法会被唤醒**,然后会将permit再次设置为0并返回



### 唤醒方法

```java
public static void unpark(Thread thread) {
    if (thread != null)
        UNSAFE.unpark(thread);
}
```

- 调用unpark(thread)方法后,就会将thread线程的许可permit设置成1(注意多次调用unpark方法,不会累加,permit值还是1)会自动唤醒thread线程,即之前阻塞中的LockSupport.park()方法会立即返回



### 例子演示

```java
//LockSupport支持先唤醒后等待,唤醒之后permit是1,只要是1,park方法就不会阻塞,park执行完毕将permit设置为0
Thread a = new Thread(() -> {
    try { TimeUnit.SECONDS.sleep(1); } catch (InterruptedException e) { e.printStackTrace(); }
    System.out.println(Thread.currentThread().getName() + "\t---->come in");
    LockSupport.park();   //被阻塞...等待通知,它要通过需要permit
    System.out.println(Thread.currentThread().getName() + "\t---->被唤醒");
}, "A");
a.start();

Thread b = new Thread(() -> {
    System.out.println(Thread.currentThread().getName() + "\t--->唤醒a" );
    LockSupport.unpark(a);
}, "B");
b.start();
```



### 原理说明

![image-20211020102848214](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20211020102848214.png)

![image-20211020102812952](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20211020102812952.png)



### 面试题

#### 为什么可以先唤醒线程后阻塞线程？

因为unpark使得线程获得了一个permit,之后这个线程调用park的时候,由于有了permit,所以不会阻塞，并把这个permit消费掉，设置为0

#### 为什么唤醒两次后阻塞两次，但最终结果还会阻塞线程？

因为凭证数量最多为1，连续调用两次unpark和调用一次unpark效果一样，只会增加一个permit,而调用两次park却需要两个permit,因为permit不够，所以会阻塞线程

也要看情况，最终结果也可能不会阻塞线程,例如下面的代码

只要执行顺序是unpark->park->unpark->park，线程就不会阻塞，这里通过Thread.yield()显式地调度线程，使得唤醒两次和阻塞两次交替执行，就不会阻塞，但如果是unpark->unpark->park->park的话，就会阻塞线程！

```java
//LockSupport支持先唤醒后等待,唤醒之后permit是1,只要是1,park方法就不会阻塞,park执行完毕将permit设置为0
Thread a = new Thread(() -> {
    System.out.println(Thread.currentThread().getName() + "\t---->come in");
    LockSupport.park();   //被阻塞...等待通知,它要通过需要permit
    System.out.println(Thread.currentThread().getName() + "\t---->被唤醒");
    Thread.yield();
    LockSupport.park();
    System.out.println(Thread.currentThread().getName() + "\t---->被唤醒");
}, "A");
a.start();

try { TimeUnit.SECONDS.sleep(1); } catch (InterruptedException e) { e.printStackTrace(); }

Thread b = new Thread(() -> {
    System.out.println(Thread.currentThread().getName() + "\t--->唤醒a" );
    LockSupport.unpark(a);
    Thread.yield();
    System.out.println(Thread.currentThread().getName() + "\t--->再次唤醒a" );
    LockSupport.unpark(a);
}, "B");
b.start();
```



## AQS

即java.util.concurrent.locks.AbstractQueuedSynchronizer,抽象队列同步器

**是用来构建锁或者其他同步器组件的重量级基础框架及整个JUC体系的基石，通过内置的FIFO队列来完成资源获取线程的排队工作，并通过一个int类型变量表示持有锁的状态**

![image-20211020104925895](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20211020104925895.png)

### AQS为什么是JUC内容中最重要的基石

![image-20211020105530977](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20211020105530977.png)

![image-20211020105515937](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20211020105515937.png)



### 锁和同步器的关系

- 锁，面向锁的使用者，定义了程序员与锁交互的api，调用即可
- 同步器，面向锁的实现者，java并发大神Doug Lea，提出统一规范并简化了锁的实现，屏蔽了同步状态管理，阻塞线程排队和通知，唤醒机制等。



### AQS能做什么?

**加锁会导致阻塞**

- 有阻塞就需要排队，实现排队必然需要有某种形式的队列来进行管理



抢到资源的线程直接使用，进行业务逻辑，抢不到资源的必然涉及一种排队等候机制，抢占资源失败的线程就去等待，但等候线程仍然保留获取锁的可能，且获取锁的流程仍在继续

 

如果共享资源被占用，就需要一定的阻塞等待唤醒机制来保证锁分配。这个机制主要用的是CLH队列的变体实现的，将暂时获取不到锁的线程加入到队列中，这个队列就是AQS的抽象表现。它将**请求共享资源的线程封装成队列的结点(Node),通过CAS、自旋以及LockSupport.park()的方式**，维护state变量的状态，使并发达到同步的控制效果

![image-20211020111219259](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20211020111219259.png)



### AQS初步

AQS使用一个volatile的int类型的成员变量来表示同步状态，通过内置的FIFO队列来完成资源获取的排队工作，将每条要去抢占资源的线程封装成一个Node节点来实现锁的分配，通过CAS完成对State值得修改。 

`AQS = state + CLH队列`

`Node = waitStatus + 前后指针指向`

```java
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {
    
    private transient volatile Node head;
    
    private transient volatile Node tail;
    
    private volatile int state;
    
    static final class Node {
        //共享
		static final Node SHARED = new Node();
        
        //独占
        static final Node EXCLUSIVE = null;
        
		//线程被取消了
        static final int CANCELLED =  1;
        
        //表示线程需要唤醒
        static final int SIGNAL    = -1;
      
        //等待condition唤醒
        static final int CONDITION = -2;
        
        //共享式同步状态获取 将会无条件地传播下去
        static final int PROPAGATE = -3;
        
        //初始为0，状态是上面的四种
        volatile int waitStatus;
        
        volatile Node prev;
		volatile Node next;
        volatile Thread thread;
    }
    
}
```

![20210702094553986](C:\Users\Administrator\Desktop\图片\20210702094553986.png)



#### AQS同步队列的基本结构

![image-20211020123140804](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20211020123140804.png)



#### AQS内部体系架构

![image-20211020121726386](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20211020121726386.png)



### ReentrantLock解读AQS源码[非公平锁为例]

Lock接口的实现类，基本都是通过【聚合】了一个【队列同步器】的子类完成线程访问控制的。

![image-20211020123630070](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20211020123630070.png)



#### 从最简单的lock方法看看公平和非公平

![image-20211020124321085](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20211020124321085.png)



对比公平锁和非公平锁的tryAcquire()方法的实现代码，其实差别就在于非公平锁获取锁时比公平锁少了一个判断`!hasQueuedProcecessors()`

`hasQueuedProcecessors()`判断了是否需要排队，导致公平锁和非公平锁的差异如下：

- 公平锁：讲究先来先到，线程在获取锁时，如果这个锁的等待队列中已经有线程在等待，那么当前线程就会进入等待队列中。
- 非公平锁：不管是否有等待队列，如果可以获取锁，则立刻占有锁对象。也就是说队列的第一个排队线程在unpark()之后还是需要竞争锁（存在线程竞争的情况下）
- ![image-20211020125612923](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20211020125612923.png)



#### lock()

```java
final void lock() {
    if (compareAndSetState(0, 1))  
        setExclusiveOwnerThread(Thread.currentThread()); //第一个线程执行[AbstractOwnableSynchronizer]
    else
        acquire(1);	 //第二个及后续线程执行
}
```



#### acquire()

```java
public final void acquire(int arg) {
    //此方法是线程没有抢到锁才调用的
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

```java
protected final boolean tryAcquire(int acquires) {
    //线程没有抢到锁，但是还是先尝试一下，看看现在锁的状态，因为有两个极端情况
    return nonfairTryAcquire(acquires);
}
```

```java
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        //极端情况一：线程没有抢到锁，执行这个方法的时候，锁被释放了，这时候去抢锁
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    //极端情况二：可重入锁，线程没有抢到锁，但是发现，这个锁就是自己的。
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```



####  不公平锁与公平锁

- 不公平锁的`nonfairTryAcquire`方法,当一个线程请求锁时，只要`compareAndSetState`成功就获取到了锁，在这个前提下，刚释放锁的线程再次获取同步状态的几率会非常大，使得其他线程只能在同步队列中等待。

- 由于非公平锁线程切换次数小【因为`nonfairTryAcquire`方法不公平的抢锁】，因此开销更小，而公平锁只能在锁没有被占有的情况下，多线程可以抢，一旦有线程抢到了锁，之后来的线程就乖乖进队列，而不能再试探性地获取锁，这就是线程切换差异的原因。

- 见并发编程的艺术P140

- 公平锁的`tryAcquire`方法,多了一个`hasQueuedPredecessors`

- ```java
  //此方法返回false,当前线程才能拿到锁
  public final boolean hasQueuedPredecessors() {
      // The correctness of this depends on head being initialized
      // before tail and on head.next being accurate if the current
      // thread is first in queue.
      Node t = tail; // Read fields in reverse initialization order
      Node h = head;
      Node s;
      return h != t &&	//若h==t,说明队列为空,直接拿锁
          //此处逻辑有点拗口,就是说,一定要保证,当前线程是傀儡节点的下一个节点,以此来实现公平
          ((s = h.next) == null || s.thread != Thread.currentThread());
  }
  ```

- 对于所有进入到队列中的线程，均是FIFO，不公平锁的不公平之处在于一开始抢锁的时候不公平，而在队列中的线程，可以说是公平的，排在队列前面的线程一定先执行【两种锁均是如此】



#### addWaiter()

但线程调用`tryAcquire`试图抢锁失败的时候，调用`addWaiter`

```java
public final void acquire(int arg) {
    //此方法是线程没有抢到锁才调用的
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

```java
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);  //将当前线程封装在CLH队列的Node节点中
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    if (pred != null) {		//但队列的节点个数>=2时,if判断会成功
        node.prev = pred;	//当前线程的节点指向之前的尾部节点
        if (compareAndSetTail(pred, node)) { //设置当前线程的node节点为队列尾部
            pred.next = node;	//之前的尾部节点指向新的尾部节点
            return node;
        }
    }
    enq(node);	//队列中没有节点时，执行此方法,返回的是老的尾部节点
    return node;
}
```

```java
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))	//CLH队列的第一个节点是哨兵节点
                tail = head;
        } else {
            node.prev = t;	//node节点指向之前的尾部节点
            if (compareAndSetTail(t, node)) {	//设置当前线程的node节点为队列尾部
                t.next = node;	//之前的尾部节点指向新的尾部节点
                return t;	//返回老的尾部节点
            }
        }
    }
}
```



#### addWaiter()细节 ---CAS--- 保证了LCH队列的线程安全

- 整个方法没有任何的同步，试想一下如果大量线程同时lock,内部的LCH队列会不会出现线程安全问题？
- `Try the fast path of enq; backup to full enq on failure`,尝试快速入队，失败之后才调用enq方法

```java
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        //其实这句注释就很好的解释了全部问题
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {  //cas保证了队尾正确赋值,若赋值失败,调用enq方法
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }
```

```java
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {  //每次节点的入队,都是通过cas保证正确赋值,若赋值失败就重来
                t.next = node;
                return t;
            }
        }
    }
}
```





双向链表中，**第一个节点为虚节点（也叫哨兵节点）**，其实并不存储任何信息，只是占位，真正的有数据的节点是从第二个节点开始的

![image-20211020174954579](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20211020174954579.png)



#### acquireQueued()

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();  //取node的前驱节点
            /**
             * 继续尝试tryAcquire获取锁,好像只有第二个节点有这个待遇？是的没错！因为第三个节点没必要获取锁了，  			              *	因为第三个节点前面有第二个节点曾经尝试过获取锁了，它已尝试失败进入阻塞，第三个节点就没必要tryAcquire 
             *	后面来的节点均是先将waitStatus=SINGAL，然后park阻塞
             */
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&	//看下面的注释【重要】
                parkAndCheckInterrupt())	//线程将会在这里调用LockSupport.park阻塞住,等待唤醒
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

```java
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        //如果是SIGNAL,即当前线程应该等待被占用的锁释放,直接返回true,去调用parkAndCheckInterrupt阻塞当前线程
        if (ws == Node.SIGNAL)	
            /*
             * 由下面的A注释可以看出,自旋一次后,当前线程还是没拿到锁，因此返回true,使这个线程受阻塞
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true;
        if (ws > 0) {
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
             * ========================A==========================
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);		//看英文注释
        }
        return false;
    }

```

```java
private final boolean parkAndCheckInterrupt() {
    //线程挂起，程序不会继续向下执行
    LockSupport.park(this);
    /**
       两种情况会继续向下执行
       	1.被unpark,此时该线程一定排在队列首部(傀儡节点除外)
       	2.被中断(interrupt)
       	Thread.interrupted();
       		由于两种情况之一的发生，调用此方法返回当前线程的中断状态，并清空中断状态
        若是由于被中断,那么该方法会返回true
     */
    return Thread.interrupted();
}
```



![image-20211020183112038](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20211020183112038.png)



#### unlock()

```java
public void unlock() {
    sync.release(1);
}

public final boolean release(int arg) {
    if (tryRelease(arg)) { //state为0就解锁,由于可重入的关系,因此这个方法可能返回false
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);  //尝试唤醒傀儡节点.next
        return true;
    }
    return false;
}
```

```java
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {  //可重入锁的体现
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

```java
private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
    int ws = node.waitStatus;	//由上面分析知道,傀儡节点的waitStatus=SINGAL=-1
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);  //唤醒傀儡节点.next
}
```



紧接着线程B被唤醒，B获取到锁，B成为新的傀儡节点，老的傀儡节点出队(GC回收)

```java
private void setHead(Node node) {
    head = node;
    node.thread = null;
    node.prev = null;
}
final Node p = node.predecessor();
if (p == head && tryAcquire(arg)) {
    setHead(node);
    p.next = null; // help GC
    failed = false;
    return interrupted;
}
```

![image-20211020220738223](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20211020220738223.png)



![image-20211020222334434](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20211020222334434.png)



![image-20211020222607242](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20211020222607242.png)



![image-20211020222802999](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20211020222802999.png)



