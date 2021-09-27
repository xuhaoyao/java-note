# Java基础

## 一、数据类型

### 基本类型

- byte/8		
- char/16   
- short/16
- int/32
- float/32
- long/64
- double/64
- boolean/~

```java
java中可以调用对应包装类型的public static final int SIZE = ? 查看基本类型占用的比特数
如Integer.SIZE = 32
 
但是Boolean比较特殊，没有SIZE这个字段
    
boolean 只有两个值：true、false，可以使用 1 bit 来存储，但是具体大小没有明确规定。JVM 会在编译时期将 boolean 类型的数据转换为 int，使用 1 来表示 true，0 表示 false。JVM 支持 boolean 数组，但是是通过读写 byte 数组来实现的。

思考: C中false和true就是0和1,JVM是C写的,所以？？？
```

### 自动装箱和拆箱

[Autoboxing and Unboxing (The Java™ Tutorials > Learning the Java Language > Numbers and Strings) (oracle.com)](https://docs.oracle.com/javase/tutorial/java/data/autoboxing.html)

**自动装箱**

- 当一个简单类型的变量传递给一个方法，而这个方法的参数是包装类型的
- 将简单类型的变量赋值给包装类型

```java
List<Integer> li = new ArrayList<>();
for (int i = 1; i < 50; i += 2)
    li.add(i);
```

- 这段代码实际上转化成以下形式

```java
List<Integer> li = new ArrayList<>();
for (int i = 1; i < 50; i += 2)
    li.add(Integer.valueOf(i)); //自动装箱
```

**自动拆箱**

- 当一个包装类型的变量传递给一个方法，而这个方法的参数是简单类型的
- 将包装类型的变量赋值给简单类型

```java
public static int sumEven(List<Integer> li) {
    int sum = 0;
    for (Integer i: li)
        if (i % 2 == 0)
            sum += i;
        return sum;
}
```

- 这段代码实际转化成以下形式

```java
public static int sumEven(List<Integer> li) {
    int sum = 0;
    for (Integer i : li)
        if (i.intValue() % 2 == 0)
            sum += i.intValue();  //自动拆箱
        return sum;
}
```

总之，自动装箱和拆箱让我们开发者可以写出更加干净的代码，让程序简单易读。

| Primitive type | Wrapper class |
| -------------- | ------------- |
| boolean        | Boolean       |
| byte           | Byte          |
| char           | Character     |
| float          | Float         |
| int            | Integer       |
| long           | Long          |
| short          | Short         |
| double         | Double        |



### 缓存池

new Integer(123) 与 Integer.valueOf(123) 的区别在于：

- new Integer(123) 每次都会新建一个对象；
- Integer.valueOf(123) 会使用缓存池中的对象，多次调用会取得同一个对象的引用。

```java
Integer x = new Integer(123);
Integer y = new Integer(123);
System.out.println(x == y);    // false
Integer z = Integer.valueOf(123);
Integer k = Integer.valueOf(123);
System.out.println(z == k);   // true
```

valueOf() 方法的实现比较简单，就是先判断值是否在缓存池中，如果在的话就直接返回缓存池的内容。

```java
public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
```

在 Java 8 中，Integer 缓存池的大小默认为 -128~127。

```java
static final int low = -128;
static final int high;
static final Integer cache[];

static {
    // high value may be configured by property
    int h = 127;
    String integerCacheHighPropValue =
        sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
    if (integerCacheHighPropValue != null) {
        try {
            int i = parseInt(integerCacheHighPropValue);
            i = Math.max(i, 127);
            // Maximum array size is Integer.MAX_VALUE
            h = Math.min(i, Integer.MAX_VALUE - (-low) -1); //为了保证下面new 数组的时候不越界
        } catch( NumberFormatException nfe) {
            // If the property cannot be parsed into an int, ignore it.
        }
    }
    high = h;

    cache = new Integer[(high - low) + 1];
    int j = low;
    for(int k = 0; k < cache.length; k++)
        cache[k] = new Integer(j++);

    // range [-128, 127] must be interned (JLS7 5.1.7)
    assert IntegerCache.high >= 127;
}
```

编译器会在自动装箱过程调用 valueOf() 方法，因此多个值相同且值在缓存池范围内的 Integer 实例使用自动装箱来创建，那么就会引用相同的对象。

```java
Integer m = 123;
Integer n = 123;
System.out.println(m == n); // true
```

基本类型对应的缓冲池如下：

- boolean values true and false

  ```java
      public static Boolean valueOf(boolean b) {
          return (b ? TRUE : FALSE);
      }
  ```

- all byte values【一个byte是一个字节,8 bit，对应的数值就是0-255,正好对应了Byte缓存数组的256位】

```java
    public static Byte valueOf(byte b) {
        final int offset = 128;
        return ByteCache.cache[(int)b + offset];
    }

        static final Byte cache[] = new Byte[-(-128) + 127 + 1];

        static {
            for(int i = 0; i < cache.length; i++)
                cache[i] = new Byte((byte)(i - 128));
        }
```

- short values between -128 and 127

- int values between -128 and 127
- char in the range \u0000 to \u007F【0 - 127】

```java
static final Character cache[] = new Character[127 + 1];

static {
    for (int i = 0; i < cache.length; i++)
        cache[i] = new Character((char)i);
}

public static Character valueOf(char c) {
    if (c <= 127) { // must cache
        return CharacterCache.cache[(int)c];
    }
    return new Character(c);
}

```

在使用这些基本类型对应的包装类型时，如果该数值范围在缓冲池范围内，就可以直接使用缓冲池中的对象。

在 jdk 1.8 所有的数值类缓冲池中，Integer 的缓冲池 IntegerCache 很特殊，这个缓冲池的下界是 - 128，上界默认是 127，但是这个上界是可调的，在启动 jvm 的时候，通过 -XX:AutoBoxCacheMax=<size> 来指定这个缓冲池的大小，该选项在 JVM 初始化的时候会设定一个名为 java.lang.IntegerCache.high 系统属性，然后 IntegerCache 初始化的时候就会读取该系统属性来决定上界。



## 二、String

### 概览

String 被声明为 final，因此它不可被继承。(Integer 等包装类也不能被继承）

```java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence
    
public final class Integer 
    extends Number implements Comparable<Integer>
```

在 Java 8 中，String 内部使用 char 数组存储数据。

```java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {
    /** The value is used for character storage. */
    private final char value[];
}
```

在 Java 9 之后，String 类的实现改用 byte 数组存储字符串，同时使用 `coder` 来标识使用了哪种编码。

```java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {
    /** The value is used for character storage. */
    private final byte[] value;

    /** The identifier of the encoding used to encode the bytes in {@code value}. */
    private final byte coder;
}
```

value 数组被声明为 final，这意味着 value 数组初始化之后就不能再引用其它数组。并且 String 内部没有改变 value 数组的方法，因此可以保证 String 不可变。



### 不可变的好处【不懂】

**1. 可以缓存 hash 值**

因为 String 的 hash 值经常被使用，例如 String 用做 HashMap 的 key。不可变的特性可以使得 hash 值也不可变，因此只需要进行一次计算。

```java
//String调用hashCode()获取和初始化hash的值
private int hash; // Default to 0
public int hashCode() {
    int h = hash;
    if (h == 0 && value.length > 0) {//默认是0,所以不是空串的时候,可以执行,然后hash被赋值一次,此后这个值不再变化
        char val[] = value;

        for (int i = 0; i < value.length; i++) {
            h = 31 * h + val[i];
        }
        hash = h;
    }
    return h;
}
```

**2. String Pool 的需要**

如果一个 String 对象已经被创建过了，那么就会从 String Pool 中取得引用。只有 String 是不可变的，才可能使用 String Pool。

![img](https://camo.githubusercontent.com/a8170bdc42d86acbe28eab2dabdc32a94ecf5441fd4f5b1a539b8848df7d8c7b/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f696d6167652d32303139313231303030343133323839342e706e67)

**3. 安全性**

String 经常作为参数，String 不可变性可以保证参数不可变。例如在作为网络连接参数的情况下如果 String 是可变的，那么在网络连接过程中，String 被改变，改变 String 的那一方以为现在连接的是其它主机，而实际情况却不一定是。

**4. 线程安全**

String 不可变性天生具备线程安全，可以在多个线程中安全地使用。

[StackOverflow : String, StringBuffer, and StringBuilder](https://stackoverflow.com/questions/2971315/string-stringbuffer-and-stringbuilder)



### String Pool

字符串常量池（String Pool）保存着所有字符串字面量（literal strings），这些字面量在编译时期就确定。不仅如此，还可以使用 String 的 intern() 方法在运行过程将字符串添加到 String Pool 中。

当一个字符串调用 intern() 方法时，如果 String Pool 中已经存在一个字符串和该字符串值相等（使用 equals() 方法进行确定），那么就会返回 String Pool 中字符串的引用；否则，就会在 String Pool 中添加一个新的字符串，并返回这个新字符串的引用。

下面示例中，s1 和 s2 采用 new String() 的方式新建了两个不同字符串，而 s3 和 s4 是通过 s1.intern() 和 s2.intern() 方法取得同一个字符串引用。intern() 首先把 "aaa" 放到 String Pool 中，然后返回这个字符串引用，因此 s3 和 s4 引用的是同一个字符串。

```java
String s1 = new String("aaa");
String s2 = new String("aaa");
System.out.println(s1 == s2);           // false
String s3 = s1.intern();
System.out.println(s1 == s3);           //false
String s4 = s2.intern();
System.out.println(s3 == s4);           // true
```

如果是采用 "bbb" 这种字面量的形式创建字符串，会自动地将字符串放入 String Pool 中。

```java
String s5 = "bbb";
String s6 = "bbb";
System.out.println(s5 == s6);  // true
```

在 Java 7 之前，String Pool 被放在运行时常量池中，它属于永久代。而在 Java 7，String Pool 被移到堆中。这是因为永久代的空间有限，在大量使用字符串的场景下会导致 OutOfMemoryError 错误。



### new String("abc")

使用这种方式一共会创建两个字符串对象（前提是 String Pool 中还没有 "abc" 字符串对象）。

- "abc" 属于字符串字面量，因此编译时期会在 String Pool 中创建一个字符串对象，指向这个 "abc" 字符串字面量；
- 而使用 new 的方式会在堆中创建一个字符串对象。

```java
public class Test{
	public static void main(String[] args) {
		String s = new String("s");
	}
}
```

```bash
javac Test.java
javap -verbose Test
...
Constant pool:
   #1 = Methodref          #6.#15         // java/lang/Object."<init>":()V
   #2 = Class              #16            // java/lang/String
   #3 = String             #17            // s
	...
  #16 = Utf8               java/lang/String
  #17 = Utf8               s
	...
{
  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=3, locals=2, args_size=1
         0: new           #2                  // class java/lang/String
         3: dup
         4: ldc           #3                  // String s
         6: invokespecial #4                  // Method java/lang/String."<init>":(Ljava/lang/String;)V
         9: astore_1
        10: return
      LineNumberTable:
        line 3: 0
        line 4: 10
}
SourceFile: "Test.java"
```

在常量池中,#17存储这个字符串字面量"s",#3是String Pool的字符串对象,它指向#17这个字符串字面量.

main方法中,0行使用new #2 在堆中创建一个字符串对象，并且使用 ldc #3 将 String Pool 中的字符串对象作为 String 构造函数的参数。

以下是 String 构造函数的源码，可以看到，在将一个字符串对象作为另一个字符串对象的构造函数参数时，并不会完全复制 value 数组内容，而是都会指向同一个 value 数组。

```java
public String(String original) {
    this.value = original.value;
    this.hash = original.hash;
}
```



### String题目补充

链接：https://www.nowcoder.com/questionTerminal/e1d84672b3044a1095987df761dc5f49
来源：牛客网

已知String a="a",String b="b",String c=a+b,String d=new String("ab") 以下操作结果为true的是

- ```
  (a+b).equals(c) true		
  ```

- ```
  a+b==c			false
  ```

- ```
  c==d			false
  ```

- ```
  c.equals(d)		true
  ```

链接：https://www.nowcoder.com/questionTerminal/e1d84672b3044a1095987df761dc5f49
来源：牛客网

**1.== 和 equals():** 

  (1)“==” 用于比较基本数据类型时比较的是值，用于比较引用类型时比较的是引用指向的地址。 

  (2)Object 中的equals() 与 “==” 的作用相同，但String类重写了equals()方法，比较的是对象中的内容。



**2.String对象的两种创建方式:** 

 (1)第一种方式: String str1 = "aaa"; 是在常量池中获取对象("aaa" 属于字符串字面量，因此编译时期会在常量池中创建一个字符串对象，如果常量池中已经存在该字符串对象则直接引用)

  (2)第二种方式: String str2 = new String("aaa") ; 一共会创建两个字符串对象一个在堆中，一个在常量池中（前提是常量池中还没有 "aaa" 象）。 

   ```java
    System.out.println(str1==str2);//false
   ```



  **3.String类型的常量池比较特殊。它的主要使用方法有两种：** 

- 直接使用双引号声明出来的String对象会直接存储在常量池中。

- 如果不是用双引号声明的String对象,可以使用 String 提供的 intern 方法。 String.intern() 是一个 Native 方法，它的作用是： 如果运行时常量池中已经包含一个等于此 String 对象内容的字符串，则返回常量池中该字符串的引用； 如果没有，则在常量池中创建与此 String 内容相同的字符串，并返回常量池中创建的字符串的引用。

  ```java
  String s1 = new String("AAA");
  String s2 = s1.intern();
  String s3 = "AAA";
  System.out.println(s2);//AAA
  System.out.println(s1 == s2);//false，因为一个是堆内存中的String对象一个是常量池中的String对象，
  
  System.out.println(s2 == s3);//true， s1,s2指向常量池中的”AAA“ 
  ```



   **4字符串拼接：**  

```java
String a = "a";
String b = "b";
String str1 = "a" + "b";//常量池中的对象
String str2 = a + b; //在堆上创建的新的对象  
String str3 = "ab";//常量池中的对象
System.out.println(str1 == str2);//false
System.out.println(str1 == str3);//true 
System.out.println(str2 == str3);//false
```



## 三、运算

### 参数传递

Java 的参数是以值传递的形式传入方法中，而不是引用传递。

```java
public class Dog {

    String name;

    Dog(String name) {
        this.name = name;
    }

    String getName() {
        return this.name;
    }

    void setName(String name) {
        this.name = name;
    }

    String getObjectAddress() {
        return super.toString();
    }
}
```

以下代码中 Dog dog 的 dog 是一个指针，存储的是对象的地址。在将一个参数传入一个方法时，本质上是将对象的地址以值的方式传递到形参中。

在方法中改变对象的字段值会改变原对象该字段值，因为引用的是同一个对象。

```java
class PassByValueExample {
    public static void main(String[] args) {
        Dog dog = new Dog("A");
        func(dog);
        System.out.println(dog.getName());          // B
    }

    private static void func(Dog dog) {
        dog.setName("B");
    }
}
```

但是在方法中将指针引用了其它对象，**那么此时方法里和方法外的两个指针指向了不同的对象**，在一个指针改变其所指向对象的内容对另一个指针所指向的对象没有影响。

```java
public class PassByValueExample {
    public static void main(String[] args) {
        Dog dog = new Dog("A");
        System.out.println(dog.getObjectAddress()); // Dog@4554617c
        func(dog);
        System.out.println(dog.getObjectAddress()); // Dog@4554617c
        System.out.println(dog.getName());          // A
    }

    private static void func(Dog dog) {
        System.out.println(dog.getObjectAddress()); // Dog@4554617c
        dog = new Dog("B");
        System.out.println(dog.getObjectAddress()); // Dog@74a14482
        System.out.println(dog.getName());          // B
    }
}
```

### float 与 double

Java 不能隐式执行向下转型，因为这会使得精度降低。

1.1 字面量属于 double 类型，不能直接将 1.1 直接赋值给 float 变量，因为这是向下转型。

```java
float f = 1.1;   //Incompatible types. Found: 'double', required: 'float'
```

1.1f 字面量才是 float 类型。

```java
float f = 1.1f;
```

### 隐式类型转换

因为字面量 1 是 int 类型，它比 short 类型精度要高，因此不能隐式地将 int 类型向下转型为 short 类型。

```
short s1 = 1;
// s1 = s1 + 1;
```

但是使用 += 或者 ++ 运算符会执行隐式类型转换。

```
s1 += 1;
s1++;
```

上面的语句相当于将 s1 + 1 的计算结果进行了向下转型：

```
s1 = (short) (s1 + 1);

short x = 3;
x = (short)(x + 4.6); //7 截断
```

**it mean that in fact `i += j;` is a shortcut for something like this `i = (type of i) (i + j)`**

[StackOverflow : Why don't Java's +=, -=, *=, /= compound assignment operators require casting?](https://stackoverflow.com/questions/8710619/why-dont-javas-compound-assignment-operators-require-casting)



### switch

从 Java 7 开始，可以在 switch 条件判断语句中使用 String 对象。

```java
String s = "a";
switch (s) {
    case "a":
        System.out.println("aaa");
        break;
    case "b":
        System.out.println("bbb");
        break;
}
```

switch 不支持 long、float、double，是因为 switch 的设计初衷是对那些只有少数几个值的类型进行等值判断，如果值过于复杂，那么还是用 if 比较合适。

```java
long flag = 2L;
switch (flag){ //报错error

}
//error:Incompatible types. Found: 'long', required: 'char, byte, short, int, Character, Byte, Short, //Integer, String, or an enum'
```

[StackOverflow : Why can't your switch statement data type be long, Java?](https://stackoverflow.com/questions/2676210/why-cant-your-switch-statement-data-type-be-long-java)



## 四、关键字

### final

**1. 数据**

声明数据为常量，可以是编译时常量，也可以是在运行时被初始化后不能被改变的常量。

- 对于基本类型，final 使数值不变；
- 对于引用类型，final 使引用不变，也就不能引用其它对象，但是被引用的对象本身是可以修改的。

```java
final int x = 1;
x = 2;  // cannot assign value to final variable 'x'
final StringBuffer s = new StringBuffer();
s.append(222);  //ok
s = new StringBuffer(); //Cannot assign a value to final variable 's'
```

**2. 方法**

声明方法不能被子类重写。

private 方法隐式地被指定为 final，如果在子类中定义的方法和基类中的一个 private 方法签名相同，此时子类的方法不是重写基类方法，而是在子类中定义了一个新的方法。

**3. 类**

声明类不允许被继承。

- `final`修饰符有多种作用：
  - `final`修饰的方法可以阻止被重写；
  - `final`修饰的class可以阻止被继承；
  - `final`修饰的field必须在创建对象时初始化，随后不可修改。



### static

**1. 静态变量**

- 静态变量：又称为类变量，也就是说这个变量属于类的，类所有的实例都共享静态变量，可以直接通过类名来访问它。静态变量在内存中只存在一份。
- 实例变量：每创建一个实例就会产生一个实例变量，它与该实例同生共死。

```java
public class A {

    private int x;         // 实例变量
    private static int y;  // 静态变量

    public static void main(String[] args) {
        // int x = A.x;  // Non-static field 'x' cannot be referenced from a static context
        A a = new A();
        int x = a.x;
        int y = A.y;
    }
}
```

**2. 静态方法**

静态方法在类加载的时候就存在了，它不依赖于任何实例。**所以静态方法必须有实现**，也就是说它不能是抽象方法。

```java
public abstract class A {
    public static void func1(){
    }
    // public abstract static void func2();  // Illegal combination of modifiers: 'abstract' and 'static'
}
```

静态方法只能访问所属类的静态字段和静态方法，方法中不能有 this 和 super 关键字，**因为这两个关键字与具体对象关联**。

```java
public class A {

    private static int x;
    private int y;

    public static void func1(){
        int a = x;
        // int b = y;  // Non-static field 'y' cannot be referenced from a static context
        // int b = this.y;     // 'A.this' cannot be referenced from a static context
    }
}
```

**3. 静态语句块**

静态语句块在类初始化时运行一次。

```java
public class StringTest {
    static {
        System.out.println("静态代码块");
    }

    public static void main(String[] args) {
        StringTest test1 = new StringTest();
        StringTest test2 = new StringTest();
    }
}

```

```
静态代码块
```

**3.5代码块详解**

|      | 普通代码块                                                   | 静态代码块                                                   | 同步代码块                                                   | 构造代码块                                                   |
| ---- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 定义 | 在方法、循环、判断等语句中出现的代码块                       | 在类中定义使用static修饰的代码块                             | 可以简单地认为同步代码块是使用 synchronized 修饰的普通代码块 | 在类中定义且没有加任何修饰的代码块                           |
| 修饰 | 只能用标签修饰 { }                                           | 使用static修饰 static{ }                                     | synchronized( obj ) { }                                      | 在类中 {}                                                    |
| 位置 | 普通代码块可以出现在方法体内除"()"外的任何地方，包括 方法体，代码块中（即可以嵌套在代码块中） | 它不能出现在方法体或者代码块内 ,只能在类中定义               |                                                              | 类中                                                         |
| 执行 | 普通代码依赖方法的执行而执行，按照正常的先后顺序执行         | 在加载类时会先执行静态代码块，且只执行一次，如果有多个静态代码块则按照先后顺序执行 | 拿到锁后才执行                                               | 依赖于构造函数的调用而执行                                   |
| 作用 |                                                              | 一般用于静态变量的初始化，创建对象前的环境的加载             | 用于多线程环境的同步保护                                     | 初始化实例变量和实例环境，一般用于提取类构造函数中的公共代码 |
| 注意 |                                                              | 静态方法只能访问所属类的静态字段和静态方法，静态方法中不能有this和super | 使用不当可能会造成“死锁”                                     | 编译器在编译的时候会把构造代码块插入到每个构造函数的最前面。构造代码块随着构造函数的执行而执行。如果某个构造函数调用了其他的构造函数，那么构造代码块不会插入到该构造函数中以免构造代码块执行多次！ |

```java
public class StringTest {

    public StringTest(){
        System.out.println("构造方法执行");
    }

    static {
        System.out.println("静态代码块");
    }

    {
        System.out.println("构造代码块");
    }

    public static void main(String[] args) {
        StringTest test = new StringTest();
    }
}
```

```java
静态代码块
构造代码块
构造方法执行
```

**补：**

对于构造代码块:

- 编译器会把构造代码块插入到不含this();的构造函数中的super();后面

- super是调用父类构造函数,先有父才有子，所有是在super后面
- this()调用自身构造函数，为了保证构造代码块在类初始化的时候只执行一次，所以会插入到不含this()的构造函数中

```java
class Father{
    Father(){
        System.out.println("Father...");
    }
}

class Children extends Father{

    {
        System.out.println("构造代码块执行...");
    }

    Children(int x){
        super();
        System.out.println("x--->" + x);
    }

    Children(){
        this(5);
    }
}
public class StringTest {
    public static void main(String[] args) {
        Children children = new Children();
    }
}
Father...
构造代码块执行...
x--->5
```





**4. 静态内部类**

非静态内部类依赖于外部类的实例，也就是说需要先创建外部类实例，才能用这个实例去创建非静态内部类。而静态内部类不需要。

```java
public class OuterClass {

    class InnerClass {
    }

    static class StaticInnerClass {
    }

    public static void main(String[] args) {
        // InnerClass innerClass = new InnerClass(); // 'OuterClass.this' cannot be referenced from a static context
        OuterClass outerClass = new OuterClass();
        InnerClass innerClass = outerClass.new InnerClass();
        StaticInnerClass staticInnerClass = new StaticInnerClass();
    }
}
```

静态内部类不能访问外部类的非静态的变量和方法。



**5. 静态导包**

在使用静态变量和方法时不用再指明 ClassName，从而简化代码，但可读性大大降低。

```java
import static com.xxx.ClassName.*
```



**6. 初始化顺序**

静态变量和静态语句块优先于实例变量和普通语句块，静态变量和静态语句块的初始化顺序取决于它们在代码中的顺序。

```java
public static String staticField = "静态变量";
static {
    System.out.println("静态代码块");
}
public String field = "实例变量";
{
    System.out.println("构造代码块");
}
```

最后才是构造函数的初始化。

```java
public InitialOrderTest() {
    System.out.println("构造函数");
}
```

存在继承的情况下，初始化顺序为：

- 父类（静态变量、静态代码块）
- 子类（静态变量、静态代码块）
- 父类（实例变量）
- 父类（构造代码块，构造函数）
- 子类（实例变量）
- 子类（构造代码块，构造函数）

```java
class Father{
    public static String staticFather = "staticFather";
    static {
        System.out.println("Father--->静态代码块:" + staticFather); //1
    }
    {
        System.out.println("Father构造代码块"); //3
    }
    Father(){
        System.out.println("Father...");  //4
    }
}

class Children extends Father{
    public static String staticChild = "staticChild";
    private String test = new String("test");
    static {
        System.out.println("Children--->静态代码块:" + staticChild); //2
    }
    {
        System.out.println("test被初始化了吗? --->" + test); //true
        System.out.println("Children构造代码块"); //5
    }
    Children(){
        System.out.println("Children....");  //6
    }
}


```



## 五、Object 通用方法

### equals()

**1. 等价关系**

两个对象具有等价关系，需要满足以下五个条件：

Ⅰ 自反性

```
x.equals(x); // true
```

Ⅱ 对称性

```
x.equals(y) == y.equals(x); // true
```

Ⅲ 传递性

```
if (x.equals(y) && y.equals(z))
    x.equals(z); // true;
```

Ⅳ 一致性

多次调用 equals() 方法结果不变

```
x.equals(y) == x.equals(y); // true
```

Ⅴ 与 null 的比较

对任何不是 null 的对象 x 调用 x.equals(null) 结果都为 false

```
x.equals(null); // false;
```

**2. 等价与相等**

- 对于基本类型，== 判断两个值是否相等，基本类型没有 equals() 方法。
- 对于引用类型，== 判断两个变量是否引用同一个对象，而 equals() 判断引用的对象是否等价。

```java
Integer x = new Integer(1);
Integer y = new Integer(1);
System.out.println(x.equals(y)); // true
System.out.println(x == y);      // false
```

**3. 实现**

- 检查是否为同一个对象的引用，如果是直接返回 true；
- 检查是否是同一个类型，如果不是，直接返回 false；
- 将 Object 对象进行转型；
- 判断每个关键域是否相等。

```java
class Test{
    private Integer age;
    private String gender;
    private Date date;

    @Override
    public boolean equals(Object o) {
        if(this == o){  //检查是否为同一个对象的引用
            return true;
        }
        if(o instanceof Test){	//检查是否同一个类型
            Test obj = (Test) o;
            //判断每个field
            return Objects.equals(this.age,obj.age) &&
                    Objects.equals(this.gender,obj.gender) &&
                    Objects.equals(this.date,obj.date);
        }
        return false;
    }
}
public final class Objects{
     public static boolean equals(Object a, Object b) {
        return (a == b) || (a != null && a.equals(b));
    }
}
```



### hashCode()

hashCode() 返回哈希值，而 equals() 是用来判断两个对象是否等价。等价的两个对象散列值一定相同，但是散列值相同的两个对象不一定等价，这是因为计算哈希值具有随机性，两个值不同的对象可能计算出相同的哈希值。

**在覆盖 equals() 方法时应当总是覆盖 hashCode() 方法，保证等价的两个对象哈希值也相等。**

HashSet 和 HashMap 等集合类使用了 hashCode() 方法来计算对象应该存储的位置，因此要将对象添加到这些集合类中，需要让对应的类实现 hashCode() 方法。

下面的代码中，新建了两个等价的对象，并将它们添加到 HashSet 中。我们希望将这两个对象当成一样的，只在集合中添加一个对象。但是 EqualExample 没有实现 hashCode() 方法，因此这两个对象的哈希值是不同的，最终导致集合添加了两个等价的对象。

```java
Test test1 = new Test(2,"a");
Test test2 = new Test(2,"a");
Set<Test> set = new HashSet<>();
set.add(test1);
set.add(test2);
System.out.println(set.size());
/*
重写了equals和hashCode,set的值是1
少重写任何一个,set的值都是2
*/
```

**盲点:底层用31...**



### toString()

默认返回 **类的全类名@十六进制** 这种形式，其中 @ 后面的数值为散列码的无符号十六进制表示。

```java
//Object源码
public String toString() {
     return getClass().getName() + "@" + Integer.toHexString(hashCode());
}
```



```java
Test test1 = new Test(2,"a");
System.out.println("package com.scnu.test;");
System.out.println("test1.toString() = " + test1.toString());
System.out.println("test1.hashCode() = " + test1.hashCode());
System.out.println("460[十六进制] = " + (4 * 256 + 6 * 16) + "[十进制]");
输出结果如下:
package com.scnu.test;
test1.toString() = com.scnu.test.Test@460
test1.hashCode() = 1120
460[十六进制] = 1120[十进制]
```



### clone()

**1. cloneable**

clone() 是 Object 的 protected 方法，它不是 public，一个类不显式去重写 clone()，其它类就不能直接去调用该类实例的 clone() 方法。

```java
public class CloneExample {
    private int a;
    private int b;
}
```



```java
CloneExample e1 = new CloneExample();
// CloneExample e2 = e1.clone(); // 'clone()' has protected access in 'java.lang.Object'
```

重写 clone() 得到以下实现：

```java
public class CloneExample {
    private int a;
    private int b;

    @Override
    public CloneExample clone() throws CloneNotSupportedException {
        return (CloneExample)super.clone();
    }
}
CloneExample e1 = new CloneExample();
try {
    CloneExample e2 = e1.clone();
} catch (CloneNotSupportedException e) {
    e.printStackTrace();
}
java.lang.CloneNotSupportedException: CloneExample
```

以上抛出了 CloneNotSupportedException，这是因为 CloneExample 没有实现 Cloneable 接口。

应该注意的是，clone() 方法并不是 Cloneable 接口的方法，而是 Object 的一个 protected 方法。Cloneable 接口只是规定，如果一个类没有实现 Cloneable 接口又调用了 clone() 方法，就会抛出 CloneNotSupportedException。

```java
public class CloneExample implements Cloneable {
    private int a;
    private int b;

    @Override
    public Object clone() throws CloneNotSupportedException {
        return super.clone();
    }
}
```

**2. 浅拷贝**

拷贝对象和原始对象的引用类型引用同一个对象。

```java
public class ShallowCloneExample implements Cloneable {

    private int[] arr;

    public ShallowCloneExample() {
        arr = new int[10];
        for (int i = 0; i < arr.length; i++) {
            arr[i] = i;
        }
    }

    public void set(int index, int value) {
        arr[index] = value;
    }

    public int get(int index) {
        return arr[index];
    }

    @Override
    protected ShallowCloneExample clone() throws CloneNotSupportedException {
        return (ShallowCloneExample) super.clone();
    }
}
ShallowCloneExample e1 = new ShallowCloneExample();
ShallowCloneExample e2 = null;
try {
    e2 = e1.clone();
} catch (CloneNotSupportedException e) {
    e.printStackTrace();
}
e1.set(2, 222);
System.out.println(e2.get(2)); // 222
```

**3. 深拷贝**

拷贝对象和原始对象的引用类型引用不同对象。

```java
public class DeepCloneExample implements Cloneable {

    private int[] arr;

    public DeepCloneExample() {
        arr = new int[10];
        for (int i = 0; i < arr.length; i++) {
            arr[i] = i;
        }
    }

    public void set(int index, int value) {
        arr[index] = value;
    }

    public int get(int index) {
        return arr[index];
    }

    @Override
    protected DeepCloneExample clone() throws CloneNotSupportedException {
        DeepCloneExample result = (DeepCloneExample) super.clone();
        result.arr = new int[arr.length];
        for (int i = 0; i < arr.length; i++) {
            result.arr[i] = arr[i];
        }
        return result;
    }
}
DeepCloneExample e1 = new DeepCloneExample();
DeepCloneExample e2 = null;
try {
    e2 = e1.clone();
} catch (CloneNotSupportedException e) {
    e.printStackTrace();
}
e1.set(2, 222);
System.out.println(e2.get(2)); // 2
```

**4. clone() 的替代方案1**

使用 clone() 方法来拷贝一个对象即复杂又有风险，它会抛出异常，并且还需要类型转换。Effective Java 书上讲到，最好不要去使用 clone()，可以使用拷贝构造函数或者拷贝工厂来拷贝一个对象。

```java
public class CloneConstructorExample {

    private int[] arr;

    public CloneConstructorExample() {
        arr = new int[10];
        for (int i = 0; i < arr.length; i++) {
            arr[i] = i;
        }
    }

    public CloneConstructorExample(CloneConstructorExample original) {
        arr = new int[original.arr.length];
        for (int i = 0; i < original.arr.length; i++) {
            arr[i] = original.arr[i];
        }
    }

    public void set(int index, int value) {
        arr[index] = value;
    }

    public int get(int index) {
        return arr[index];
    }
}
CloneConstructorExample e1 = new CloneConstructorExample();
CloneConstructorExample e2 = new CloneConstructorExample(e1);
e1.set(2, 222);
System.out.println(e2.get(2)); // 2
```

**5.clone的代替方案2**

```java
ByteArrayOutputStream、ObjectOutputStream、ByteArrayInputStream、ObjectInputStream(推荐的原型模式写法)
需要实现序列化接口Serializable,不用实现克隆接口Cloneable
```

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
class Test implements Serializable{
    private Integer age;
    private String gender;
    private Date date;

    @Override
    public Object clone() {
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        try {
            ObjectOutputStream oos = new ObjectOutputStream(baos);
            oos.writeObject(this);  ////序列化时,对象的所有属性层级关系会被序列化自动处理
            oos.close();  //只关上层流

            byte[] bytes = baos.toByteArray();
            ByteArrayInputStream bais = new ByteArrayInputStream(bytes);
            ObjectInputStream ois = new ObjectInputStream(bais);
            Test clone = (Test) ois.readObject();
            return clone;
        } catch (IOException | ClassNotFoundException e) {
            e.printStackTrace();
            return null;
        }
    }

}

public class StringTest {
    public static void main(String[] args) {
        Test test = new Test(1,"a",new Date());
        Test clone = (Test) test.clone();
        clone.setDate(new Date(System.currentTimeMillis() + 1000L));
        System.out.println(test.getDate());     //Mon Sep 27 16:27:50 CST 2021
        System.out.println(clone.getDate());    //Mon Sep 27 16:27:51 CST 2021
    }
}
```



## 六、面向对象基础

面向对象程序设计(Object Oriented Programming)

- 封装：隐藏对象的属性和实现细节，仅对外提供公共访问方式，将变化隔离，提高复用性和安全性

- 继承：提高代码的复用性，是多态的前提

- 多态：针对某个类型的方法调用，其真正执行的方法取决于运行时期实际类型的方法，提高了程序的可拓展性

  - 这个拓展性，怎么说呢，例如有一个Animal类,有一个eat的方法，因为不同的动物吃食物的动作可能有所不同，我们可以让这个eat方法是抽象的，将行为固定住，新的动物只需要去实现这个eat的方法，就可以展现不同吃食物的动作

  - ```java
    Animal[] animals = new Animal[]{
      	new Cat(),
        new Dog(),
        new Fish()
    };
    for(Animal animal : animals){
        animal.eat();  //运行时动态决定要调用哪个类型的方法
    }
    ```

  - 可见，当需要新增一个动物类的时候，只需要从Animal派生，然后实现eat方法，不需要修改父类的任何代码



**1.在某些时候，就必须使用`super`**

```java
public class Main {
    public static void main(String[] args) {
        Student s = new Student("Xiao Ming", 12, 89);
    }
}

class Person {
    protected String name;
    protected int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }
}

class Student extends Person {
    protected int score;

    public Student(String name, int age, int score) {
        this.score = score;
    }
}
```

- 上面的代码会报错,大意是在`Student`的构造方法中，无法调用`Person`的构造方法。
- 因为Person重写了构造函数，那么编译器就不会生成默认构造函数，而Student调用构造函数的时候，如果**没明确指定super,那么编译器会在子类构造函数的第一行插入super()**;这里产生了错误。因为父类Person没有默认构造函数
- 所以此处就必须使用super

```java
class Student extends Person {
    protected int score;

    public Student(String name, int age, int score) {
        super(name, age); // 调用父类的构造方法Person(String, int)
        this.score = score;
    }
}
```



**2.向上转型(upcasting)**

如果一个引用变量的类型是`Student`，那么它可以指向一个`Student`类型的实例：

```
Student s = new Student();
```

如果一个引用类型的变量是`Person`，那么它可以指向一个`Person`类型的实例：

```
Person p = new Person();
```

如果`Student`是从`Person`继承下来的，那么，一个引用类型为`Person`的变量，能否指向`Student`类型的实例？

```
Person p = new Student(); // true
```

测试一下就可以发现，这种指向是允许的！

这是因为`Student`继承自`Person`，因此，它拥有`Person`的全部功能。`Person`类型的变量，如果指向`Student`类型的实例，对它进行操作，是没有问题的！

这种把一个子类类型安全地变为父类类型的赋值，被称为向上转型（upcasting）。



**3.向下转型(downcasting)**

和向上转型相反，如果把一个父类类型强制转型为子类类型，就是向下转型（downcasting）。例如：

```java
Person p1 = new Student(); // upcasting, ok
Person p2 = new Person();
Student s1 = (Student) p1; // ok
Student s2 = (Student) p2; // runtime error! ClassCastException!
```

`Person`类型`p1`实际指向`Student`实例，`Person`类型变量`p2`实际指向`Person`实例。在向下转型的时候，把`p1`转型为`Student`会成功，因为`p1`确实指向`Student`实例，把`p2`转型为`Student`会失败，因为`p2`的实际类型是`Person`，不能把父类变为子类，因为子类功能比父类多，多的功能无法凭空变出来。

因此，向下转型很可能会失败。失败的时候，Java虚拟机会报`ClassCastException`。

为了避免向下转型出错，Java提供了`instanceof`操作符，可以先判断一个实例究竟是不是某种类型

```java
Person p = new Student();
if (p instanceof Student) {
    // 只有判断成功才会向下转型:
    Student s = (Student) p; // 一定会成功
}
```

从Java 14开始，判断`instanceof`后，可以直接转型为指定变量，避免再次强制转型。

```java
public class Main {
    public static void main(String[] args) {
        Object obj = "hello";
        if (obj instanceof String s && s.equals("hello")) {
            // 可以直接使用变量s:
            System.out.println(s.toUpperCase()); //HELLO
        }
    }
}
```

**注意:**但是这里不能使用操作符||，因为操作符左边的表达式为false时不会造成短路，当执行到右边的表达式时变量还没有被定义。当然如果操作符||右边没有使用到强转后的变量是没有问题的。



### 重写与重载

**1. 重写（Override）**

存在于继承体系中，指子类实现了一个与父类在方法声明上完全相同的一个方法。

为了满足里式替换原则，重写有以下三个限制：

- 子类方法的访问权限必须大于等于父类方法；
- 子类方法的返回类型必须是父类方法返回类型或为其子类型。
- 子类方法抛出的异常类型必须是父类抛出异常类型或为其子类型。

使用 @Override 注解，可以让编译器帮忙检查是否满足上面的三个限制条件。

下面的示例中，SubClass 为 SuperClass 的子类，SubClass 重写了 SuperClass 的 func() 方法。其中：

- 子类方法访问权限为 public，大于父类的 protected。
- 子类的返回类型为 ArrayList<Integer>，是父类返回类型 List<Integer> 的子类。
- 子类抛出的异常类型为 Exception，是父类抛出异常 Throwable 的子类。
- 子类重写方法使用 @Override 注解，从而让编译器自动检查是否满足限制条件。

```
class SuperClass {
    protected List<Integer> func() throws Throwable {
        return new ArrayList<>();
    }
}

class SubClass extends SuperClass {
    @Override
    public ArrayList<Integer> func() throws Exception {
        return new ArrayList<>();
    }
}
```

在调用一个方法时，先从本类中查找看是否有对应的方法，如果没有再到父类中查看，看是否从父类继承来。否则就要对参数进行转型，转成父类之后看是否有对应的方法。总的来说，方法调用的优先级为：

- this.func(this)
- super.func(this)
- this.func(super)
- super.func(super)



```java
/*
    A
    |
    B
    |
    C
    |
    D
 */


class A {

    public void show(A obj) {
        System.out.println("A.show(A)");
    }

    public void show(C obj) {
        System.out.println("A.show(C)");
    }
}

class B extends A {

    @Override
    public void show(A obj) {
        System.out.println("B.show(A)");
    }
}

class C extends B {
}

class D extends C {
}
public static void main(String[] args) {

    A a = new A();
    B b = new B();
    C c = new C();
    D d = new D();

    // 在 A 中存在 show(A obj)，直接调用
    a.show(a); // A.show(A)
    // 在 A 中不存在 show(B obj)，将 B 转型成其父类 A
    a.show(b); // A.show(A)
    // 在 B 中存在从 A 继承来的 show(C obj)，直接调用
    b.show(c); // A.show(C)
    // 在 B 中不存在 show(D obj)，但是存在从 A 继承来的 show(C obj)，将 D 转型成其父类 C
    b.show(d); // A.show(C)

    // 引用的还是 B 对象，所以 ba 和 b 的调用结果一样
    A ba = new B();
    ba.show(c); // A.show(C)
    ba.show(d); // A.show(C)
}
```

**2. 重载（Overload）**

存在于同一个类中，指一个方法与已经存在的方法名称上相同，但是参数类型、个数、顺序至少有一个不同。

应该注意的是，返回值不同，其它都相同不算是重载。【编译报错】

```java
class OverloadingExample {
    public void show(int x) {
        System.out.println(x);
    }

    public void show(int x, String y) {
        System.out.println(x + " " + y);
    }
}
public static void main(String[] args) {
    OverloadingExample example = new OverloadingExample();
    example.show(1);
    example.show(1, "2");
}
```



### 抽象类和接口

​	TODO



## 七、反射

每个类都有一个 **Class** 对象，包含了与类有关的信息。当编译一个新类时，会产生一个同名的 .class 文件，该文件内容保存着 Class 对象。

类加载相当于 Class 对象的加载，类在第一次使用时才动态加载到 JVM 中。也可以使用 `Class.forName("com.mysql.jdbc.Driver")` 这种方式来控制类的加载，该方法会返回一个 Class 对象。

反射可以提供运行时的类信息，并且这个类可以在运行时才加载进来，甚至在编译时期该类的 .class 不存在也可以加载进来。

Class 和 java.lang.reflect 一起对反射提供了支持，java.lang.reflect 类库主要包含了以下三个类：

- **Field** ：可以使用 get() 和 set() 方法读取和修改 Field 对象关联的字段；
- **Method** ：可以使用 invoke() 方法调用与 Method 对象关联的方法；
- **Constructor** ：可以用 Constructor 的 newInstance() 创建新的对象。

**反射的优点：**

- **可扩展性** ：应用程序可以利用全限定名创建可扩展对象的实例，来使用来自外部的用户自定义类。
- **类浏览器和可视化开发环境** ：一个类浏览器需要可以枚举类的成员。可视化开发环境（如 IDE）可以从利用反射中可用的类型信息中受益，以帮助程序员编写正确的代码。
- **调试器和测试工具** ： 调试器需要能够检查一个类里的私有成员。测试工具可以利用反射来自动地调用类里定义的可被发现的 API 定义，以确保一组测试中有较高的代码覆盖率。

**反射的缺点：**

尽管反射非常强大，但也不能滥用。如果一个功能可以不用反射完成，那么最好就不用。在我们使用反射技术时，下面几条内容应该牢记于心。

- **性能开销** ：反射涉及了动态类型的解析，所以 JVM 无法对这些代码进行优化。因此，反射操作的效率要比那些非反射操作低得多。我们应该避免在经常被执行的代码或对性能要求很高的程序中使用反射。
- **安全限制** ：使用反射技术要求程序必须在一个没有安全限制的环境中运行。如果一个程序必须在有安全限制的环境中运行，如 Applet，那么这就是个问题了。
- **内部暴露** ：由于反射允许代码执行一些在正常情况下不被允许的操作（比如访问私有的属性和方法），所以使用反射可能会导致意料之外的副作用，这可能导致代码功能失调并破坏可移植性。反射代码破坏了抽象性，因此当平台发生改变的时候，代码的行为就有可能也随着变化。
- [深入解析 Java 反射（1）- 基础](http://www.sczyh30.com/posts/Java/java-reflection-1/)



## 八、异常

参考[Java提高篇——Java 异常处理 - 萌小Q - 博客园 (cnblogs.com)](https://www.cnblogs.com/Qian123/p/5715402.html)

Java把异常当作对象来处理，并定义一个基类`java.lang.Throwable`作为所有异常的超类。

在Java API中已经定义了许多异常类，这些异常类分为两大类，**错误`Error`和异常`Exception`**。

Java异常层次结构图如下图所示：

![img](https://images2015.cnblogs.com/blog/690102/201607/690102-20160728164909622-1770558953.png)

从图中可以看出所有异常类型都是内置类`Throwable`的子类，因而`Throwable`在异常类的层次结构的顶层。

接下来`Throwable`分成了两个不同的分支，**一个分支是`Error`，它表示不希望被程序捕获或者是程序无法处理的错误**。**另一个分支是`Exception`，它表示用户程序可能捕捉的异常情况或者说是程序可以处理的异常**。其中异常类`Exception`又分为运行时异常(`RuntimeException`)和非运行时异常。

Java异常又可以分为不受检查异常（`Unchecked Exception`）和检查异常（`Checked Exception`）。

- `RuntimeException`及其子类和`Error `是`Unchecked Exception`
- `RuntimeException`之外的异常我们统称为非运行时异常，类型上属于`Exception`类及其子类，从程序语法角度讲是必须进行处理的异常，如果不处理，程序就不能编译通过。如`IOException`、`SQLException`等



对于`运行时异常`、`错误`和`检查异常`，Java技术所要求的异常处理方式有所不同。

由于运行时异常及其子类的不可查性，为了更合理、更容易地实现应用程序，Java规定，**运行时异常将由Java运行时系统自动抛出，允许应用程序忽略运行时异常**。

对于方法运行中可能出现的`Error`，当运行方法不欲捕捉时，Java允许该方法不做任何抛出声明。因为，大多数`Error`异常属于永远不能被允许发生的状况，也属于合理的应用程序不该捕捉的异常。

对于所有的检查异常，Java规定：一个方法必须捕捉，或者声明抛出方法之外。也就是说，当一个方法选择不捕捉检查异常时，它必须声明将抛出异常。



**1.`Throws`抛出异常的规则**：

- 如果是不受检查异常（`unchecked exception`），即`Error`、`RuntimeException`或它们的子类，那么可以不使用`throws`关键字来声明要抛出的异常，编译仍能顺利通过，但在运行时会被系统抛出。
- 必须声明方法可抛出的任何检查异常（`checked exception`）。即如果一个方法可能出现受可查异常，要么用`try-catch`语句捕获，要么用`throws`子句声明将它抛出，否则会导致编译错误
- 仅当抛出了异常，该方法的调用者才必须处理或者重新抛出该异常。当方法的调用者无力处理该异常的时候，应该继续抛出，而不是囫囵吞枣。
- 调用方法必须遵循任何可查异常的处理和声明规则。若覆盖一个方法，则不能声明与覆盖方法不同的异常。声明的任何异常必须是被覆盖方法所声明异常的同类或子类。
- 覆盖的方法可以随意地抛出unchecked exception

**2.try{} 里有一个 return 语句，那么紧跟在这个 try 后的 finally{} 里的 code 会不会被执行，什么时候被执行，在 return 前还是后?**

答案：会执行，在方法返回调用者前执行。

- 在return的时候,程序会保存return时候的返回值

- 基本类型就是保存值，引用类型保存的是内存地址

- ```bash
  1.执行：expression，计算该表达式，结果保存在操作数栈顶；
  2.执行：操作数栈顶值（expression的结果）复制到局部变量区作为返回值；
  3.执行：finally语句块中的代码；
  4.执行：将第2步复制到局部变量区的返回值又复制回操作数栈顶；
  5.执行：return指令，返回操作数栈顶的值；
  ```

- 见下面的例子

```java
@Data
@AllArgsConstructor
class TestObj{
    private String test;
}

@Data
class Obj{
    private Integer x = 10;

    private String y = "yy";

    private TestObj t = new TestObj("123");

    private List<Integer> list = new ArrayList<>();
}

public class StringTest {

    public static Integer fun1(){
        Integer a = 1;
        try {
            a = 2;
            return a;
        }finally {
            a = 3;
        }
    }

    public static Integer fun2(){
        Obj obj = new Obj();
        try {
            obj.setX(100);
            return obj.getX();
        }
        finally {
            obj.setX(999);
        }
    }

    public static String fun3(){
        Obj obj = new Obj();
        try {
            return obj.getY();
        }
        finally {
            obj.setY("xxzxzx");
        }
    }

    public static TestObj fun4(){
        Obj obj = new Obj();
        try {
            return obj.getT();
        }
        finally {
            obj.setT(new TestObj("dsdsdss"));
        }
    }

    public static StringBuffer fun5(){
        StringBuffer s = new StringBuffer("hello");
        try {
            return s;
        }finally {
            s.append(" world");
        }
    }

    public static List<Integer> fun6(){
        Obj obj = new Obj();
        try {
            return obj.getList();
        }
        finally {
            List<Integer> list = obj.getList();
            list.add(66);
        }
    }


    public static void main(String[] args) {
        System.out.println(StringTest.fun1());  //2
        System.out.println(StringTest.fun2());  //10
        System.out.println(StringTest.fun3());  //yy
        System.out.println(StringTest.fun4());  //TestObj(test=123)
        System.out.println(StringTest.fun5());  //hello world
        System.out.println(StringTest.fun6());  //[66]
    }
}
```

- finally代码块什么时候不执行？
- **Note:** If the JVM exits while the `try` or `catch` code is being executed, then the `finally` block may not execute. Likewise, if the thread executing the `try` or `catch` code is interrupted or killed, the `finally` block may not execute even though the application as a whole continues.
- [The finally Block (The Java™ Tutorials > Essential Java Classes > Exceptions) (oracle.com)](https://docs.oracle.com/javase/tutorial/essential/exceptions/finally.html)



![img](https://images2015.cnblogs.com/blog/690102/201607/690102-20160729105042231-937633681.jpg)

![img](https://images2015.cnblogs.com/blog/690102/201607/690102-20160729100445981-600538221.jpg)