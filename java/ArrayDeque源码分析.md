## ArrayDeque源码分析

### null元素禁止,用它来代替栈和队列

```bash
Null elements are prohibited. This class is likely to be faster than Stack when used as a stack, and faster than LinkedList when used as a queue.
```



### 默认初始化容量是16

```java
    
	transient Object[] elements; // non-private to simplify nested class access	

	transient int head;  //弹出的元素下标

	transient int tail;  //插入元素下标

	public ArrayDeque() {
        elements = new Object[16];
    }
```



### 容量必须是2的幂,向上取整

可以看到，容量最大是2^30，最小是2^3

```java
    private static final int MIN_INITIAL_CAPACITY = 8;

	private static int calculateSize(int numElements) {
        int initialCapacity = MIN_INITIAL_CAPACITY;
        // Find the best power of two to hold elements.
        // Tests "<=" because arrays aren't kept full.
        if (numElements >= initialCapacity) {
            initialCapacity = numElements;
            initialCapacity |= (initialCapacity >>>  1);
            initialCapacity |= (initialCapacity >>>  2);
            initialCapacity |= (initialCapacity >>>  4);
            initialCapacity |= (initialCapacity >>>  8);
            initialCapacity |= (initialCapacity >>> 16);
            initialCapacity++;

            if (initialCapacity < 0)   // Too many elements, must back off
                initialCapacity >>>= 1;// Good luck allocating 2 ^ 30 elements
        }
        return initialCapacity;
    }
```

```java
求一个数的掩码最快的方法
对于 10000000 这样的数要扩展成 11111111
mask |= mask >> 1    11000000
mask |= mask >> 2    11110000
mask |= mask >> 4    11111111
```



### addLast

```java
public void addLast(E e) {
    if (e == null)
        throw new NullPointerException();
    elements[tail] = e;
    if ( (tail = (tail + 1) & (elements.length - 1)) == head)
        doubleCapacity();
}
```

由于容量是2的幂，因此`elements.length - 1`就发挥作用了,用来快速判断`tail == head`

`(tail + 1) & (elements.length - 1)`:就相当于取余操作，保证不会有数组越界

- 1τ=10ns

- 取余要14τ	
- 而逻辑运算只需要2τ	



### doubleCapacity(每次扩容一倍)

```java
private void doubleCapacity() {
    assert head == tail;
    int p = head;
    int n = elements.length;
    int r = n - p; // number of elements to the right of p
    int newCapacity = n << 1;
    if (newCapacity < 0)
        throw new IllegalStateException("Sorry, deque too big");
    Object[] a = new Object[newCapacity];
    System.arraycopy(elements, p, a, 0, r);	//将目前右边元素(head开始的部分)拷贝到扩容数组的最左边
    System.arraycopy(elements, 0, a, r, p);	//将目前左边元素(tail之前的部分)拷贝到扩容数组的右边
    elements = a;
    head = 0;
    tail = n;
}
```

扩容过程可以看下面这张图

![image-20211217001658078](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20211217001658078.png)



### addFirst

```java
    public void addFirst(E e) {
        if (e == null)
            throw new NullPointerException();
        elements[head = (head - 1) & (elements.length - 1)] = e;
        if (head == tail)
            doubleCapacity();
    }
```

head是要弹出去的元素,head - 1就是可以插入的

对于head - 1 < 0的情况： `-1 & 7 = 7`     ->    ` 0xffffffff & 0x00000007 = 7`



### pollFirst

从这得知head是要弹出去的元素下标

```java
    public E pollFirst() {
        int h = head;
        @SuppressWarnings("unchecked")
        E result = (E) elements[h];
        // Element is null if deque empty
        if (result == null)
            return null;
        elements[h] = null;     // Must null out slot
        head = (h + 1) & (elements.length - 1);
        return result;
    }
```



### pollLast

```java
    public E pollLast() {
        int t = (tail - 1) & (elements.length - 1);
        @SuppressWarnings("unchecked")
        E result = (E) elements[t];
        if (result == null)
            return null;
        elements[t] = null;
        tail = t;
        return result;
    }
```





