# Java面试题

## Java语言的三大特性

封装、继承、多态。

封装：隐藏对象的属性和实现细节，仅对外提供公共访问方式，我们只需要调用方法去获取我们想要的结果即可，这个封装的对象内部是怎么变化的，我们并不关心，**将变化隔离，提高复用性和安全性**

继承：extends，继承父类，提高代码的复用性。子类拥有父类的所有属性和方法，但是父类中的私有方法和私有属性子类无法访问，只是拥有。

多态：针对某个类的方法调用，我们只有在运行的时候才知道它实际调用的是哪个方法。提高了程序的可拓展性。

> 如何提高扩展性？举个例子
>
> 比如说有一个动物类Animal，动物是可以吃东西的嘛，不同的动物吃东西的方式可能有所不同，而且动物本身就是抽象的，我们可以让动物类有一个抽象方法eat，将吃的这个行为固定住，具体的动物怎么吃，留给具体的子类去实现。
>
> ```java
> Animal[] animals = new Animal[]{
>   	new Cat(),
>     new Dog(),
>     new Fish()
> };
> for(Animal animal : animals){
>     animal.eat();  //运行时动态决定要调用哪个类型的方法
> }
> ```
>
> **可见，当需要新增一个动物类的时候，只需要从Animal派生，然后实现eat方法，不需要修改父类的任何代码，这就提高了代码的可扩展性**



## 重载和重写的区别

重载是发生在同一个类中，具有相同的方法名，但是参数列表不相同，返回值类型和抛出的异常列表可以不同。

- 典型的，构造方法重载，有无参构造和有参构造
- **重载在编译时期就已经确定（静态分派）**

重写是发生在子类继承父类的时候，指子类实现了一个与父类在方法声明上完全相同的一个方法。

- 重写的时候，子类方法的访问权限 >= 父类方法的访问权限，比如父类是protected，子类就应该是protected或者public
- 重写的时候，子类方法的返回值 必须是父类方法返回值类型或者子类型，比如父类是List,子类就应该是List或者ArrayList
- 重写的时候，子类方法抛出的异常类型必须是父类抛出异常类型或为其子类型。比如父类是Exception，子类应该是Exception或者RuntimeException
- **重写在运行期间才能确定（动态分派）**

> 重载：编译器在重载时是通过参数的静态类型而不是实际类型作为判断依据的。由于静态类型在编译时已知，所以编译阶段Javac编译器就根据参数的静态类型决定了会使用哪个重载版本。
>
> 重写：Java中默认方法是虚方法（没有被final修饰的），虚方法调用指令invokevirtual会找到运行期间对象的实际类型，根据实际类型来选择方法版本



## 接口和抽象类的区别

- 接口，就是interfact，抽象类的话，就是abstract class

- 相同点：

  - 接口和抽象类都不能被实例化

  - 接口的实现类和抽象类的实现类都需要实现了**所有抽象方法**才能实例化。

  - > 抽象类有点特殊，若一个类是abstract class的，但是它的方法没有abstract，也可以是一个抽象类，但是new的时候必须实现任意一个方法才能实例化，ClassLoader就是这样。

- 不同点：
  - 接口只有定义，不能有方法实现，不过jdk1.8中可以定义default，让接口有默认实现方法。
  - 实现接口的关键字为implements，实现抽象类的关键字为extends
  - 一个类可以实现多个接口，但只能继承一个类，但是接口可以做到继承多个接口
  - **从设计层面上看，我觉得接口中定义的方法，就是一套规范，接口是对行为的抽象，是一种行为的规范。而抽象类，更多的是一种模板设计，将一个类的行为在高层固定住，行为已经固定好了，具体怎么实现这些行为，留给子类去完成。**
  - 接口方法是public abstract的，接口的字段是public static final的，而抽象类中可以有private，protected，public这些修饰符



> 为什么要增加一个default字段？
>
> 主要为了兼容，高版本的JDK必须向下兼容低版本的Class文件，即用JDK1.8的虚拟机去运行1.7JDK编译后的class文件，也应该能够运行。default关键字就起到了作用，若没有default关键字，Java容器类的Collection接口在1.8新增了一个stream，那么所有它的子类都必须实现这个stream才能通过编译，而有了default关键字，子类就可以不去实现了。
>
> 这样一来，1.8版本以下的class文件，由于没有stream方法，但是JDK1.8的Collection默认实现了stream，因此1.8版本以下的容器了即使没有stream方法，编译也不会有冲突。



## Java中的内部类

内部类有四种，分别是静态内部类，成员内部类，局部内部类，匿名内部类。

**静态内部类**：在一个类里面定义的类static class，若内部类不需要用到外部类的属性，都定义成静态的比较好，比如HashMap里面的Node节点，就是静态内部类，**因为这样更节省资源，减少内部类其中的一个指向外部类的引用**。

**成员内部类**：就是在一个类里面再定义一个类，没有static修饰，它能够访问到外部类的字段和方法

```java
class MyMap{
    private int c = 0;
    
    public void put(){
    }
    
    static class NodeA{
        public void f(){
            System.out.println(MyMap.this.c); //'MyMap.this' cannot be referenced from a static context
            MyMap.this.put();                 //'MyMap.this' cannot be referenced from a static context
        }
    }
    class NodeB{
        public void f(){
            System.out.println(MyMap.this.c);
            MyMap.this.put();
        }
    }
}

```

**局部内部类**：定义在方法中的类叫做局部内部类。

```java
class MyClass{
    
    public void func(){
        class testClass{  //局部内部类
            public void f(){
                System.out.println("f...");
            }
        }
        
        testClass testClass = new testClass();
        testClass.f();
    }
    
}
```

**匿名内部类**：没有名字，必须继承或者实现一个接口。

```java
    public static void main(String[] args) throws IOException {
        class Node{
            int val;
        }
        Node[] nodes = new Node[2];
        Arrays.sort(nodes, new Comparator<Node>() {  //这个匿名内部类实现了Comparator的接口
            @Override
            public int compare(Node o1, Node o2) {
                return Integer.compare(o1.val,o2.val);
            }
        });

    }

	public static void main(String[] args) throws IOException {
        class Node{
            int val;
        }
        Node[] nodes = new Node[2];
        //函数式接口@FunctionalInterface可以用lambda表达式简写
        Arrays.sort(nodes,(a,b) -> Integer.compare(a.val,b.val));

    }
```



### 为什么使用内部类？

1、实现隐藏。就比如一个类，它需要依赖于某个数据结构，同时这个数据结构只会在这个类被使用到，这时就可以用内部类来定义这个数据结构，就比如HashMap里面的Node节点，我们外界无法访问它，它只在HashMap里面使用。

2、间接实现多重继承。

```java
//ExampleOne.java
public class ExampleOne {
    public String name() {
        return "inner";
    }
}
//ExampleTwo.java
public class ExampleTwo {
    public int age() {
        return 25;
    }
}
```

```java
//MainExample.java
public class MainExample {

   /**
    * 内部类1继承ExampleOne
    */
   private class InnerOne extends ExampleOne {
       public String name() {
           return super.name();
       }
   }
   /**
    * 内部类2继承ExampleTwo
    */
   private class InnerTwo extends ExampleTwo {
       public int age() {
           return super.age();
       }
   }
   public String name() {
       return new InnerOne().name();
   }

   public int age() {
       return new InnerTwo().age();
   }

   public static void main(String[] args) {

       MainExample mi = new MainExample();

       System.out.println("姓名:" + mi.name());

       System.out.println("年龄:" + mi.age());
   }
}

```

**MainExample里面分别实现了两个内部类 InnerOne,和InnerTwo ，InnerOne类又继承了ExampleOne，InnerTwo继承了ExampleTwo，这样MainExample就拥有了ExampleOne和ExampleTwo的方法和属性，也就间接地实现了多继承。**



## final关键字的作用

final可以修饰类，方法，字段

- final修饰类的时候，这个类不能被继承，一般将一个类定义为final，就说明这个类的功能很完善了，不需要、也不希望子类覆盖它，就比如String类
- final修饰方法的时候，就说明这个方法不能再被重写了。
- final修饰属性的时候，它要么直接赋值，要么就得在构造函数完成初始化。若是基本类型（int,char）这些，赋值完毕之后值不再更改，若是引用类型，赋值完毕，这个引用就一直指向堆的那个对象，引用不变，但是引用的对象内部可以发生变化。



## static关键字的作用

static可以定义一个静态内部类，修饰字段或者方法

- static定义静态内部类，当我们在一个类里面想要实现某种数据结构，而这种数据结构不会访问这个类，那么就可以定义静态内部类来节省空间，比如HashMap里面的Node节点
- **静态导包**，在使用某个类的静态变量或者静态方法的时候就不再需要指定类名，但是可读性很低。
  - `import static java.lang.Integer.*;`
  - `int a = valueOf("22");`  直接valueOf即可
- **静态变量：**修饰的字段就是一个静态变量，又叫做类变量，一个类就一份，每个类的实例都共享静态变量，节省了空间。
- **静态方法：**调用某个类的静态方法，不需要实例化这个类，直接`类名.method()`,静态方法只能访问所属类的静态字段和静态方法，方法中不能有 this 和 super 关键字，**因为这两个关键字与具体对象关联**。



## String，StringBuilder和StringBuffer的区别

- String是一个不可变的对象，底层用private final char[]数组存储，它的任何方法都不会修改底层的数组，所以是一个不可变的对象。
- 而StringBuilder和StringBuffer继承自AbstractStringBuilder,底层的char数组没有用final修饰，而且也通过了append等方法可以改变底层的数组，因此他俩是可变的
- 线程安全性方法，String做到了安全发布，所以它本身就是线程安全的，而StringBuilder不是线程安全的，StringBuffer因为对于更改的方法都加了synchronized，因此它是线程安全的。



## ==和equals的区别

- ==就是判断两个基本类型是否相同或者两个对象的地址是否相同，基本类型比较的是值，引用类型比较的是内存地址。
- equals的话
  - 若对象没有重写equals，那么比较的就是内存地址，这时候和==的作用就一样了
  - 若正确重写了equals，那么比较的就是对象的内容是否一致



## hashCode()和equals()

**在覆盖 equals() 方法时应当总是覆盖 hashCode() 方法，保证等价的两个对象哈希值也相等。**

equals的话

- 若对象没有重写equals，那么比较的就是内存地址，这时候和==的作用就一样了
- 若正确重写了equals，那么比较的就是对象的内容是否一致

hashCode的话

- 若没有重写，那么返回的就是对象内存地址的整数值，不同对象的内存地址都不相同
- **因此，若我们想要对象内容相同的,hashCode也相同，那么就要重写hashCode方法**

**重写情况下，内容相同的两个对象,equals返回true，那么它们的hashCode也是相同的，而两个对象hashCode相同，内容未必相同，因为可能有hash冲突**

为什么重写了equals也一定要重写hashCode呢？

**其实要看情况而定，若这个类，它不会保存在hashMap这类需要用到hash值的容器里面，就不需要重写。**

- 一个对象怎么放到hashMap里面的呢？

  - 1、计算hash值（这里就会调用hashCode方法，显然，想要内容相同的对象，那么就需要重写hashCode了）

  - 2、取余找到存放的桶的下标

  - 3、若这个桶上有存放的元素了，它是怎么放到桶里面去的？

    - 先看两个key的hash值是否相同

    - 再看两个key是否是指向同一个引用

    - 再调用equals方法

    - ```java
      if (p.hash == hash &&
          ((k = p.key) == key || (key != null && key.equals(k))))
      ```

      

## Java异常体系

![image-20220314225238904](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220314225238904.png)

Error：运行时错误，如果程序在启动的时候Error，那么启动失败，如果程序在运行过程中出现Error，系统会退出程序。Error在程序运行过程中不能动态处理

Exception：Java运行时异常，可以被try Catch捕捉到，不致于会退出程序。

- 可以分为运行时异常（RuntimeException）和受检异常
  - 受检异常一定要try Catch处理，或者往上抛，否则编译通不过。
    - 常见的有`IOException,SQLException`
  - 而运行时异常没那么严格，若本方法不处理，可以留给上层去捕获，编译器不会报错。
    - 常见的有`NullPointerException,ArrayIndexOutOfBoundException,ClassCastException`



## 深拷贝和浅拷贝

浅拷贝：

- Java是值传递，clone拷贝的全都是值，引用类型的话也是拷贝这个引用的值

深拷贝：

- 需要自己重写clone，对于那些引用类型，也要重写引用类型的clone方法

举个例子：

类A里面有一个数组字段`int[] nums`，提供一个方法allAdd，让nums的所有数字+1

- 浅拷贝，若克隆对象调用allAdd，那么类A和克隆对象的nums的所有数字都加了1
- 深拷贝，若克隆对象调用allAdd，那么只有克隆对象的nums的所有数字加了1