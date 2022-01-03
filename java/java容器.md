- [java容器](#java容器)
  - [Collection](#collection)
    - [ArrayList](#arraylist)
      - [1.RandomAccess](#1randomaccess)
      - [2. 扩容](#2-扩容)
      - [3. 删除元素](#3-删除元素)
      - [4. 序列化](#4-序列化)
      - [5. Fail-Fast](#5-fail-fast)
    - [Vector](#vector)
      - [1. 同步](#1-同步)
      - [2. 扩容](#2-扩容-1)
      - [3. 与 ArrayList 的比较](#3-与-arraylist-的比较)
      - [4. 替代方案](#4-替代方案)
    - [CopyOnWriteArrayList](#copyonwritearraylist)
      - [写时复制](#写时复制)
      - [读写分离](#读写分离)
      - [适用场景](#适用场景)
    - [LinkedList](#linkedlist)
  - [Map](#map)
    - [Map的实现类的结构](#map的实现类的结构)
    - [HashMap理解](#hashmap理解)
    - [HashMap的底层实现原理(jdk7)](#hashmap的底层实现原理jdk7)
      - [初始化方法](#初始化方法)
      - [一般为了不让它扩容[比较耗时],可以算估算一下map的初始化容量，虽然容量可以随意指定，但是它最后都被转换成2的次幂【原因见下文分析】](#一般为了不让它扩容比较耗时可以算估算一下map的初始化容量虽然容量可以随意指定但是它最后都被转换成2的次幂原因见下文分析)
      - [put方法细节分析【分5点具体分析】](#put方法细节分析分5点具体分析)
      - [扩容细节](#扩容细节)
      - [get方法细节](#get方法细节)
    - [HashMap的底层实现原理(jdk8)](#hashmap的底层实现原理jdk8)
    - [ConcurrentHashMap](#concurrenthashmap)

# java容器

## Collection

### ArrayList

#### 1.RandomAccess

- 该接口标识着这个类支持快速随机访问

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```

数组的默认大小为 10。

```java
//jdk7的时候在初始化时就创建了数组
//jdk8做了变化：当第一次使用add操作的时候才创建数组
private static final int DEFAULT_CAPACITY = 10;
```

#### 2. 扩容

添加元素时使用 ensureCapacityInternal() 方法来保证容量足够，如果不够时，需要使用 grow() 方法进行扩容，新容量的大小为 `oldCapacity + (oldCapacity >> 1)`，即 oldCapacity+oldCapacity/2。其中 oldCapacity >> 1 需要取整，所以新容量大约是旧容量的 1.5 倍左右。（oldCapacity 为偶数就是 1.5 倍，为奇数就是 1.5 倍-0.5）

扩容操作需要调用 `Arrays.copyOf()` 把原数组整个复制到新数组中，这个操作代价很高，因此最好在创建 ArrayList 对象时就指定大概的容量大小，减少扩容操作的次数。

```java
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}

private void ensureCapacityInternal(int minCapacity) {
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    ensureExplicitCapacity(minCapacity);
}

private void ensureExplicitCapacity(int minCapacity) {
    modCount++;
    // overflow-conscious code
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}

private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);
}

```

#### 3. 删除元素

需要调用 System.arraycopy() 将 index+1 后面的元素都复制到 index 位置上，该操作的时间复杂度为 O(N)，可以看到 ArrayList 删除元素的代价是非常高的。

```java
public E remove(int index) {
    rangeCheck(index);
    modCount++;
    E oldValue = elementData(index);
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index, numMoved);
    elementData[--size] = null; // clear to let GC do its work
    return oldValue;
}
```

#### 4. 序列化

ArrayList 基于数组实现，并且具有动态扩容特性，因此保存元素的数组不一定都会被使用，那么就没必要全部进行序列化。

保存元素的数组 elementData 使用 transient 修饰，该关键字声明数组默认不会被序列化。

```java
transient Object[] elementData; // non-private to simplify nested class access
```

ArrayList 实现了 writeObject() 和 readObject() 来控制只序列化数组中有元素填充那部分内容。

```java
private void readObject(java.io.ObjectInputStream s)
    throws java.io.IOException, ClassNotFoundException {
    elementData = EMPTY_ELEMENTDATA;

    // Read in size, and any hidden stuff
    s.defaultReadObject();

    // Read in capacity
    s.readInt(); // ignored

    if (size > 0) {
        // be like clone(), allocate array based upon size not capacity
        ensureCapacityInternal(size);

        Object[] a = elementData;
        // Read in all elements in the proper order.
        for (int i=0; i<size; i++) {
            a[i] = s.readObject();
        }
    }
}
private void writeObject(java.io.ObjectOutputStream s)
    throws java.io.IOException{
    // Write out element count, and any hidden stuff
    int expectedModCount = modCount;
    s.defaultWriteObject();

    // Write out size as capacity for behavioural compatibility with clone()
    s.writeInt(size);

    // Write out all elements in the proper order.
    for (int i=0; i<size; i++) {
        s.writeObject(elementData[i]);
    }

    if (modCount != expectedModCount) {
        throw new ConcurrentModificationException();
    }
}
```

序列化时需要使用 ObjectOutputStream 的 writeObject() 将对象转换为字节流并输出。而 writeObject() 方法在传入的对象存在 writeObject() 的时候会去反射调用该对象的 writeObject() 来实现序列化。反序列化使用的是 ObjectInputStream 的 readObject() 方法，原理类似。

```java
ArrayList list = new ArrayList();
ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream(file));
oos.writeObject(list);
```

#### 5. Fail-Fast

modCount 用来记录 ArrayList 结构发生变化的次数。结构发生变化是指添加或者删除至少一个元素的所有操作，或者是调整内部数组的大小，仅仅只是设置元素的值不算结构发生变化。

在进行序列化或者迭代等操作时，需要比较操作前后 modCount 是否改变，如果改变了需要抛出 ConcurrentModificationException。代码参考上节序列化中的 writeObject() 方法。



### Vector

#### 1. 同步

它的实现与 ArrayList 类似，但是使用了 synchronized 进行同步。

```java
public synchronized boolean add(E e) {
    modCount++;
    ensureCapacityHelper(elementCount + 1);
    elementData[elementCount++] = e;
    return true;
}

public synchronized E get(int index) {
    if (index >= elementCount)
        throw new ArrayIndexOutOfBoundsException(index);

    return elementData(index);
}
```

#### 2. 扩容

Vector 的构造函数可以传入 capacityIncrement 参数，它的作用是在扩容时使容量 capacity 增长 capacityIncrement。如果这个参数的值小于等于 0，扩容时每次都令 capacity 为原来的两倍。

```java
public Vector(int initialCapacity, int capacityIncrement) {
    super();
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    this.elementData = new Object[initialCapacity];
    this.capacityIncrement = capacityIncrement;
}
private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + ((capacityIncrement > 0) ?
                                     capacityIncrement : oldCapacity);
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

调用没有 capacityIncrement 的构造函数时，capacityIncrement 值被设置为 0，也就是说默认情况下 Vector 每次扩容时容量都会翻倍。

```java
public Vector(int initialCapacity) {
    this(initialCapacity, 0);
}

public Vector() {
    this(10);
}
```

#### 3. 与 ArrayList 的比较

- Vector 是同步的【每个可能引起线程安全问题的方法都在方法上标注了`synchronized`】，因此开销就比 ArrayList 要大，访问速度更慢。最好使用 ArrayList 而不是 Vector，因为同步操作完全可以由程序员自己来控制；
- Vector 每次扩容请求其大小的 2 倍（也可以通过构造函数设置增长的容量），而 ArrayList 是 1.5 倍。
- Vector在初始化时就创建了数组，ArrayList在第一次添加的时候才创建【jdk8】

#### 4. 替代方案

可以使用 `Collections.synchronizedList();` 得到一个线程安全的 ArrayList。

```java
List<String> list = new ArrayList<>();
List<String> synList = Collections.synchronizedList(list);
```

```java
//这个效率也不高，底层都是这种代码
synchronized(mutex){
    ...
}
//而mutex就是这个线程安全的ArrayList
SynchronizedCollection(Collection<E> c) {
    this.c = Objects.requireNonNull(c);
    mutex = this;
}
```



也可以使用 concurrent 并发包下的 CopyOnWriteArrayList 类。

```java
List<String> list = new CopyOnWriteArrayList<>();
```



### CopyOnWriteArrayList

#### 写时复制

```tex
CopyOnWrite容器即写时复制的容器。往一个容器添加元素的时候，不直接往当前容器Object[]添加，而是先将当前容器Object[]进行Copy，复制出一个新的容器Object[] newElements，然后向新的容器Object[] newElements里添加元素。
添加元素后，再将原容器的引用指向新的容器setArray(newElements)。
这样做的好处是可以对CopyOnWrite容器进行并发的读，而不需要加锁，因为当前容器不会添加任何元素。
所以CopyOnWrite容器也是一种读写分离的思想，读和写不同的容器。
```

#### 读写分离

```java
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();   
        int len = elements.length;
        Object[] newElements = Arrays.copyOf(elements, len + 1); //复制出一个新数组,比原来的长度多了1
        newElements[len] = e;
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
final Object[] getArray() {
    return array;
}
final void setArray(Object[] a) {
    array = a;
}
public E get(int index) {
    return get(getArray(), index);
}
```

- 写操作在一个复制的数组上进行，读操作还是在原来的数组上操作，读写分离，互不影响
- 写操作需要加锁，防止并发写入导致的数据写入丢失
- 写操作结束后需要把原来数组指向新的复制数组

#### 适用场景

CopyOnWriteArrayList 在写操作的同时允许读操作，大大提高了读操作的性能，因此很适合读多写少的应用场景。

但是 CopyOnWriteArrayList 有其缺陷：

- 内存占用：在写操作时需要复制一个新的数组，使得内存占用为原来的两倍左右；
- 数据不一致：读操作不能读取实时性的数据，因为部分写操作的数据还未同步到读数组中。
- 效率不高：写操作复制一个新的数据需要O(n)的时间，故一次写操作从原来的O(1)变成了O(n)

所以 CopyOnWriteArrayList 不适合内存敏感以及对实时性要求很高的场景。



### LinkedList

TODO



## Map

### Map的实现类的结构

![image-20210928162559003](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20210928162559003.png)

**HashMap**：作为Map的主要实现类，线程不安全，效率高，可以存储null的key和value

- HashMap的底层
  - 数组+链表( jdk7及之前 )
  - 数组+链表+红黑树（jdk8）

**LinkedHashMap**：继承了HashMap，保证在遍历map元素的时候，可以按照添加的顺序实现遍历。

- 原因：在原有HashMap底层结构基础上，添加了一对指针，指向前一个和后一个元素【双向链表】
- 对于频繁的遍历操作，此类执行效率高于HashMap

**TreeMap**：保证按照添加的key-value对进行排序，实现排序遍历。此时考虑key的自然排序或定制排序【Comparator<? super K>】，底层使用红黑树

**Hashtable**：作为古老的实现类，线程安全的，效率低，不能存储null的key和value，提交表单的时候不能采用它

**Properties**：常用来处理配置文件。key和value都是String类型



### HashMap理解

![image-20210928182059503](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20210928182059503.png)

**key**：无序的，不可重复的，使用Set存储所有的key --> key所在的类要重写equals和hashCode

**value**：无序的，可重复的、使用Collection存储所有的value --> value所在的类要重写equals

**键值对**：key-value构成了一个Entry对象，它是无序的，不可重复的，使用Set存储所有的entry

### HashMap的底层实现原理(jdk7)

```java
HashMap map = new HashMap();
在实例化以后,底层创建了长度是16的一维数组Entry[] table.
map.put(key,value);
首先,调用key所在类的hashCode()计算key的hash,此时hash经过某种算法计算以后，得到Entry数组中的存放位置
如果此位置上的数据为空,此时key-value添加成功 --> new Entry<K,V>(hash,key,value)放到指定位置【头插法】
如果此位置上的数据不为空,(意味着此位置上存在一个或多个数据,以链表形式存在),比较key和已经存在的一个或多个数据的哈希值：
    如果key的哈希值与已经存在的数据的哈希值都不相同,此时key-value添加成功
    如果key的哈希值与已经存在的某一个数据的哈希值相同,调用key所在类的equals方法比较
    	如果equals返回false:继续遍历链表,若全是false,key-value添加成功
        如果equals返回true: 使用value去替换相同key的value值
            
在不断的添加过程中，会涉及到扩容问题【超过threshold[capacity * load factor]】，默认的扩容方式：扩容为原来容量的2倍。并将原来的数据复制过来【entry的位置可能发生变化,看下面分析】
```

![image-20210928183743938](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20210928183743938.png)

#### 初始化方法

```java
/**
* Constructs an empty <tt>HashMap</tt> with the default initial capacity
* (16) and the default load factor (0.75).
*/
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    threshold = (int)(DEFAULT_INITIAL_CAPACITY * DEFAULT_LOAD_FACTOR);
    table = new Entry[DEFAULT_INITIAL_CAPACITY];
    init(); //留给子类的钩子方法,LinkedHashMap就实现了它
}
```

#### 一般为了不让它扩容[比较耗时],可以算估算一下map的初始化容量，虽然容量可以随意指定，但是它最后都被转换成2的次幂【原因见下文分析】

```java
public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}
public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);

    // Find a power of 2 >= initialCapacity
    int capacity = 1;
    while (capacity < initialCapacity) //logN的时间算出初始化容量,必须是2的次幂
        capacity <<= 1;

    this.loadFactor = loadFactor;
    threshold = (int)(capacity * loadFactor);
    table = new Entry[capacity];
    init();
}
```

#### put方法细节分析【分5点具体分析】

```java
public V put(K key, V value) {
    if (key == null)
        return putForNullKey(value); 		 				//1
    int hash = hash(key.hashCode()); 					    //2
    int i = indexFor(hash, table.length); 					//3
    for (Entry<K,V> e = table[i]; e != null; e = e.next) {  //4
        Object k;
        if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
            V oldValue = e.value;
            e.value = value;
            e.recordAccess(this);
            return oldValue;
        }
    }

    modCount++;
    addEntry(hash, key, value, i);							//5
    return null;
}
```

**1.前面提到,Hashtable不能存null的值,但是HashMap可以,这里就能发现,而且它会把key为null的这个entry放在table[0]的位置**

```java
private V putForNullKey(V value) {
    for (Entry<K,V> e = table[0]; e != null; e = e.next) {
        //原本这个桶可能已经有元素了,且也可能有key=null的entry了,先找一下,找到的话就更新
        if (e.key == null) {  
            V oldValue = e.value;
            e.value = value;
            e.recordAccess(this);
            return oldValue;
        }
    }
    modCount++;
    //没找到的话就头插法,将这个key=null的entry放在首位
    addEntry(0, null, value, 0);
    return null;
}
```

**2.计算key的hash值**

**从这里也明白,必须重写key的hashCode方法，然后计算出key的hash值后,调用HashMap的静态方法hash再算一遍，得到key最后的hash值**

```java
int hash = hash(key.hashCode());
//看不懂
static int hash(int h) {
    // This function ensures that hashCodes that differ only by
    // constant multiples at each bit position have a bounded
    // number of collisions (approximately 8 at default load factor).
    h ^= (h >>> 20) ^ (h >>> 12);
    return h ^ (h >>> 7) ^ (h >>> 4);
}
```

**3.计算要放在哪个桶**

```java
int i = indexFor(hash, table.length);
/**
* Returns index for hash code h.
*/
static int indexFor(int h, int length) {
    return h & (length-1);
}
```

确定桶下标的最后一步是将 key 的 hash 值对桶个数取模：hash%capacity，如果能保证 capacity 为 2 的 n 次方，那么就可以将这个操作转换为位运算。

**从这里也可以看出，为什么HashMap的容量一定是2的次幂 -> 为了找桶下标更快**

令 x = 1<<4，即 x 为 2 的 4 次方，它具有以下性质：

```bash
x   : 00010000
x-1 : 00001111
```

令一个数 y 与 x-1 做与运算，可以去除 y 位级表示的第 4 位以上数：

```bash
y       : 10110010
x-1     : 00001111
y&(x-1) : 00000010
```

这个性质和 y 对 x 取模效果是一样的：

```bash
y   : 10110010
x   : 00010000
y%x : 00000010
```

我们知道，位运算的代价比求模运算小的多，因此在进行这种计算时用位运算的话能带来更高的性能。



**4.如果找到的这个桶,已经存放了值了，那么就在这个桶形成的链表上找合适的位子**

**因为桶的数量有限,默认只有16个桶，大量的元素put进来肯定有放在同一个桶的**

- 先判断entry的hash是否相等，如果不相等，那么它们的key肯定不一样。
- 如果hash相等，再通过equals判断key是否相同
- 从此处也明白必须重写HashMap的equals方法
- 若在这个链表上找到了key相同的，那么覆盖原来的值，并将原来的值返回出去【相当于告诉外界被覆盖的值是什么】

```java
for (Entry<K,V> e = table[i]; e != null; e = e.next) {  
    Object k;
    if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
        V oldValue = e.value;
        e.value = value;
        e.recordAccess(this);
        return oldValue;
    }
}
```



**5.如果没有在链表上更新值,那么用new一个entry,用头插法插入链表**

```java
modCount++;
addEntry(hash, key, value, i);							
return null;
```

```java
void addEntry(int hash, K key, V value, int bucketIndex) {
    Entry<K,V> e = table[bucketIndex];
    table[bucketIndex] = new Entry<>(hash, key, value, e); //头插法
    if (size++ >= threshold)
        resize(2 * table.length);  //jdk7默认扩容2倍
}
//Creates new entry
Entry(int h, K k, V v, Entry<K,V> n) {
    value = v;
    next = n;
    key = k;
    hash = h;
}
```



#### 扩容细节

```java
void resize(int newCapacity) {
    Entry[] oldTable = table;
    int oldCapacity = oldTable.length;
    if (oldCapacity == MAXIMUM_CAPACITY) {
        threshold = Integer.MAX_VALUE;
        return;
    }
    Entry[] newTable = new Entry[newCapacity];
    transfer(newTable);   //看这个方法是如何进行扩容的
    table = newTable;
    threshold = (int)(newCapacity * loadFactor);
}
```

```java
/**
* Transfers all entries from current table to newTable.
*/
void transfer(Entry[] newTable) {
    Entry[] src = table;
    int newCapacity = newTable.length;
    for (int j = 0; j < src.length; j++) {
        Entry<K,V> e = src[j];
        if (e != null) {
            src[j] = null;
            do {
                Entry<K,V> next = e.next;
                //根据新容量重新计算下标,可能不在同一个桶了
                int i = indexFor(e.hash, newCapacity);  
                //还是头插法
                e.next = newTable[i]; 
                newTable[i] = e;
                e = next;
            } while (e != null);
        }
    }
}
```

**扩容-重新计算桶下标**

在进行扩容时，需要把键值对重新计算桶下标，从而放到对应的桶上。在前面提到，HashMap 使用 hash%capacity 来确定桶下标。HashMap capacity 为 2 的 n 次方这一特点能够极大降低重新计算桶下标操作的复杂度。

假设原数组长度 capacity 为 16，扩容之后 new capacity 为 32：

```bash
capacity     : 00010000
new capacity : 00100000
```

对于一个 Key，它的哈希值 hash 在第 5 位：

- 为 0，那么 hash%00010000 = hash%00100000，桶位置和原来一致；
- 为 1，hash%00010000 = hash%00100000 + 16，桶位置是原位置 + 16。
- 因此扩容后，原来的map的结构可能会发生变化



#### get方法细节

```java
public V get(Object key) {
    if (key == null)
        return getForNullKey(); //从table[0]找key=null的value,找不到就返回null
    int hash = hash(key.hashCode());
    //跟之前put的分析一致,找到key就取对应的value,否则返回null
    for (Entry<K,V> e = table[indexFor(hash, table.length)];
         e != null;
         e = e.next) {
        Object k;
        //hash值一样,key未必一样,可能有哈希冲突,还得具体用equals判断
        if (e.hash == hash && ((k = e.key) == key || key.equals(k)))
            return e.value;
    }
    return null;
}
```





### HashMap的底层实现原理(jdk8)

```java
1.new HashMap(); 底层没有创建一个长度为16的数组,仅仅设置loadFactor = 0.75f
2.jdk8底层的数组是Node[],而不是Entry[]
3.首次调用put方法时,底层创建长度为16的数组
4.jdk7底层结构:数组+链表。
5.jdk8底层结构:数组+链表+红黑树
6.当数组的某一个索引位置上的元素以链表形式存在的数据个数>=8且当前数组的长度>=64时,此索引位置上的所有数据改为使用红黑树存储。
7.在链表操作的时候,jdk8采用尾插法,这与jdk7是不同的【因为采用头插法多线程下可能导致闭环】
```

![image-20210928183720267](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20210928183720267.png)







### ConcurrentHashMap

TODO



