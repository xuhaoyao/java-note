# spring到底为什么要用三级缓存

文章出处：[Spring系列第56篇：一文搞懂spring到底为什么要用三级缓存？？ - 云+社区 - 腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/1796740)



问题：**spring 中为什么需要用三级缓存来解决这个问题？用二级缓存可以么？**

先给出答案：**不可用**。

这里先声明下：

**本文未指明 bean scope 默认情况下，所有 bean 都是单例的，即 scope 是 singleton，即下面所有问题都是在单例的情况下分析的。**

**代码中注释很详细，一定要注意多看代码中的注释。**

>  **只有单例的Bean会通过三级缓存提前暴露来解决循环依赖的问题，而非单例的Bean，每次从容器中获取都是一个新的对象，都会重新创建，所有非单例的Bean是没有缓存的，不会将其放到三级缓存中**

## **1、循环依赖相关问题**

1、什么是循环依赖？

2、循环依赖的注入对象的 2 种方式：构造器的方式、setter 的方式

3、构造器的方式详解

4、spring 是如何知道有循环依赖的？

5、setter 方式详解

6、需注意循环依赖注入的是半成品

7、为什么必须用三级缓存？

## **2、什么是循环依赖？**

A 依赖于 B，B 依赖于 A，比如下面代码

```javascript
public class A {
    private B b;
}

public class B {
    private A a;
}
```

## **3、循环依赖注入对象的 2 种方式**

### **3.1、构造器的方式**

通过构造器相互注入对方，代码如下

```javascript
public class A {
    private B b;

    public A(B b) {
        this.b = b;
    }
}

public class B {
    private A a;

    public B(A a) {
        this.a = a;
    }
}
```

### **3.2、setter 的方式**

通过 setter 方法注入对方，代码如下

```javascript
public class A {
    private B b;

    public B getB() {
        return b;
    }

    public void setB(B b) {
        this.b = b;
    }
}

public class B {
    private A a;

    public A getA() {
        return a;
    }

    public void setA(A a) {
        this.a = a;
    }
}
```

## **4、构造器的方式详解**

### **4.1、构造器的方式知识点**

1、构造器的方式如何注入？

2、循环依赖，构造器的方式，spring 的处理过程是什么样的？

3、循环依赖构造器的方式案例代码解析

### **4.2、构造器的方式如何注入？**

再来看一下下面这 2 个类，相互依赖，通过构造器的方式相互注入对方。

```javascript
public class A {
    private B b;

    public A(B b) {
        this.b = b;
    }
}

public class B {
    private A a;

    public B(A a) {
        this.a = a;
    }
}
```

大家来思考一个问题：**2 个类都只能创建一个对象，大家试试着用硬编码的方式看看可以创建这 2 个类的对象么？**

我想大家一眼就看出来了，无法创建。

创建 A 的时候需要先有 B，而创建 B 的时候需要先有 A，导致无法创建成功。

### **4.3、循环依赖，构造器的方式，spring 的处理过程是什么样的？**

spring 在创建 bean 之前，会将当前正在创建的 bean 名称放在一个列表中，这个列表我们就叫做 **singletonsCurrentlyInCreation**，用来记录正在创建中的 bean 名称列表，创建完毕之后，会将其从 singletonsCurrentlyInCreation 列表中移除，并且会将创建好的 bean 放到另外一个单例列表中，这个列表叫做 singletonObjects，下面看一下这两个集合的代码，如下：

```javascript
代码位于org.springframework.beans.factory.support.DefaultSingletonBeanRegistry类中

//用来存放正在创建中的bean名称列表
private final Set<String> singletonsCurrentlyInCreation =
   Collections.newSetFromMap(new ConcurrentHashMap<>(16));

//用来存放已经创建好的单例bean，key为bean名称，value为bean的实例
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);
```

下面我们来看下面 2 个 bean 的创建过程

```javascript
@Compontent
public class A {
    private B b;

    public A(B b) {
        this.b = b;
    }
}

@Compontent
public class B {
    private A a;

    public B(A a) {
        this.a = a;
    }
}
```

过程如下

```javascript
1、从singletonObjects查看是否有a，此时没有
2、准备创建a
3、判断a是否在singletonsCurrentlyInCreation列表，此时明显不在，则将a加入singletonsCurrentlyInCreation列表
4、调用a的构造器A(B b)创建A，这里真正利用反射创建一个Bean
5、spring发现A的构造器需要用到b
6、则向spring容器查找b，从singletonObjects查看是否有b，此时没有
7、spring准备创建b
8、判断b是否在singletonsCurrentlyInCreation列表，此时明显不在，则将b加入singletonsCurrentlyInCreation列表
9、调用b的构造器B(A a)创建b【反射】
10、spring发现B的构造器需要用到a，则向spring容器查找a
11、则向spring容器查找a，从singletonObjects查看是否有a，此时没有
12、准备创建a
13、判断a是否在singletonsCurrentlyInCreation列表，上面第3步中a被放到了这个列表，此时a在这个列表中，走到这里了，说明a已经存在创建列表中了，此时程序又来创建a，说明这么一直走下去会死循环，此时spring会弹出异常，终止bean的创建操作。
	异常名称:BeanCurrentlyInCreationException
```

### **4.4、通过这个过程，我们得到了 2 个结论**

1、循环依赖如果是构造器的方式，bean 无法创建成功，这个前提是 bean 都是单例的，bean 如果是多例的，大家自己可以分析分析。

2、**spring 是通过 singletonsCurrentlyInCreation 这个列表来发现循环依赖的，这个列表会记录创建中的 bean，当发现 bean 在这个列表中存在了，说明有循环依赖，并且这个循环依赖是无法继续走下去的，如果继续走下去，会进入死循环，此时 spring 会抛出异常让系统终止。**

判断循环依赖的源码在下面这个位置，`singletonsCurrentlyInCreation`是 Set 类型的，Set 的 add 方法返回 false，说明被 add 的元素在 Set 中已经存在了，然后会抛出循环依赖的异常。

```javascript
org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#beforeSingletonCreation

private final Set<String> singletonsCurrentlyInCreation =
  Collections.newSetFromMap(new ConcurrentHashMap<>(16));

protected void beforeSingletonCreation(String beanName) {
    //bean名称已经存在创建列表中，则抛出循环依赖异常
    if (!this.inCreationCheckExclusions.contains(beanName) && !this.singletonsCurrentlyInCreation.add(beanName)) {
        //抛出循环依赖异常
        throw new BeanCurrentlyInCreationException(beanName);
    }
}

//循环依赖异常
public BeanCurrentlyInCreationException(String beanName) {
    super(beanName,
          "Requested bean is currently in creation: Is there an unresolvable circular reference?");
}
```

### **4.5、spring 构造器循环依赖案例**

创建类 A

```javascript
package com.javacode2018.cycledependency.demo1;

import org.springframework.stereotype.Component;

@Component
public class A {
    private B b;

    public A(B b) {
        this.b = b;
    }
}
```

创建类 B

```javascript
package com.javacode2018.cycledependency.demo1;

import org.springframework.stereotype.Component;

@Component
public class B {
    private A a;

    public B(A a) {
        this.a = a;
    }
}
```

启动类

```javascript
package com.javacode2018.cycledependency.demo1;

import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;

@Configuration
@ComponentScan
public class MainConfig {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
        context.register(MainConfig.class);
        //刷新容器上下文，触发单例bean创建
        context.refresh();
        //关闭上下文
        context.close();
    }
}
```

运行上面的 main 方法，产生了异常，部分异常信息如下，说明创建 bean`a`的时候出现了循环依赖，导致创建 bean 无法继续进行，以后大家遇到这个错误了，应该可以很快定位到问题了。

```javascript
Caused by: org.springframework.beans.factory.BeanCurrentlyInCreationException: Error creating bean with name 'a': Requested bean is currently in creation: Is there an unresolvable circular reference?
 at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.beforeSingletonCreation(DefaultSingletonBeanRegistry.java:347)
```

## **5、setter 方式详解**

再来看看 setter 的 2 个类的源码

```javascript
public class A {
    private B b;

    public B getB() {
        return b;
    }

    public void setB(B b) {
        this.b = b;
    }
}

public class B {
    private A a;

    public A getA() {
        return a;
    }

    public void setA(A a) {
        this.a = a;
    }
}
```

大家试试通过硬编码的方式来相互注入，很简单吧，如下面这样

```javascript
A a = new A();
B b = new B();
a.setB(b);
b.setA(a);
```

咱们通过硬编码的方式可以搞成功的，spring 肯定也可以搞成功，确实，setter 循环依赖，spring 可以正常执行。

下面来看 spring 中 setter 循环依赖注入的流程。

## **6、spring 中 setter 循环依赖注入流程**

spring 在创建单例 bean 的过程中，会用到三级缓存，所以需要先了解三级缓存。

### **6.1、三级缓存是哪三级？**

spring 中使用了 3 个 map 来作为三级缓存，每一级对应一个 map

| 第几级缓存 | 对应的 map                                       | 说明                                                         |
| :--------- | :----------------------------------------------- | :----------------------------------------------------------- |
| 第 1 级    | Map<String, Object> singletonObjects             | 用来存放已经完全创建好的单例 beanbeanName->bean 实例         |
| 第 2 级    | Map<String, Object> earlySingletonObjects        | 用来存放早期的 beanbeanName->bean 实例                       |
| 第 3 级    | Map<String, ObjectFactory<?>> singletonFactories | 用来存放单例 bean 的 ObjectFactorybeanName->ObjectFactory 实例 |

这 3 个 map 的源码位于`org.springframework.beans.factory.support.DefaultSingletonBeanRegistry`类中。

![image-20220127001825502](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220127001825502.png)

### **6.2、单例 bean 创建过程源码解析**

代码入口

```java
org.springframework.beans.factory.support.AbstractBeanFactory#doGetBean
```

#### **step1：doGetBean**

如下，这个方法首先会调用 getSingleton 获取 bean，如果可以获取到，就会直接返回，否则会执行创建 bean 的流程

![image-20220127005617722](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220127005617722.png)

#### **step2：getSingleton(beanName, true)**

源码如下，这个方法内部会调用`getSingleton(beanName, true)`获取 bean，注意第二个参数是`true`，这个表示是否可以获取早期的 bean，这个参数为 true，会尝试从三级缓存`singletonFactories`中获取 bean，然后将三级缓存中获取到的 bean 丢到二级缓存中。

```java
public Object getSingleton(String beanName) {
    return getSingleton(beanName, true);
}

protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    //从第1级缓存中获取bean
    Object singletonObject = this.singletonObjects.get(beanName);
    //第1级中没有,且当前beanName在创建列表中
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        synchronized (this.singletonObjects) {
            //从第2级缓存汇总获取bean
            singletonObject = this.earlySingletonObjects.get(beanName);
            //第2级缓存中没有 && allowEarlyReference为true，
            //也就是说2级缓存中没有找到bean且beanName在当前创建列表中的时候，才会继续想下走。
            if (singletonObject == null && allowEarlyReference) {
                //从第3级缓存中获取bean
                ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                //第3级中有获取到了
                if (singletonFactory != null) {
                    //3级缓存汇总放的是ObjectFactory，所以会调用其getObject方法获取bean
                    singletonObject = singletonFactory.getObject();
                    //将3级缓存中的bean丢到第2级中
                    this.earlySingletonObjects.put(beanName, singletonObject);
                    //将bean从三级缓存中干掉
                    this.singletonFactories.remove(beanName);
                }
            }
        }
    }
    return singletonObject;
}
```

#### **step3：getSingleton(String beanName, ObjectFactory<?> singletonFactory)**

上面调用`getSingleton(beanName, true)`没有获取到 bean，所以会继续走 bean 的创建逻辑，会走到下面代码，如下

![image-20220127005737791](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220127005737791.png)

进入`getSingleton(String beanName, ObjectFactory<?> singletonFactory)`，源码如下，只留了重要的部分

```java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
    //从第1级缓存中获取bean，如果可以获取到，则自己返回
    Object singletonObject = this.singletonObjects.get(beanName);
    if (singletonObject == null) {
        //将beanName加入当前创建列表中
        beforeSingletonCreation(beanName);
        //①：创建单例bean
        singletonObject = singletonFactory.getObject();
        //将beanName从当前创建列表中移除
        afterSingletonCreation(beanName);
        //将创建好的单例bean放到1级缓存中,并将其从2、3级缓存中移除
        addSingleton(beanName, singletonObject);
    }
    return singletonObject;
}
```

注意代码`①`，会调用`singletonFactory.getObject()`创建单例 bean，我们回头看看`singletonFactory`这个变量的内容，如下图，可以看出主要就是调用`createBean`这个方法

![image-20220127005845326](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220127005845326.png)

下面我们进入`createBean`方法，这个内部最终会调用`doCreateBean`来创建 bean，所以我们主要看`doCreateBean`。

#### **step4：doCreateBean**

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
        throws BeanCreationException {

    // ①：创建bean实例，通过反射实例化bean，相当于new X()创建bean的实例
    BeanWrapper instanceWrapper = createBeanInstance(beanName, mbd, args);

    // bean = 获取刚刚new出来的bean
    Object bean = instanceWrapper.getWrappedInstance();

    // ②：是否需要将早期的bean暴露出去，所谓早期的bean相当于这个bean就是通过new的方式创建了这个对象，
    //	但是这个对象还没有填充属性，所以是个半成品
    
    // 是否需要将早期的bean暴露出去，判断规则（bean是单例 && 是否允许循环依赖 && bean是否在正在创建的列表中）
    boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
            isSingletonCurrentlyInCreation(beanName));

    if (earlySingletonExposure) {
        //③：调用addSingletonFactory方法，这个方法内部会将其丢到第3级缓存中，并从二级缓存中删除
        //getEarlyBeanReference的源码可以看一下,内部会调用一些方法获取早期的bean对象,比如可以在这个里面通过aop生成代理对象
        addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
    }

    // 这个变量用来存储最终返回的bean
    Object exposedObject = bean;
    //填充属性，这里面会调用setter方法或者通过反射将依赖的bean注入进去
    populateBean(beanName, mbd, instanceWrapper);
    //④：初始化bean，内部会调用BeanPostProcessor的一些方法，对bean进行处理，这里可以对bean进行包装，比如生成代理
    exposedObject = initializeBean(beanName, exposedObject, mbd);


    //早期的bean是否被暴露出去了
    if (earlySingletonExposure) {
        /**
         *⑤：getSingleton(beanName, false)，注意第二个参数是false，这个为false的时候，
         * 只会从第1和第2级中获取bean，此时第1级中肯定是没有的（只有bean创建完毕之后才会放入1级缓存）
         */
        Object earlySingletonReference = getSingleton(beanName, false);
        /**
         * ⑥：如果earlySingletonReference不为空，说明第2级缓存有这个bean，二级缓存中有这个bean，说明了什么？
         * 大家回头再去看看上面的分析【step2】，看一下什么时候bean会被放入2级缓存?
         * （若 bean存在三级缓存中 && beanName在当前创建列表的时候，此时其他地方调用了getSingleton(beanName, true)方法，那么bean会从三级缓存移到二级缓存）
         */
        if (earlySingletonReference != null) {
            //⑥：exposedObject==bean，说明bean创建好了之后，后期没有被修改
            if (exposedObject == bean) {
                //earlySingletonReference是从二级缓存中获取的，二级缓存中的bean来源于三级缓存，三级缓存中可能对bean进行了包装，比如生成了代理对象
                //那么这个地方就需要将 earlySingletonReference 作为最终的bean
                exposedObject = earlySingletonReference;
            } else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
                //回头看看上面的代码，刚开始exposedObject=bean，
                // 此时能走到这里，说明exposedObject和bean不一样了，他们不一样了说明了什么？
                // 说明initializeBean内部对bean进行了修改
                // allowRawInjectionDespiteWrapping（默认是false）：是否允许早期暴露出去的bean(earlySingletonReference)和最终的bean不一致
                // hasDependentBean(beanName)：表示有其他bean以利于beanName
                // getDependentBeans(beanName)：获取有哪些bean依赖beanName
                String[] dependentBeans = getDependentBeans(beanName);
                Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
                for (String dependentBean : dependentBeans) {
                    //判断dependentBean是否已经被标记为创建了，就是判断dependentBean是否已经被创建了
                    if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                        actualDependentBeans.add(dependentBean);
                    }
                }
                /**
                 *
                 * 能走到这里，说明早期的bean被别人使用了，而后面程序又将exposedObject做了修改
                 * 也就是说早期创建的bean是A，这个A已经被有些地方使用了，但是A通过initializeBean之后可能变成了B，比如B是A的一个代理对象
                 * 这个时候就坑了，别人已经用到的A和最终容器中创建完成的A不是同一个A对象了，那么使用过程中就可能存在问题了
                 * 比如后期对A做了增强（Aop），而早期别人用到的A并没有被增强
                 */
                if (!actualDependentBeans.isEmpty()) {
                    //弹出异常（早期给别人的bean和最终容器创建的bean不一致了，弹出异常）
                    throw new BeanCurrentlyInCreationException(beanName,"异常内容见源码。。。。。");
                }
            }
        }
    }

    return exposedObject;
}
```

上面的 step1~step4，大家要反复看几遍**，下面这几个问题搞清楚之后，才可以继续向下看，不懂的结合源码继续看上面几个步骤**

**1、什么时候 bean 被放入 3 级缓存？**

bean实例化完成（通过反射创建之后），还没有属性填充之前（populateBean），在放入3级缓存的时候同时若这个beanName出现在了2级缓存，那么将2级缓存中的对象删除。

**2、什么时候 bean 会被放入 2 级缓存？**

当 beanX 还在创建的过程中，此时被加入当前 beanName 创建列表了，但是这个时候 bean 并没有被创建完毕（bean 被丢到一级缓存才算创建完毕），此时 bean 还是个半成品，这个时候其他 bean 需要用到 beanX，此时会从三级缓存中获取到 beanX，beanX 会从三级缓存中丢到 2 级缓存中。

**3、什么时候 bean 会被放入 1 级缓存？**

bean 实例化完毕，初始化完毕，属性注入完毕，bean 完全组装完毕之后，才会被丢到 1 级缓存。

**4、populateBean 方法是干什么的？**

填充属性的，比如注入依赖的对象。

### **6.3、下面来看 A、B 类 setter 循环依赖的创建过程**

1、getSingleton("a", true) 获取 a：会依次从 3 个级别的缓存中找 a，此时 3 个级别的缓存中都没有 a

2、将 a 丢到正在创建的 beanName 列表中（Set<String> singletonsCurrentlyInCreation）【真正实例化之前】

3、实例化 a：A a = new A();【利用反射】这个时候 a 对象是早期的 a，属于半成品

4、将早期的 a 丢到三级缓存中（Map<String, ObjectFactory<?> > singletonFactories），删除2级缓存中的a【属性填充之前】

5、调用 populateBean 方法，注入依赖的对象，发现 setB 需要注入 b

6、调用 getSingleton("b", true) 获取 b：会依次从 3 个级别的缓存中找 a，此时 3 个级别的缓存中都没有 b

7、将 b 丢到正在创建的 beanName 列表中【真正实例化之前】

8、实例化 b：B b = new B();这个时候 b 对象是早期的 b，属于半成品

9、将早期的 b 丢到三级缓存中（Map<String, ObjectFactory<?> > singletonFactories）【属性填充之前】

10、调用 populateBean 方法，注入依赖的对象，发现 setA 需要注入 a

11、调用 getSingleton("a", true) 获取 a：此时 a 会从第 3 级缓存中被移到第 2 级缓存，然后将其返回给 b 使用，此时 a 是个半成品（属性还未填充完毕）

12、b 通过 setA 将 11 中获取的 a 注入到 b 中

13、b 被创建完毕，此时 b 会从第 3 级缓存中被移除，也从2级缓存中移除，然后被丢到 1 级缓存

14、b 返回给 a，然后 b 被通过 A 类中的 setB 注入给 a

15、a 的 populateBean 执行完毕，即：完成属性填充，到此时 a 已经注入到 b 中了

16、调用`a= initializeBean("a", a, mbd)`对 a 进行处理，这个内部可能对 a 进行改变，有可能导致 a 和原始的 a 不是同一个对象了

17、调用`getSingleton("a", false)`获取 a，注意这个时候第二个参数是 false，这个参数为 false 的时候，只会从前 2 级缓存中尝试获取 a，而 a 在步骤 11 中已经被丢到了第 2 级缓存中，所以此时这个可以获取到 a，这个 a 已经被注入给 b 了

18、此时判断注入给 b 的 a 和通过`initializeBean`方法产生的 a 是否是同一个 a，不是同一个，则弹出异常

**从上面的过程中我们可以得到一个非常非常重要的结论**

**当某个 bean 进入到 2 级缓存的时候，说明这个 bean 的早期对象被其他 bean 注入了，也就是说，这个 bean 还是半成品，还未完全创建好的时候，已经被别人拿去使用了，所以必须要有 3 级缓存，2 级缓存中存放的是早期的被别人使用的对象，如果没有 2 级缓存，是无法判断这个对象在创建的过程中，是否被别人拿去使用了。**

3 级缓存是为了解决一个非常重要的问题：早期被别人拿去使用的 bean 和最终成型的 bean 是否是一个 bean，如果不是同一个，则会产生异常，所以以后面试的时候被问到为什么需要用到 3 级缓存的时候，你只需要这么回答就可以了：**三级缓存是为了判断循环依赖的时候，早期暴露出去已经被别人使用的 bean 和最终的 bean 是否是同一个 bean，如果不是同一个则弹出异常，如果早期的对象没有被其他 bean 使用，而后期被修改了，不会产生异常，如果没有三级缓存，是无法判断是否有循环依赖，且早期的 bean 被循环依赖中的 bean 使用了。**。

spring 容器默认是不允许早期暴露给别人的 bean 和最终的 bean 不一致的，但是这个配置可以修改，而修改之后存在很大的分享，所以不要去改，通过下面这个变量控制

```javascript
org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#allowRawInjectionDespiteWrapping

private boolean allowRawInjectionDespiteWrapping = false;
```



## aop情况下的循环依赖

A和B循环依赖，A和B都被一个切面类切了一下

1、A首先实例化

2、将A放到三级缓存

3、开始对A进行属性填充，这时候发现A依赖B

4、一二三级缓存中都没有发现B，B开始实例化

5、将B放到三级缓存

6、对B开始属性填充，这时候发现B依赖A

7、在三级缓存中发现了A，这里会对A进行AOP，然后将A从三级缓存中删除，把**代理后的A放到二级缓存中**【下面有源码】

8、B将A注入，注入的是代理后的A，此时B完成属性填充

9、B因为被切面类切了一下，属性填充完毕之后由BeanPostProcessor【AOP那个】进行增强

10、B增强完毕，放入一级缓存，从二三级缓存中删掉B，这时候B就在一级缓存中了

11、回到A，这时候A将B注入，注入的是代理后的B，但是！这时候注意，这里的A是没有被增强的

12、**A属性填充完毕，它不会被再次增强【warpIfNecessary】！**

13、A会去二级缓存中，拿到经过代理的A，再将这个A放到一级缓存

14、至此，A和B的循环依赖解决

**warpIfNecessary**方法如下：可以看到，如果Bean是经过代理放到二级缓存的，它会被put进一个earlyProxyReferences的Map中保存，以此来判断在属性填充完毕之后到底要不要再次对这个Bean进行增强，这也是这个方法为什么这么命名的缘故`WarpIfNecessary【包装如果需要的话】`

![image-20220127125045399](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220127125045399.png)



#### 从三级缓存中拿还没有初始化的bean的时候，可能会有AOP

```java
	protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
		Object exposedObject = bean;
		if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
			for (BeanPostProcessor bp : getBeanPostProcessors()) {
                //AbstractAutoProxyCreator继承了这个SmartXXX,AbstractAutoProxyCreator就是拿来做AOP的
				if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
					SmartInstantiationAwareBeanPostProcessor ibp 
                        = (SmartInstantiationAwareBeanPostProcessor) bp;
                    //这里会对Bean增强
					exposedObject = ibp.getEarlyBeanReference(exposedObject, beanName);
				}
			}
		}
		return exposedObject;
	}

	@Override
	public Object getEarlyBeanReference(Object bean, String beanName) {
		Object cacheKey = getCacheKey(bean.getClass(), beanName);
        //放进这个早期代理引用，然后会对bean进行增强
		this.earlyProxyReferences.put(cacheKey, bean);
		return wrapIfNecessary(bean, beanName, cacheKey);
	}
```

