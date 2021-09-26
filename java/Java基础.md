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

