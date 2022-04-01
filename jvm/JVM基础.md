# JVM面试题

## Java运行时数据区

Java运行时数据区主要有堆、方法区、程序计数器、虚拟机栈、本地方法栈。

- 线程共享：堆和方法区。
- 线程私有：程序计数器、虚拟机栈、本地方法栈。（与线程生命周期相同）

**程序计数器：**

1、当前线程的行号指示器，字节码解释器通过程序计数器来选取下一条需要执行的字节码指令，程序的分支，循环，异常处理等就是通过它来完成

2、线程切换后，通过程序计数器恢复到原来执行的代码位置继续执行，使得线程好像从来没有切换过一样。

3、对于Java方法，记录的是虚拟机字节码指令的地址，对于本地（Native）方法，计数器则为空（Undefined）

4、不会有OutOfMemoryError

**Java虚拟机栈：**

1、每个方法执行的时候，都会同步创建一个栈帧，用于存储局部变量表，操作数栈，动态连接、方法出口等信息。

2、方法调用，压入一个栈帧，方法结束或者异常了，就会弹出一个栈帧。

3、若线程请求的栈深度大于虚拟机所允许的深度，StackOverflowError

4、HotSpot虚拟机的栈容量不允许动态扩展，不会由于虚拟机栈无法扩展而导致OOM，但是在一个方法里不断的new线程，**最后也会导致OOM（创建线程申请内存时无法获得足够内存而OOM）**

**本地方法栈：**

HotSpot虚拟机将虚拟机栈和本地方法栈合并在一起了

**Java堆：**

唯一的目的就是存放对象实例。

由于即时编译技术的进步，尤其是逃逸分析，**栈上分配、标量替换优化手段**导致了Java对象实例都分配在堆上也不是那么绝对了。

> **逃逸分析**：一个对象在方法里被定义后，它可能被外部方法引用（return返回这个对象或者作用调用参数传给其他方法），这就叫方法逃逸；若被外部线程访问到，如赋值给可以在其他线程中访问的实例变量，叫做线程逃逸。
>
> 不逃逸->方法逃逸->线程逃逸（逃逸程度由低到高）
>
> **栈上分配：**支持方法逃逸，不支持线程逃逸，直接将一个对象在栈上分配内存，那么这个对象随着方法的结束而随栈帧出栈而销毁。
>
> **标量替换：**若一个对象不逃逸，程序真正执行的时候不去创建这个对象，而是创建它的若干实例字段即可
>
> - 比如Node(intx,int y)，不去创建，直接创建int x和int y

**方法区：**

存储已经被虚拟机加载的类型信息、常量、静态变量、即时编译器编译后的代码缓存等数据。

> 永久代或者元空间，都是对方法区的一种实现而已。
>
> - JDK8以前，方法区的实现叫永久代。
>
> - JDK8开始，方法区的实现为元空间（本地内存实现元空间）。
>
> - JDK7开始，静态变量、字符串常量池放到了堆中
>   - 运行时常量池中的字符串常量池被单独拿到了堆中，其他还在方法区
>   - 运行时常量池是方法区的一部分，Class 文件中除了有类的版本、字段、方法、接口等描述信息外，还有常量池表（用于存放编译期生成的各种字面量和符号引用），这部分会在类加载后存放到方法区的运行时常量池中。



### 程序计数器的值可以为空吗？

- 执行Java方法的时候，值是字节码解释器指令的地址，执行Native方法的时候，为空（Undefined）



### 堆中对象怎么细分的？

根据分代收集理念，堆中可以分为新生代和老年代，**新生代又可以分为Eden区，Survivor from和Survivor to区，比例为8：1：1**



### 哪些区域会OOM

除了程序计数器，其他地方都会OOM

- 通过参数`-XX:+HeapDumpOnOutOfMemoryError`可以让虚拟机在出现内存溢出异常的时候Dump出当前内存堆转储快照进行分析。
- jmap -dump:format=b,file=xxx.bin pid ，可以生成一个堆转储快照，用jhat xxx.bin 来分析生成的jmap堆转储快照。
  - jmap：Java内存映像工具
  - jhat：虚拟机堆转储快照分析工具。
- ![image-20220316113351139](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220316113351139.png)

#### 堆溢出

```java
//-Xms10m -Xmx10m -XX:+HeapDumpOnOutOfMemoryError
public class HeapOOM {
    static class OOMObject{

    }
    public static void main(String[] args) {
        List<OOMObject> list = new ArrayList<>();
        while (true){
            list.add(new OOMObject());
        }
    }
}
//Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
```

堆溢出怎么办？

- 首先通过内存映像工具（jhat等）对Dump出来的堆转储快照进行分析
- 第一步首先应确认内存中导致OOM的对象是否有必要，要看是否是内存泄漏还是内存溢出
- 如果是内存泄漏，通过工具查看泄漏对象的GC Roots的引用链，找到泄漏对象是通过怎么样的引用路径、与哪些GC Roots引用，使得GC无法回收它们
- 如果是内存溢出，就应该查看虚拟机的堆参数（-Xmx与-Xms）设置，与机器内存相比，看看是否还有向上调整的空间



#### 虚拟机栈和本地方法栈的溢出

- 如果线程请求的栈深度大于虚拟机所允许的最大深度，将抛出StackOverflowError异常。
- 如果虚拟机的栈内存允许动态扩展，当扩展栈容量无法申请到足够的内存时，将抛出OutOfMemoryError
  - HotSpot虚拟机不支持动态扩展，但是创建线程申请内存而无法获得足够内存的时候，也会OOM

```java
//-Xss2M:每个线程堆栈大小为2M
//32位下系统运行
public class JavaVMStackOOM {
    
    private void dontStop(){
        while (true){
            
        }
    }
    
    public void stackLeakByThread(){
        while (true){
            Thread thread = new Thread(this::dontStop);
            thread.start();
        }
    }

    public static void main(String[] args) {
        JavaVMStackOOM oom = new JavaVMStackOOM();
        oom.stackLeakByThread();
    }
    
}
//java.lang.OutOfMemoryError: unable to create native thread
```



#### 方法区和运行时常量池溢出

JDK7开始，字符串常量池以及静态变量这些搬到了Java堆，方法区OOM比较少见了



#### 本机直接内存溢出

直接内存的容量大小可通过-XX:MaxDirectMemorySize参数来指定，如果不指定，则默认与Java堆最大值（-Xmx指定）一致。

```java
//-Xmx20m -XX:MaxDirectMemorySize=10M
public class DirectMemoryOOM {

    private static final int _1MB = 1024 * 1024;

    public static void main(String[] args) throws IllegalAccessException {
        Field unsafeField = Unsafe.class.getDeclaredFields()[0];
        unsafeField.setAccessible(true);
        Unsafe unsafe = (Unsafe) unsafeField.get(null);
        while (true){
            unsafe.allocateMemory(_1MB);
        }
    }
}
//Exception in thread "main" java.lang.OutOfMemoryError
```

**直接内存溢出有个明显特征就是Heap Dump文件不会有什么明显的异常情况，且Dump文件很小**





## Java对象的创建过程

**1、类加载检查**

当Java虚拟机遇到一条new指令时，首先去检查这个指令的参数能否在常量池中定位到一个类的符号引用，并且检查这个符号引用代表的类是否已经被加载、解析、初始化过，如果没有，那就必须先进行类加载的过程。

> 在编译的时候一个每个java类都会被编译成一个class文件，但在编译的时候虚拟机并不知道所引用类的地址，多以就用符号引用来代替，而在这个解析阶段就是为了把这个符号引用转化成为真正的地址的阶段。

[(19条消息) 走进java_符号引用与直接引用_麻-雀的博客-CSDN博客_符号引用](https://blog.csdn.net/qq_34402394/article/details/72793119?spm=1001.2101.3001.6650.2&utm_medium=distribute.pc_relevant.none-task-blog-2~default~CTRLIST~Rate-2.pc_relevant_default&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2~default~CTRLIST~Rate-2.pc_relevant_default&utm_relevant_index=5)

**2、分配内存**

类加载检查过后，就为这个对象分配内存。分配方式有**指针碰撞和空闲列表两种。**

- 选择哪种分配方式，要看堆内存是否规整，是否规整又要看垃圾收集器具体的垃圾收集算法

**3、初始化零值**

内存分配完毕之后，虚拟机需要将分配到的内存空间（不包括对象头）初始化为零

**4、设置对象头**

比如这个对象是哪个类的实例，如何才能找到类的元数据信息，对象的哈希码、对象的GC分代年龄等。

两部分：一、Mark Word；二、类型指针，指向这个类元数据的指针，通过这个指针可以知道这个对象是哪个类的实例。

**5、执行初始化方法**

就是执行我们自己写的构造函数



### 对象内存的分配方式

**指针碰撞：**

- 若堆中内存是绝对规整的，所有使用过的内存放在一边，没使用过的内存放在另一边，中间放着一个指针作为分界点，那么分配内存只需要算出这个对象占用了多大的内存，指针相应挪动就好了。
- 对于使用标记复制、标记整理的，垃圾收集过后内存规整，就可以使用指针碰撞
  - Serial、ParNew、Parallel Scavenge等

**空闲列表：**

- 堆内存分配是零散的。
- 对于使用标记清除的，就应该使用空闲列表的方式来分配，虚拟机会维护一个列表记录哪些内存块时可用的，在分配的时候从列表中找到一块足够大的空间划分给对象实例，并更新列表上的记录。
  - CMS



#### 如何保证内存分配的线程安全？

1、CAS配上失败重试保证分配的原子性。

2、TLAB（Thread Local Allocation Buffer）：把内存分配的动作按照线程划分在不同的空间之中，即每个线程在Java堆中预先分配一小块内存，称为本地线程分配缓冲TLAB，哪个线程要分配内存，就在哪个线程的TLAB中分配，只有TLAB分配完了，分配新的缓冲区的时候才同步锁定。



## 对象的内存布局

在HotSpot虚拟机中，对象在内存中的布局可以分为三部分：对象头、实例数据和对齐填充。

**对象头：**

- 第一类是用于存储对象自身运行时数据，如哈希码，GC分代年龄，锁状态标志，偏向线程ID，指向锁记录的指针等等（**Mark Word**）,32位虚拟机占用32个比特，64位虚拟机占用64个比特。
- 第二类是类型指针，即对象指向它的类型元数据的指针，通过这个指针来确定该对象是哪个类的实例。

**实例数据：**

对象真正存储的有效信息，就是我们在代码里面所定义的各种类型的字段内容。

**对齐填充:**

占位符的作用，HotSpot虚拟机**要求对象起始地址必须是8字节的整数倍**，如果实例数据部分没有对齐的话，就需要对齐填充来补全。



### Object obj=new Object()占用字节

以64位操作系统为例，new Object()占用大小分为两种情况：

- 未开启压缩指针
  - 8（Mark Word） + 8（类型指针） = 16字节
- 开启压缩指针（默认是开启的）
  - 8（Mark Word） + 4（类型指针） + 4（对其填充）= 16字节



#### 压缩指针的好处

```java
public class MyItem {
    byte i = 0;
}
```

现在看看 new MyItem()占用多少字节？

- 未开启压缩指针
  - 8（Mark Word） + 8（类型指针） + 1（实例数据中的byte）+ 7（对齐填充） = 24字节
- 开启压缩指针（默认是开启的）
  - 8（Mark Word）+ 4（类型指针） + 1（实例数据中的byte）+ 3（对齐填充） = 16字节

**这个时候就能看出来开启了指针压缩的优势了，如果不断创建大量对象，指针压缩对性能还是有一定优化的。**



### java 引用占几个字节

| 类型                  | 64位（无压缩） | 64位（压缩） |
| --------------------- | -------------- | ------------ |
| boolean               | 1              | 1            |
| byte                  | 1              | 1            |
| short                 | 2              | 2            |
| char                  | 2              | 2            |
| int                   | 4              | 4            |
| float                 | 4              | 4            |
| long                  | 8              | 8            |
| double                | 8              | 8            |
| 普通对象头            | 16             | 12           |
| 数组对象头            | 24             | 16           |
| reference（引用类型） | 8              | 4            |



## 对象的访问定位

Java程序通过栈上的reference数据来操作堆上的具体对象，reference定位堆上的对象可以有两种方式。

**使用句柄：**Java在堆上会划分出一块句柄池，reference中存储的就是对象的句柄地址，而句柄中包含了对象实例数据与类型数据各自的具体地址信息。即通过这个句柄地址就能访问对象。

<img src="https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220315124028135.png" alt="image-20220315124028135" style="zoom:67%;" />

**直接指针：**reference中存储的就是对象地址（HotSpot虚拟机采用直接指针）

![image-20220315124046598](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220315124046598.png)

**采用句柄最大的好处就是reference中存储的是稳定句柄地址，对象被移动时（垃圾收集时移动对象很普遍）只会改变句柄中的实例数据指针，而reference本身不需要修改。**

**使用直接指针的话最大的好处就是速度更快，节省了一次指针定位的时间开销。**



## 如何判断一个对象是否存活？

常见的方法有引用计数法（Redis采用），可达性分析算法（Java采用）

**引用计数法：**在对象中添加一个计数器，每当有一个地方引用它，计数器+1，当引用失效，计算器-1，计数器为0的对象就是不再被使用的。`refCount`

```java
/*
这个方法实现简单，效率高，但是目前主流的虚拟机中并没有选择这个算法来管理内存
其最主要的原因是它很难解决对象之间相互循环引用的问题。
*/
public class ReferenceCountingGc {
    Object instance = null;
	public static void main(String[] args) {
		ReferenceCountingGc objA = new ReferenceCountingGc();
		ReferenceCountingGc objB = new ReferenceCountingGc();
		objA.instance = objB;
		objB.instance = objA;
		objA = null;
		objB = null;

	}
}
```

**可达性分析：**通过一系列`GC Roots`的根对象作为起始节点集，从这些节点开始根据引用关系向下搜索，搜索过程所走过的路径称为引用链，如果一个对象到`GC Roots`间没有任何引用链相连，就是`GC Roots不可达这个对象`那么此对象是不再被使用的。

下图中红色部分就是不可达对象，全都可以回收了。

<img src="https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220315125458142.png" alt="image-20220315125458142" style="zoom:67%;" />

**固定可以作为GC Roots的对象包括以下几类：**

- 虚拟机栈的栈帧中，局部变量表引用的对象。（当前运行方法用到的参数，局部变量等）
- 方法区中类静态属性引用的对象。（在一个类中定义的 static Object obj = ...）
- 方法区中常量引用的对象
- 被同步锁持有的对象
- 本地方法栈中引用的对象。



### 不可达的对象一定会被回收吗？

一个对象宣告死亡，最多经历两次标记过程。

一、若这个对象不可达，进行第一次标记

**随后进行筛选，筛选的条件是这个对象有没有必要执行finalize()方法或者这个对象已经执行过了**

二、若需要执行finalize()方法，那么对象可以在这里拯救自己，**在finalize()方法中重新与引用链上的任何一个对象建立关联即可**，如果对象还没有逃脱，那么就要被回收。



## 四种引用

**强引用：**就是在代码之中普遍存在的引用赋值，`Object obj = new Object()`，只要强引用关系存在，对象就不会被回收。

**软引用：**在即将发生OOM的时候，系统会对软引用进行回收，如果回收过后内存还是不足，才OOM。根据这个特性，**软引用可用来实现内存敏感的高速缓存。**

**弱引用：**被弱引用关联的对象，在下一次垃圾回收的时候就会被回收掉。

**虚引用：**最弱的一种引用关系，没有办法通过虚引用来取得一个对象实例，为一个对象设置虚引用关联的唯一目的只是为了能在这个对象被收集器回收时得到一个系统通知。



## 垃圾收集算法

### 分代收集理论--弱、强、跨代

弱分代假说：绝大多数对象都是朝生夕灭的。

强分代假说：熬过越多次垃圾回收过程的对象就越难以消亡。

基于这两个假说，如果一个区域中绝大多数对象都是朝生夕灭的话，我们只要考虑如何保存少量存活的对象而不是去标记大量将要被回收的对象，就能以较低的代价回收到大量的空间，如果剩下的对象都是难以消亡的，那么把它们集中放在一起，虚拟机便可以使用较低的频率来回收这个区域。因此，**根据分代收集理论，应该把Java堆分为新生代和老年代**。新生代中每次都会有大批对象死去，而每次回收后少存活的少量对象，就可以逐步晋升到老年代去。



**跨代引用假说：**跨代引用相对于同代引用来说只占极少数。

什么是跨代引用？

假如现在来一个Minor GC，**新生代中的对象是完全有可能被老年代所引用的**，为了找出新生代存活对象，就不得不在固定的GC Roots之外，额外遍历整个老年代的对象。这种操作肯定不行。

**如何解决跨代引用？**

只需要在新生代上建立一个全局的数据结构（记忆集），这个结构把老年代划分成若干小块，标识出老年代的哪一块内存会存在跨代引用。此后发生Minor GC，只有包含了跨代引用的小块内存里的对象才会被加入到GC Roots进行扫描。



**记忆集是一种抽象，卡表是它的实现。**

字节数组CARD_TABLE的每一个元素都对应着其标识的内存区域中一块特定大小的内存块，这个内存块称为“卡页”。一个卡页的内存中通常包含不止一个对象，只要卡页内有一个（或更多）对象的字段存在着跨代指针，那就将对应卡表的数组的值标志为1，称这个元素变脏了，垃圾收集的时候，筛选出卡表中变脏的元素，就能知道卡页内存块中包含跨代指针，把它们加入GC Roots中一并扫描。



### 标记清除算法

算法分为标记和清除两个部分。首先标记出所有需要回收的对象，在标记完成后，统一回收掉所有被标记的对象。

- 主要缺点：
  - 执行效率不稳定，若Java堆中有大量需要回收的对象，那么标记清除的效率就很低下
  - 内存空间的碎片化，标记清除后会产生大量不连续的内存碎片，下一次分配大一点的对象又会有问题，可能提前触发Full GC

<img src="https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220315142449743.png" alt="image-20220315142449743" style="zoom:67%;" />



### 标记复制算法

将可用内存按容量划分为大小相等的两块，每次只使用其中的一块。当这一块的内存用完了，就将还存活着的对象顺序复制到另外一块上面，然后再把已使用过的内存空间一次清理掉。

优点：这样使得每次都是对整个半区进行内存回收，内存分配时也就不用考虑内存碎片等复杂情况，只要按顺序分配内存即可，实现简单，运行高效。

缺点：只是这种算法的代价是将内存缩小为了原来的一半。

<img src="https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220315142911225.png" alt="image-20220315142911225" style="zoom:67%;" />



### 标记整理算法

首先标记出所有需要回收的对象，在标记完成后，后续步骤不是直接对可回收对象进行清理，而是让所有存活的对象都向一端移动，然后直接清理掉端边界以外的内存。

![image-20220315143038972](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220315143038972.png)



### 移动对象的效率问题

对于标记整理：如果移动存活对象，尤其是在老年代这种每次回收都有大量对象存活区域，移动存活对象并更新所有引用这些对象的地方将会是一种极为负重的操作，“Stop The World”

对于标记清除：完全不考虑移动，会导致内存碎片问题，就必须得通过空闲列表来解决内存分配问题，这又会在内存访问的时候增加额外负担，内存访问增加负担会影响应用程序的吞吐量。

**移动则内存回收更加复杂，不移动则内存分配会更复杂。**

关注吞吐量的Parallel Old收集器是基于标记整理算法的，而关注延迟的CMS收集器则是基于标记清除算法的。



## 垃圾收集器

垃圾收集器在新生代和老年代都有

新生代典型的有Serial，ParNew，Parallel Scavenge。

老年代典型的有Serial Old，Parallel Old，CMS

还有不区分年代的G1



### Serial收集器

单线程收集器，它在进行垃圾收集的时候会暂停其他所有工作线程，直到它收集结束。“Stop The World”

- 用在新生代，采用标记复制算法

  

### ParNew收集器

就是Serial收集器的多线程版本。

- 用在新生代，采用标记复制算法

**只有Serial和ParNew能和CMS配合工作**



### Parallel Scavenge收集器

也是作用于新生代，同样采用标记复制算法实现，也能够并行收集

**Parallel Scavenge 收集器关注点是吞吐量（高效率的利用 CPU）。CMS 等垃圾收集器的关注点更多的是用户线程的停顿时间（提高用户体验）。所谓吞吐量就是 CPU 中用于运行用户代码的时间与 CPU 总消耗时间的比值。**

Parallel Scavenge收集器提供了两个参数用于精确控制吞吐量，使用 Parallel Scavenge 收集器配合**自适应调节策略**，把内存管理优化交给虚拟机去完成也是一个不错的选择。



### Serial Old收集器

Serial Old收集器是Serial收集器的老年代版本，同样是单线程收集器，使用标记整理算法。



### Parallel Old收集器

Parallel Scavenge收集器的老年代版本，支持多线程并行收集，基于标记整理算法实现。



### CMS收集器

CMS收集器是一种以获取**最短回收停顿**时间为目标的收集器，作用于老年代，基于标记清除实现。

**CMS（Concurrent Mark Sweep）收集器是 HotSpot 虚拟机第一款真正意义上的并发收集器，它第一次实现了让垃圾收集线程与用户线程（基本上）同时工作。**

- **初始标记**：仅仅只是标记下`GC Roots`能直接关联到的对象，速度很快
- **并发标记**：从`GC Roots`开始遍历整个对象图的过程，这个过程耗时长，但是不需要停顿用户线程。
- **重新标记**：为了修正并发标记期间由于**用户程序的运作而导致标记产生变动**的那一部分对象的标记记录（增量更新）。
- **并发清除**：清除删掉那些标记阶段判断的已经死亡的对象，由于不需要移动对象，这个阶段也可以与用户线程并发执行。

**初始标记和重新标记需要Stop The World，但是暂停的时间很短。**

主要的优点：并发收集、低停顿。

三个明显的缺点：

- CPU敏感，因为并发标记和并发清除阶段是多线程，会占用处理器资源，可能导致线程上下文切换。
- 无法处理**浮动垃圾**，在并发标记和并发清除阶段，由于用户程序还在运行，那么就可能产生新的可回收对象，CMS无法在此次收集中处理掉它们，只好留到下次垃圾收集再清理掉。
  - 若浮动垃圾过多，导致了垃圾收集的时候内存都满了，那么就出现了**并发失败**，此时冻结用户线程，**临时启用Serial Old收集器**来重新进行老年代的垃圾回收，这样停顿时间就很长了。
- 基于标记清除，就会有内存碎片的问题，内存碎片过多，大对象分配不过来，那么就可能提前触发Full GC.

> 什么是浮动垃圾？
>
> GC Roots -> A -> B，现在垃圾收集器已经扫描完A和B了，目前来看，A和B都应该是存活的，但是如果这时候，A的引用由于用户程序的更改，变成了这样，GC Roots -> A  B，这就是浮动垃圾，B应该是要被回收的，但是应该扫描过，在此轮垃圾回收过程中就不会回收B
>
> 什么是增量更新？
>
> 本来的GC Roots的引用链是这样的：GC Roots -> A  B，扫描之后，认为A存活，但是B应该被回收，但是用户程序的改动，变成了这种情况，GC Roots -> A -> B，这时候B就不应该被回收，CMS的增量更新就能够处理这种情况，这时候会将A记录起来，在**重新标记**阶段再次处理A的引用关系。
>
> 三色标记：
>
> **白色：**表示对象还没有被垃圾收集器访问过。若遍历GC Roots完毕，这个对象还是白色的，那就判定它不可达，需要回收掉。
>
> **黑色：**表示对象已经被垃圾收集器访问过，且这个对象的所有引用都也扫描完毕，黑色的对象代表扫描过，它是安全存活的。
>
> **灰色：**表示对象已经被垃圾收集器访问过，但这个对象上至少还存在一个引用没有被扫描过。



### G1收集器

G1是一款面向服务端引用的垃圾收集器，他代替了JDK8的Parallel Scavenge和Parallel Old的组合，成为服务端模式下的默认收集器组合

**G1仍然遵循分代收集的理论设计**，但其堆内存的布局与其他收集器有很大差异：G1不再坚持固定大小及固定数量的分代区域划分，而是**把Java堆划分为多个大小相等的独立区域（Region）**，每一个Region可以根据需要，扮演新生代的Eden空间，Survivor空间或者老年代空间。

Region中有一类特殊的Humongous区域，专门用来存储大对象（超过了Region容量一半的即可判定为大对象）

G1收集器之所以能建立**可预测的停顿时间模型**，是因为它将**Region作为单次回收的最小单元**，即每次收集到的内存空间都是Region大小的整数倍，这样就可以避免了在整个Java堆进行全区域的垃圾收集。

- G1收集器去跟踪各个**Region里面垃圾堆积的价值，判断依据就是回收这个Region能得到的空间大小以及回收所需要的时间**，然后在后台维护一个**优先级列表**，每次根据用户设定允许的收集停顿时间（-XX:MaxGCPauseMillis，**默认200毫秒**），优先处理回收价值最大的Region，这也是“Garbage First”名字的由来。
- 使用Region划分内存空间，以及具有优先级的区域回收方式，保证了G1收集器在有限的时间内获取尽可能高的收集效率。



G1为每个Region设计了两个名为TAMS（Top at Mark Start）的指针，把Region中的一部分空间划分出来用于并发回收过程中的新对象分配，并发回收时新分配的对象地址都需要在指针位置以上。**G1默认在TAMS指针以上的对象是存活的。**

G1收集器运作过程大致分为四个步骤：

**初始标记：**仅仅标记一下GC Roots能直接关联到的对象，并且修改一下TAMS指针，让并发标记阶段用户线程分配对象时，分配在TAMS指针位置之上。

**并发标记：**从GC Roots开始遍历整个对象图，这一阶段耗时长，但是可以与用户线程并发执行。对象图扫描完之后，还要处理一下原始快照（SATB：Snapshot At The Beginning）记录下的并发时有可能变动的对象。

**最终标记：**这里会对用户线程做一个短暂的停顿，用于处理并发标记阶段结束后仍然遗留下来的最后少量的SATB记录。（因为在并发标记阶段，也是并发处理SATB的，此时用户程序可能又修改了引用关系，在最终标记阶段停顿用户线程，完成全部标记）

**筛选回收：**更新Region的统计数据，看哪个Region回收价值高（回收所获得的空间大小以及所需要的时间），根据用户期望的停顿时间来指定回收计划，把决定回收的那一部分Region存活对象，复制到空的Region中，然后清理掉整个旧Region的全部空间。这里涉及存活对象的移动，是必须暂停用户线程，由多条收集器线程并行完成。

> SATB：原始快照，当灰色对象要删除指向白色对象的引用关系时，就将整个要删除的引用记录下来，在并发扫描结束后，再将这些记录过的引用关系中的灰色对象为根，重新扫描一次。**可以简单理解为，无论引用关系删除与否，都会按照刚刚开始扫描那一刻的对象图快照来进行搜索。**
>
> 比如现在：GC Roots -> A -> B，现在扫描到A对象，这时候A是灰色对象，在这个时候，用户程序设置 A->null，那么，A->B这个引用关系就会被记录下来，并发扫描结束后，重新以A为根，再扫描一次。



**G1突出的地方：**

- 不再追求一次把整个Java堆清理干净，而是优先回收价值高的Region
- G1整体基于标记整理算法，局部（两个Region之间）基于标记复制，这就不会产生内存碎片，垃圾收集完毕之后内存是规整的
- 可以根据用户期望的停顿时间（默认是200毫秒）来决定具体回收哪些Region

> 怎么建立可靠的停顿预测模型的？
>
> 垃圾收集过程中，G1收集器会记录每个Region的回收耗时、每个Region记忆集里脏卡的数量等统计信息，通过这些信息进行预测。
>
> - Region的统计状态越新越能决定其回收的价值。



### 现在jdk默认使用的是哪种垃圾回收器？

JDK7 默认垃圾收集器Parallel Scavenge（新生代）+Parallel Old（老年代）
JDK8 默认垃圾收集器Parallel Scavenge（新生代）+Parallel Old（老年代）
JDK9 默认垃圾收集器G1



## 内存分配与回收的策略是怎样的？

Java技术体系的自动内存管理，最根本的目标是自动化的解决两个问题：**自动给对象分配内存以及自动回收分配给对象的内存。**

**不同版本的收集器策略是不一样的，这里记录的是书里面的例子**

### 对象优先在Eden分配

大多数情况下，对象在新生代Eden区中分配，当Eden区没有足够空间的时候，虚拟机将发起一次Minor GC

```java
//Java堆大小为20M，不可扩展，10MB分配给新生代，Eden区和一个Survivor的比例是8:1
//-verbose:gc -Xms20M -Xmx20M -Xmn10M -XX:+PrintGCDetails -XX:SurvivorRatio=8
//那么此时 Eden区有8MB,from区有1MB,to区有1MB
public class Main {

    private static final int _1MB = 1024 * 1024;

    public static void main(String[] args){
        byte[] a1,a2,a3,a4;
        a1 = new byte[2 * _1MB];
        a2 = new byte[2 * _1MB];
        a3 = new byte[2 * _1MB];
        a4 = new byte[4 * _1MB]; //出现一次Minor GC
    }
}
```

此次收集结束后，4MB的a4对象顺利分配在Eden，而老年代占用6MB，因为2MB的a1、a2、a3无法进入to区，只能通过分配担保机制提前进入老年代。



### 大对象直接进入老年代

大对象就是指需要大量连续内存空间的Java对象，最典型的大对象便是那种很长的字符串或者元素数量很多的数组。

- 要避免大对象的原因主要是：
  - 在分配空间时，它容易导致内存明明还有不少空间就提前触发垃圾回收，以获取足够的连续空间安置它们
  - 而当复制对象时，就意味着高额的内存复制开销。
  - HotSpot提供`-XX:PretenureSizeThreshold`,指定大于该值的对象直接在老年代分配，避免了在Eden区及两个Survivor区之间来回复制，产生大量内存复制操作。

```java
//-verbose:gc -Xms20M -Xmx20M -Xmn10M -XX:+PrintGCDetails -XX:SurvivorRatio=8 
//-XX:PretenureSizeThreshold=3145728
//-XX:PretenureSizeThreshold只对Serial和ParNew两款新生代收集器有效。
public class Main {

    private static final int _1MB = 1024 * 1024;

    public static void main(String[] args){
        byte[] a1,a2,a3,a4;
        a4 = new byte[4 * _1MB];
    }
}
```



### 长期存活的对象直接进入老年代

虚拟机给每个对象定义了对象年龄计数器，存储在对象头。

通常对象在Eden诞生，如果经过一个Minor GC后仍然存活，能够被Survivor区容纳，那么进入Survivor区，并且年龄为一岁，对象在Survivor区每熬过一次垃圾回收，年龄就加1，默认到15岁就被晋升到老年代中。

> 对象晋升到老年代的年龄阈值，可以通过参数-XX:MaxTenuringThreshold设置



### 动态对象年龄判定

为了能更好地适应不同程序的内存状况，虚拟机并不是永远地要求对象的年龄必须达到了MaxTenuringThreshold才能晋升老年代，如果**在Survivor空间中低于某个年龄的所有对象的总和超过了Survivor空间的一半**,那么大于等于该年龄的所有对象直接晋升到老年代，而不用等到`-XX:MaxTenuringThreshold`要求的年龄



### 空间分配担保

场景：Eden区和from区和to区的比例是8:1:1，默认将对象复制到to区，若to区不够放怎么办？

**需要老年代进行分配担保，把Survivor无法容纳的对象直接送入老年代。**

**只要老年代的连续空间大于新生代所有对象的总和或者历此平均晋升到老年代对象的总和，就进行Minor GC，否则进行Full GC**



## 类文件结构

```java
ClassFile {
    u4             magic; //Class 文件的标志
    u2             minor_version;//Class 的小版本号
    u2             major_version;//Class 的大版本号
    u2             constant_pool_count;//常量池的数量
    cp_info        constant_pool[constant_pool_count-1];//常量池
    u2             access_flags;//Class 的访问标记
    u2             this_class;//当前类
    u2             super_class;//父类
    u2             interfaces_count;//接口
    u2             interfaces[interfaces_count];//一个类可以实现多个接口
    u2             fields_count;//Class 文件的字段属性
    field_info     fields[fields_count];//一个类可以有多个字段
    u2             methods_count;//Class 文件的方法数量
    method_info    methods[methods_count];//一个类可以有个多个方法
    u2             attributes_count;//此类的属性表中的属性数
    attribute_info attributes[attributes_count];//属性表集合
}
```

![image-20220315163644904](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220315163644904.png)

每当 Java 发布大版本（比如 Java 8，Java9）的时候，主版本号都会加 1。你可以使用 `javap -v` 命令来快速查看 Class 文件的版本号信息。

**高版本的 Java 虚拟机可以执行低版本编译器生成的 Class 文件，但是低版本的 Java 虚拟机不能执行高版本编译器生成的 Class 文件**。所以，我们在实际开发的时候要确保开发的 JDK 版本和生产环境的 JDK 版本保持一致。



## 类加载

### 类的生命周期

![image-20220315160742390](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220315160742390.png)



### 类加载的过程

Class 文件需要加载到虚拟机中之后才能运行和使用，那么虚拟机是如何加载这些 Class 文件呢？

系统加载 Class 类型的文件主要三步：**加载->连接->初始化**。连接过程又可分为三步：**验证->准备->解析**。

#### 加载

1. 通过全类名获取定义此类的二进制字节流
2. 将字节流所代表的静态存储结构转换为方法区的运行时数据结构
3. 在堆内存中生成一个代表该类的 `Class` 对象，作为方法区这些数据的访问入口

**如何获取某个类的二进制字节流？**

- zip压缩包中读取（日后JAR、WAR格式的基础）
- 运行时计算生成，动态代理技术，在java.lang.reflect.Proxy中，ProxyGenerator.generateProxyClass()来为特定接口生成形式为"*$Proxy"的代理类的二进制字节流
- JSP引用，由JSP文件生成对应的Class文件

> 当服务器上的一个JSP页面被第一次请求执行时，服务器上的JSP引擎首先将JSP页面文件转译成一个java文件，并编译这个java文件生成字节码文件，然后执行字节码文件响应客户的请求



#### 验证

- 文件格式验证：
  - 看一下开头的魔数对不对，是不是CAFEBABE
  - 看一下这个class文件里面的jdk版本号能不能被当前虚拟机处理
- 元数据验证：
  - 看一下这个类有没有父类（除了java.lang.Object之外的类都应该有父类）
  - 看一下这个类的继承是不是正确的，有没有继承到一个final的类（不允许的操作）
  - 若这个类不是抽象类的话，看一下这个类有没有实现所有的方法

![image-20220315162356638](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220315162356638.png)



#### 准备

**准备阶段是正式为类变量分配内存并设置类变量初始值的阶段**，这些内存都将在方法区中分配。对于该阶段有以下几点需要注意：

1. 这时候进行内存分配的仅包括类变量（ Class Variables ，即静态变量，被 `static` 关键字修饰的变量，只与类相关，因此被称为类变量），而不包括实例变量。实例变量会在对象实例化时随着对象一块分配在 Java 堆中。
2. 从概念上讲，类变量所使用的内存都应当在 **方法区** 中进行分配。不过有一点需要注意的是：JDK 7 之前，HotSpot 使用永久代来实现方法区的时候，实现是完全符合这种逻辑概念的。 而在 JDK 7 及之后，**HotSpot 已经把原本放在永久代的字符串常量池、静态变量等移动到堆中，这个时候类变量则会随着 Class 对象一起存放在 Java 堆**中。
3. 这里所设置的初始值"通常情况"下是数据类型默认的零值（如 0、0L、null、false 等），比如我们定义了`public static int value=111` ，那么 value 变量在准备阶段的初始值就是 0 而不是 111（初始化阶段才会赋值）。特殊情况：比如给 value 变量加上了 final 关键字`public static final int value=111` ，那么准备阶段 value 的值就被赋值为 111。



#### 解析

解析阶段是虚拟机将常量池内的符号引用替换为直接引用的过程。

符号引用就是一组符号来描述目标，可以是任何字面量。**直接引用就是直接指向目标的指针、相对偏移量或一个间接定位到目标的句柄**。在程序实际运行时，只有符号引用是不够的，举个例子：在程序执行方法时，系统需要明确知道这个方法所在的位置。Java 虚拟机为每个类都准备了一张方法表来存放类中所有的方法。当**需要调用一个类的方法的时候，只要知道这个方法在方法表中的偏移量就可以直接调用该方法了。通过解析操作符号引用就可以直接转变为目标方法在类中方法表的位置，从而使得方法可以被调用**。

综上，解析阶段是虚拟机将常量池内的符号引用替换为直接引用的过程，也就是得到类或者字段、方法在内存中的指针或者偏移量。

- 类的引用字段，得到一个指向目标的直接指针
- 类的方法，得到一个它在方法表中的偏移量

> 在类加载的解析阶段，对于方法的符号引用，会将一部分符号引用转化为直接引用，能够转化的前提是，这些方法在程序真正运行之前就有一个可确定的调用版本，并且这个调用版本在运行期是不可变的。（主要有static方法，私有方法）



#### 初始化

初始化阶段是执行初始化方法 `<clinit> ()`方法的过程，是类加载的最后一步，这一步 JVM 才开始真正执行类中定义的 Java 程序代码(字节码)。

对于`<clinit> ()` 方法的调用，虚拟机会自己确保其在多线程环境中的安全性。因为 `<clinit> ()` 方法是带锁线程安全，所以在多线程环境下进行类初始化的话可能会引起多个进程阻塞，并且这种阻塞很难被发现。

对于初始化阶段，出现以下情况下，必须对类进行初始化(只有主动去使用类才会初始化类)：

- 当遇到 `new` 、 `getstatic`、`putstatic` 或 `invokestatic` 这 4 条直接码指令时，比如 `new` 一个类，读取一个静态字段(未被 final 修饰)、或调用一个类的静态方法时
  - 当 jvm 执行 `new` 指令时会初始化类。即当程序创建一个类的实例对象。
  - 当 jvm 执行 `getstatic` 指令时会初始化类。即程序访问类的静态变量(不是静态常量，常量会被加载到运行时常量池)。
  - 当 jvm 执行 `putstatic` 指令时会初始化类。即程序给类的静态变量赋值。
  - 当 jvm 执行 `invokestatic` 指令时会初始化类。即程序调用类的静态方法。
- 使用 `java.lang.reflect` 包的方法对类进行反射调用时如 `Class.forname("...")`, `newInstance()` 等等。如果类没初始化，需要触发其初始化。
- 初始化一个类，如果其父类还未初始化，则先触发该父类的初始化。
- 当虚拟机启动时，用户需要定义一个要执行的主类 (包含 `main` 方法的那个类)，虚拟机会先初始化这个类。
- 当一个接口中定义了 JDK8 新加入的默认方法（被 default 关键字修饰的接口方法）时，如果有这个接口的实现类发生了初始化，那该接口要在其之前被初始化。



#### 卸载

卸载类即该类的 Class 对象被 GC。

卸载类需要满足 3 个要求:

1. 该类的所有的实例对象都已被 GC，也就是说堆不存在该类的实例对象。
2. 该类没有在其他任何地方被引用
3. 该类的类加载器的实例已被 GC

**所以，在 JVM 生命周期内，由 jvm 自带的类加载器加载的类是不会被卸载的。但是由我们自定义的类加载器加载的类是可能被卸载的。**

jdk 自带的 `BootstrapClassLoader`, `ExtClassLoader`, `AppClassLoader` 负责加载 jdk 提供的类，所以它们(类加载器的实例)肯定不会被回收。而我们自定义的类加载器的实例是可以被回收的，所以**使用我们自定义加载器加载的类是可以被卸载掉的**。而JDK自带的类不会被卸载。



## 类加载器

一个非数组类的加载阶段（加载阶段获取类的二进制字节流的动作）是可控性最强的阶段，这一步我们可以去自定义类加载器去控制字节流的获取方式（重写一个类加载器的 `loadClass()` 方法）。**数组类型不通过类加载器创建，它由 Java 虚拟机直接创建。**

所有的类都由类加载器加载，加载的作用就是将 `.class`文件加载到内存。



### 三层类加载器

JVM 中内置了三个重要的 ClassLoader，除了 BootstrapClassLoader 其他类加载器均由 Java 实现且全部继承自`java.lang.ClassLoader`：

1. **BootstrapClassLoader(启动类加载器)** ：最顶层的加载类，由 C++实现，负责加载 `%JAVA_HOME%/lib`目录下的 jar 包和类或者被 `-Xbootclasspath`参数指定的路径中的所有类。（rt.jar，tools.jar）
2. **ExtensionClassLoader(扩展类加载器)** ：主要负责加载 `%JAVA_HOME%/lib/ext` 目录下的 jar 包和类，或被 `java.ext.dirs` 系统变量所指定的路径下的 jar 包。
3. **AppClassLoader(应用程序类加载器)** ：面向我们用户的加载器，负责加载当前应用 classpath 下的所有 jar 包和类。



### 双亲委派模型

每一个类都有一个对应它的类加载器。系统中的 ClassLoader 在协同工作的时候会默认使用 **双亲委派模型** 。即在类加载的时候，系统会首先判断当前类是否被加载过。已经被加载的类会直接返回，否则才会尝试加载。加载的时候，首先会把该请求委派给父类加载器的 `loadClass()` 处理，因此所有的请求最终都应该传送到顶层的启动类加载器 `BootstrapClassLoader` 中。当父类加载器无法处理时，才由自己来处理。当父类加载器为 null 时，会使用启动类加载器 `BootstrapClassLoader` 作为父类加载器。

<img src="https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220315165956557.png" alt="image-20220315165956557" style="zoom:67%;" />

```java
public class ClassLoaderDemo {
    public static void main(String[] args) {
        System.out.println(
            "ClassLodarDemo's ClassLoader is " + ClassLoaderDemo.class.getClassLoader()
        );
        System.out.println(
            "The Parent of ClassLodarDemo's ClassLoader is " + ClassLoaderDemo.class.getClassLoader()
            .getParent()
        );
        System.out.println(
            "The GrandParent of ClassLodarDemo's ClassLoader is " + ClassLoaderDemo.class.getClassLoader()
            .getParent().getParent());
    }
}
/*
ClassLodarDemo's ClassLoader is sun.misc.Launcher$AppClassLoader@18b4aac2
The Parent of ClassLodarDemo's ClassLoader is sun.misc.Launcher$ExtClassLoader@1b6d3586
The GrandParent of ClassLodarDemo's ClassLoader is null
*/
```

`AppClassLoader`的父类加载器为`ExtClassLoader`， `ExtClassLoader`的父类加载器为 null，**null 并不代表`ExtClassLoader`没有父类加载器，而是 `BootstrapClassLoader`** 



### 源码实现

```java
private final ClassLoader parent;
protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // 首先，检查请求的类是否已经被加载过
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                //若没有加载过,先尝试委托给父类去加载
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {//父加载器不为空，调用父加载器loadClass()方法处理
                        c = parent.loadClass(name, false);
                    } else {//父加载器为空，使用启动类加载器 BootstrapClassLoader 加载
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                   //抛出异常说明父类加载器无法完成加载请求
                    //此时调用自己的findClass尝试加载。
                }

                if (c == null) {
                    long t1 = System.nanoTime();
                    //自己尝试加载
                    c = findClass(name);

                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }
```



### 双亲委派模型的好处

Java中的类随着它的类加载器一起具备了一种带有优先级的层次关系。比如java.lang.Object，它存在于rt.jar中，无论哪一个类加载器要加载这个类，最终都是委托给启动类加载器去加载，这样就保证了Object类在程序的各种类加载器环境中都能保证是同一个类。

如果没有使用双亲委派模型，而是每个类加载器加载自己的话就会出现一些问题，比如我们编写一个称为 `java.lang.Object` 类的话，那么程序运行的时候，系统就会出现多个不同的 `Object` 类，Java类型体系中最基础的行为也就无从保证。



### 如果我们不想用双亲委派模型怎么办？

自定义加载器的话，需要继承 `ClassLoader` 。如果我们不想打破双亲委派模型，就重写 `ClassLoader` 类中的 `findClass()` 方法即可，无法被父类加载器加载的类最终会通过这个方法被加载。但是，如果想打破双亲委派模型则需要重写 `loadClass()` 方法



## Java内存模型

Java并发采用的是共享内存模型。

Java线程之间的通信由Java内存模型（JMM）控制，JMM决定一个线程对共享变量的写入何时对另一个线程可见。JMM定义了线程和主内存之间的抽象关系：**线程之间的共享变量存储在主内存中，每个线程都有一个私有的本地内存，本地内存中存储了该线程以读/写共享变量的副本。**本地内存式JMM的一个抽象概念，并不真实存在。

![image-20220324201125183](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220324201125183.png)

从上图来看，如果线程A要和线程B通信的话，需要经历两个步骤

1、线程A把本地内存A中更新过的共享变量刷新到主内存中去

2、线程B到主内存中读取线程A之前已更新过的共享变量。



## 从源代码到指令序列的重排序

在执行程序时，为了提高性能，编译器和处理器常常会对指令做重排序。重排序分3种类型：

1、编译器优化的重排序。编译器可以在不改变单线程程序语义的前提下，重新安排语句的执行顺序。

2、指令级并行的重排序。现代处理器采用了指令级并行技术（Instruction-Level Parallelism,ILP）来将多条指令重叠执行。如果不存在数据依赖性，处理器可以改变语句对应机器指令的执行顺序。

3、内存系统的重排序。由于处理器使用缓存和读/写缓冲区，这使得加载和存储操作看上去可能是在乱序执行。

**因此，从Java源代码到最终实际执行的指令序列，会分别经历下面3种重排序：**

源代码 -> 编译器优化重排序 -> 指令级并行重排序 -> 内存系统重排序 -> 最终指令序列



## 原子性、可见性、有序性

并发编程的三大特性就是原子性、可见性、有序性。

### 原子性

Java内存模型来直接保证的原子性变量操作包括read、load、assign、use、store和write这六个，基本数据类型的访问、读写都是具备原子性的（long和double的非原子性协定）

如果应用场景需要更大范围的原子性保证，尽管虚拟机没有把loc和unlock操作开放给用户，但是提供了monitorenter和monitorexit来隐式使用这两个操作。即synchronized关键字，同步块之间的操作也具有原子性。



### 可见性

可见性就是指当一个线程修改了共享变量的值时，其他线程能够立即得知这个修改。

**Java内存模型是通过在变量修改后将新值同步回主内存，在变量读取前从主内存刷新变量值这种依赖主内存作为传递媒介的方式实现可见性的。**

volatile：写一个变量时，会在写变量后加入一条内存屏障指令，JMM会把该线程对应的本地变量刷新到主内存；读一个变量时，JMM会把该线程对应的本地变量内存置为无效，线程接下来将从主内存读共享变量。

synchronized：对一个变量unlock操作之前，必须先把此变量同步回主内存中（执行store,write）

final：被final修饰的字段在构造器中一旦完成初始化，并且构造器没有把this的引用传递出去

> 内存屏障指令
>
> - 保证特定操作的执行顺序
> - 保证某些变量的内存可见性（利用该特性实现volatile的内存可见性）

#### this指针逃逸

假设线程A执行write方法，线程B执行reader方法。这里的操作2使得对象还没有初始化完毕就对线程B可见(调用了reader)，且这时候操作1和操作2可能重排序，这就可能出现问题，使得线程B读到了i还没有正确初始化的值

```java
public class FinalExample{
    final int i;
    static FinalExample fe;
    
    public FinalExample(){
        i = 20;		//1
        fe = this;  //2
    }
    
    public static void write(){
        new FinalExample();
    }
    
    public static void reader(){
        if(fe != null)		//3
            int tmp = fe.i; //4
    }
}
```



### 有序性

如果在本线程内观察，所有操作都是有序的。

如果在一个线程中观察另一个线程，所有操作都是无序的（指令重排序和工作内存与主内存同步延迟）

volatile：通过插入内存屏障来保证它的禁止指令重排序

synchronized：一个变量在同一时刻只允许一条线程对它进行lock操作，且对一个变量unlock之前，必须把此变量同步回主内存，保证了持有同一个锁的两个同步块只能串行的进入。



## happens-before

如果一个操作happens-before另一个操作，那么第一个操作的执行结果将对第二个操作可见，而且第一个操作的执行顺序排在第二个操作之前。

> as-if-serial语义：不管怎么重排序（编译器或处理器为了提高并行度），单线程程序的执行结果不能被改变。

**as-if-serial语义保证单线程内程序的执行结果不被改变，happens-before关系保证正确同步的多线程程序的执行结果不被改变，它们都是为了在不改变程序执行结果的前提下，尽可能提高程序的并发度。**



1、程序顺序规则：一个线程中的每个操作，happens-before于该线程中的任意后续操作。

2、监视器锁规则：对一个锁的解锁，happens-before于后续对这个锁的加锁

3、volatile变量规则：对一个volatile变量的写，happens-before于任意后续对这个volatile变量的读

4、传递性：如果A happens-before B，B happens-before C，那么A happens-before C

5、start()规则：如果线程A执行threadB.start()【启动线程B】，那么A线程的threadB.start()操作happens-before于线程B中的任意操作

6、join()规则：如果线程A执行操作threadB.join()并成功返回，那么线程B中的后续的任意操作happens-before于线程A从threadB.join()操作成功返回。
