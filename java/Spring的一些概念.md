# Spring的一些概念

## 在IoC容器的初始化中，@Bean的BeanDefinition是什么时候进入容器的?

对于xml配置的，在IoC源码中详细解释了

```java
ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
此方法执行的过程中，会扫描每一个xml文件，转成DOM树，然后解析这个树，把一个个bean标签变成BeanDefinition放入BeanFactory中
```

但是@Bean标注的组件，不是这样做的。@Bean标注要放入IoC的组件，源码流程如下：

```bash
refresh() -> invokeBeanFactoryPostProcessors(beanFactory);
在此方法执行的过程中才将BeanDefinition放入BeanFactory
1.拿到ConfigurationClassPostProcessor【BeanDefinitionRegistryPostProcessor】
2.调用postProcessor.postProcessBeanDefinitionRegistry(registry);
3.找到@Configuration标注的类
4.执行loadBeanDefinitionsForConfigurationClass(configClass, trackedConditionEvaluator);
5.找到所有@Bean标注的方法configClass.getBeanMethods()
6.执行loadBeanDefinitionsForBeanMethod(beanMethod);
7.然后就是对这个@Bean标注的方法进行一系列解析了
8.this.beanDefinitionMap.put(beanName, beanDefinition);往IoC容器中放入BeanDefinition
```



## @Configuration是一个配置类，他是如何配置的? @Component?@Service?

Debug的时候给了大体的流程，代码很复杂，会处理很多情况

- **注解的Spring和XML的Spring往容器中加BeanDefinition的时机是不同的**
- 注解模式中：我们自己要给容器中加的组件，都是在invokeBeanFactoryPostProcessors这个代码里面注入。
  - 找一个个@Configuration，然后for循环依次分析
- XML模式中：都是在ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory()这个代码里面注入。
  - 找一个个Spring的xml配置文件，然后for循环依次分析。

```java
@ComponentScan("com.scnu.aop")
@Configuration
@EnableAspectJAutoProxy
@Import({...})
public class MainConfigOfAop
```

```java
1.invokeBeanFactoryPostProcessors(beanFactory);
2.拿到ConfigurationClassPostProcessor【BeanDefinitionRegistryPostProcessor】
    调用postProcessor.postProcessBeanDefinitionRegistry(registry);
3.然后会找到标注了注解@Configuration的类
4.ConfigurationClassParser parser解析这个类
5.标注了@Configuration的类是一个AnnotatedBeanDefinition
6.拿到这个类的所有注解信息,开始解析,这里就会拿到@ComponentScan,@Import这四个
7.若有@Import，那么记录起来
8.若是@Component,处理一下,@Configuration就是一个@Component
9.若标注了@ComponentScan,处理这个注解
    1、这里就可以看到basePackages，excludeFilters，includeFilters这些，其实就是处理这个注解的每个参数，
    2、这个例子中只有basePackages="com.scnu.aop"
    3、去类路径扫这个baskPackages的类文件 "classpath*:com/scnu/aop/**/*.class"
    4、找这些class,看看哪一个标注了@Component注解！
    	注意了，就是这里，@Controller,@Service,@Repository,@Component都在这里
    	然后会生成beanDefinition!
    5、this.beanDefinitionMap.put(beanName, beanDefinition); 存到IoC容器
10.若标注了@Import,那么处理这个注解
    看看有没有导入ImportSelector，ImportBeanDefinitionRegistrar
    	@EnableAspectJAutoProxy导入了一个AspectJAutoProxyRegistrar，它是一个ImportBeanDefinitionRegistrar
    若有的话，Candidate class is an ImportBeanDefinitionRegistrar ->
			delegate to it to register additional bean definitions
    代码中给了个注释，后面会处理它。
11.若有@Bean的方法，存起来待用。
    configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
12.this.reader.loadBeanDefinitions(configClasses);
	这里会处理@Bean的方法,也是往IoC容器加BeanDefinition
```





## 组件注册

```
给容器中注册组件:
1.包扫描+组件标注注解 @Controller @Component @Service @Repository【局限于自己写的】
2.@Bean  【能够导入的第三方包里面的组件】
3.@Import【快速给容器中导入一个组件】
     1)、@Import(组件.class):容器中就会自动注入这个组件,id默认是全类名
     2)、@ImportSelector:返回需要导入的组件的全类名数组
     3)、@ImportBeanDefinitionRegistrar:手动注册Bean到容器中
4.使用Spring提供的FactoryBean
     1)、默认获取的是工厂bean调用getObject创建的对象
     2)、要获取工厂Bean本身，我们需要给id前面加一个&
             BeanFactory中有一个属性String FACTORY_BEAN_PREFIX = "&";
```

### @Import

#### @Import

```java
public class Color {
}
@Configuration
@Import({Color.class})
public class MainConfig {
}
//这样就可以在容器中导入Color这个类，组件的名字默认是全类名
```

#### ImportSelector --- 这个用的很多

```java
//自定义逻辑,返回需要导入的组件
public class MyImportSelector implements ImportSelector {

    //返回值就是要导入到容器中的组件全类名
    //AnnotationMetadata:当前标注@Import注解类的所有注解信息
    	//此方法中,能拿到@Configuration和@Import
    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        //返回值为null会报错
        return new String[]{"java.util.Date","com.scnu.lfy.bean.Color"};
    }
}
@Configuration
@Import({Color.class, MyImportSelector.class})
public class MainConfig {
}

public static void main(String[] args) {
    ApplicationContext ac = new AnnotationConfigApplicationContext(MainConfig.class);
    String[] beanDefinitionNames = ac.getBeanDefinitionNames();
    for(String name :beanDefinitionNames){
        System.out.println(name);
    }
}
/*
....
com.scnu.lfy.bean.Color
java.util.Date
com.scnu.lfy.bean.Person
*/
```

#### ImportBeanDefinitionRegistrar

在IoC启动的invokeBeanFactoryPostProcessors(beanFactory);这个环节调用

![image-20220123194929596](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220123194929596.png)

```java
public class MyImportBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar {
    /**
     *
     *  AnnotationMetadata:当前类的注解信息
     *  BeanDefinitionRegistry:BeanDefinition注册类
     *      把所有需要添加到容器中的bean:
     *      调用BeanDefinition.registerBeanDefinition手工注册
     */
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        boolean red = registry.containsBeanDefinition("com.scnu.lfy.bean.Red");
        boolean blue = registry.containsBeanDefinition("com.scnu.lfy.bean.Blue");
        if(red && blue){
            //指定Bean定义信息,(Bean的类型,单例还是多例)
            RootBeanDefinition rootBeanDefinition = new RootBeanDefinition(RainBow.class);
            registry.registerBeanDefinition("rainBow",rootBeanDefinition);
        }
    }
}
@Configuration
@Import({Red.class, Blue.class,MyImportBeanDefinitionRegistrar.class})
public class MainConfig {
}
/*
...
com.scnu.lfy.bean.Red
com.scnu.lfy.bean.Blue
rainBow
*/
```



### FactoryBean

​	若是直接在容器中注入这个FactoryBean的话，情况如下面的代码所示，只有等到第一次拿这个bean的时候，FactoryBean才调用getObject方法真正的获取我们想要的bean

```java
//创建一个Spring定义的FactoryBean
public class ColorFactoryBean implements FactoryBean<Color> {
    //返回一个Color对象,这个对象会添加到容器中
    @Override
    public Color getObject() throws Exception {
        System.out.println("ColorFactoryBean....");
        return new Color();
    }

    @Override
    public Class<?> getObjectType() {
        return Color.class;
    }

    //true:这个bean是单例,在容器中保存一份
    //false:多实例,每次获取都会创建一个新的Bean
    /*
        若返回true,容器在初始化的时候并不会将它放进容器,只有等到用到了才创建一个实例,并放入容器
     */
    @Override
    public boolean isSingleton() {
        return true;
    }
}
@Configuration
public class MainConfig {
    @Bean
    public ColorFactoryBean colorFactoryBean(){
        return new ColorFactoryBean();
    }
}
public static void main(String[] args) {
    ApplicationContext ac = new AnnotationConfigApplicationContext(MainConfig.class);
    String[] beanDefinitionNames = ac.getBeanDefinitionNames();
    for(String name :beanDefinitionNames){
        System.out.println(name);
        if(name.equals("colorFactoryBean")){
            System.out.println(ac.getBean(name));
        }
    }
}
/*
...
colorFactoryBean
ColorFactoryBean....
com.scnu.lfy.bean.Color@646007f4
*/
```

​	如果我们一定想拿到FactoryBean怎么办？ 加一个前缀 & 即可

```java
public static void main(String[] args) {
    ApplicationContext ac = new AnnotationConfigApplicationContext(MainConfig.class);
    String[] beanDefinitionNames = ac.getBeanDefinitionNames();
    for(String name :beanDefinitionNames){
        System.out.println(name);
        if(name.equals("colorFactoryBean")){
            System.out.println(ac.getBean("&" + name));
        }
    }
}
/*
...
mainConfig
colorFactoryBean
com.scnu.lfy.bean.ColorFactoryBean@646007f4
*/

```

​	标识FactoryBean的前缀是&

![image-20220123135421839](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220123135421839.png)

![image-20220123135705538](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220123135705538.png)

​	底层会判断，如果加了前缀的话，就知道要返回FactoryBean，因此到了相应步骤就返回了，而不是继续执行去调用这个FactoryBean的getObject方法

![image-20220123140417631](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220123140417631.png)

![image-20220123140516930](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220123140516930.png)



## Bean的生命周期

bean的生命周期:
     bean创建---初始化---销毁的过程

容器管理bean的生命周期:

​	我们可以自定义初始化和销毁的方法，当容器在bean进行到当前生命周期的时候来调用我们自定义的初始化和销毁方法。

构造（对象创建）：

​	单实例在容器启动的时候就会创建对象

​	多实例在每次获取的时候创建

### 指定初始化和销毁方法

xml配置中，可以在bean标签处声明init-method和destory-method

注解配置中，通过@Bean指定init-method 和 destroy-method

对于数据源来说，初始化的时候要有很多配置，销毁的时候要关闭一些资源。

```
初始化:对象创建完成,并赋值好(populateBean之后),调用初始化方法
销毁: 单实例,容器关闭的时候
     多实例,容器不会管理这个Bean,不会调用销毁方法,这时候可能需要自己完成销毁工作
```

```java
@Configuration
public class MainConfig {

    @Bean(initMethod = "init",destroyMethod="destroy")
    public Car car(){
        return new Car();
    }

}
public class Car {
    public Car() {
        System.out.println("car constructor...");
    }

    public void init(){
        System.out.println("car ... init");
    }

    public void destroy(){
        System.out.println("car ... destroy");
    }
}
```

```java
@Test
public void test02(){
    AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(MainConfig.class);
    System.out.println("容器创建完毕");
    ac.close();		//不调用close的话, destroy方法不会被调用
}
/*
car constructor...
car ... init
容器创建完毕
car ... destroy
*/
```

### Bean实现InitializingBean(定义初始化逻辑)， (定义销毁逻辑)

```java
@Component
public class Cat  implements InitializingBean, DisposableBean {
    public Cat() {
        System.out.println("cat constructor...");
    }

    @Override
    public void destroy() throws Exception {	//实现了DisposableBean要重写的
        System.out.println("cat destroy ....");
    }

    @Override
    public void afterPropertiesSet() throws Exception {	//实现了InitializingBean要重写的
        System.out.println("cat afterPropertiesSet...");
    }
}
@Configuration
public class MainConfig {
    
    @Bean
    public Cat cat(){
        return new Cat();
    }

}
/*
cat constructor...
cat afterPropertiesSet...
容器创建完毕
cat destroy ....
*/
```

### 可以使用JSR250

@PostConstruct:在bean创建完成并且属性赋值完成,来执行初始化方法
@PreDestory:在容器销毁bean之前通知我们进行清理工作

**@PostConstruct这个注解会给容器加一个InitDestroyAnnotationBeanPostProcessor，即加一个bean后置处理器，在postProcessBeforeInitialization这个方法中完成@PostConstruct注解对应的方法,这里是init**

```java
@Component
public class Dog {
    public Dog() {
        System.out.println("Dog constructor...");
    }

    //对象创建并赋值之后调用
    @PostConstruct
    public void init(){
        System.out.println("Dog init...");
    }

    //容器移除对象之前
    @PreDestroy
    public void destroy(){
        System.out.println("Dog destroy...");
    }
}
@Configuration
public class MainConfig {

    @Bean
    public Dog dog(){
        return new Dog();
    }

}
/*
Dog constructor...
Dog init...
容器创建完毕
Dog destroy...
*/
```

### BeanPostProcessor:bean的后置处理器

在bean初始化前后进行一些处理工作

postProcessBeforeInitialization:在初始化之前工作
postProcessAfterInitialization:在初始化之后工作

```java
@Component
public class MyBeanPostProcessor implements BeanPostProcessor {
    /*
    bean – the new bean instance
	beanName – the name of the bean
    */
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("postProcessBeforeInitialization..." + beanName+"==>"+bean.getClass());
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("postProcessAfterInitialization..." + beanName+"==>"+bean.getClass());
        return bean;
    }
}
@Configuration
@ComponentScan("com.scnu.lifeCycle")
public class MainConfig {
 	//这里用包扫描才能看到效果??怪怪的,@Bean的话看不到输出 
    //这个demo扫描了Dog和Cat
}
/*
...其他一些Bean

cat constructor...
postProcessBeforeInitialization...cat==>class com.scnu.lifeCycle.Cat
cat afterPropertiesSet...
postProcessAfterInitialization...cat==>class com.scnu.lifeCycle.Cat

Dog constructor...
postProcessBeforeInitialization...dog==>class com.scnu.lifeCycle.Dog
Dog init...
postProcessAfterInitialization...dog==>class com.scnu.lifeCycle.Dog

容器创建完毕
Dog destroy...
cat destroy ....
*/
```

#### BeanPostProcessor的原理

> ```
> createBean(); 	//创建一个bean
> populateBean(); //为这个bean进行属性赋值,set方法等。
> initializeBean(beanName, exposedObject, mbd){
> 	//注意看,可以包装bean,即可以增强Bean的功能
>     wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
>     invokeInitMethods(beanName, wrappedBean, mbd);	//这里就是一些初始化方法,指定init,@PostConstruct等
>     wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
> }
> 
> applyBeanPostProcessorsBeforeInitialization的调用:
> 遍历容器中所有的BeanPostProcessor,挨个执行postProcessBeforeInitialization,若返回null,那么将null之前的对象返回,不再执行后面的BeanPostProcessor
> 	@Override
> 	public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
> 			throws BeansException {
> 
> 		Object result = existingBean;
> 		for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) {
> 			result = beanProcessor.postProcessBeforeInitialization(result, beanName);
> 			if (result == null) {
> 				return result;
> 			}
> 		}
> 		return result;
> 	}
> ```



#### Spirng底层一些BeanPostProcessor的使用

##### 动态拿到ApplicationContext对象

```java
@Component
public class MyApplicationContext implements ApplicationContextAware {

    private ApplicationContext ac;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        System.out.println("拿到applicationContext");
        this.ac = applicationContext;
    }
}
```

Spring容器中有ApplicationContextAwareProcessor

```java
class ApplicationContextAwareProcessor implements BeanPostProcessor {
    //在目标Bean初始化方法执行前就拿到IoC容器了
	@Override
	public Object postProcessBeforeInitialization(final Object bean, String beanName) throws BeansException {
		AccessControlContext acc = null;
		...
		else {
			invokeAwareInterfaces(bean);
		}

		return bean;
	}

    	private void invokeAwareInterfaces(Object bean) {
		if (bean instanceof Aware) {
            ...
                //此处拿到ApplicationContext,即IoC容器
			if (bean instanceof ApplicationContextAware) {
				((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
			}
		}
	}
}
```

##### JSR303校验

BeanValidationPostProcessor

当Bean属性赋值完毕,在执行初始化方法之前，进行数据校验

```java
	@Override
	public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		if (!this.afterInitialization) {
			doValidate(bean);	//此方法会具体校验bean是否合法,不合法抛异常
		}
		return bean;
	}
```

##### JSR250规范：@PostConsurt和@PreDestroy

InitDestroyAnnotationBeanPostProcessor

```java
	@Override
	public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		LifecycleMetadata metadata = findLifecycleMetadata(bean.getClass());
		try {
			metadata.invokeInitMethods(bean, beanName);	//此处调用由@PostConstruct标注的方法
		}
        ...
	}
```

```java
	@Override
	public void postProcessBeforeDestruction(Object bean, String beanName) throws BeansException {
		LifecycleMetadata metadata = findLifecycleMetadata(bean.getClass());
		try {
			metadata.invokeDestroyMethods(bean, beanName);	//此处调用由@PreDestroy标注的方法
		}
        ...
	}
```

##### 自动装配

通过AutowiredAnnotationBeanPostProcessor来实现。





### 生命周期图示

注意，如果有@PostConstruct,它的初始化逻辑是由一个InitDestroyAnnotationBeanPostProcessor完成的，在图示的BeanPostProcessor处完成。

![image-20220122180739690](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220122180739690.png)



## 如何使用Spring底层的一些组件？

自定义组件实现xxxAware接口：在执行到具体生命周期过程中会调用接口规定的方法注入相关组件。

- ApplicationContextAware接口：拿到Ioc容器
- BeanNameAware接口：拿到实现这个接口的bean在beanFactory中的名字
- EmbeddedValueResolverAware接口：解析字符串
- xxxAware的功能是使用xxxProcessor来完成的。

```java
@Component
public class MyApplicationContext implements ApplicationContextAware, BeanNameAware, EmbeddedValueResolverAware {

    private ApplicationContext ac;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        System.out.println("拿到applicationContext");
        this.ac = applicationContext;
    }

    @Override
    public void setBeanName(String name) {
        System.out.println("在beanFactory中是什么名字？---》" + name);
    }

    @Override
    public void setEmbeddedValueResolver(StringValueResolver resolver) {
        System.out.println(resolver.resolveStringValue("你好,${os.name},哈哈哈,#{3 + 4}"));
        //输出你好,Windows 10,哈哈哈,7
    }
}

```



## BeanFactoryPostProcessor

​	BeanFactoryPostProcessor:beanFactory的后置处理器，在beanFactory标准初始化之后调用,所有的bean定义已经保存加载到beanFactory中,但是bean的实例还未创建。

**ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();**即在此行代码执行完之后的时间点，详细可以看Ioc源码分析

![image-20220122220520746](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220122220520746.png)

从下面代码可以看到，postProcessBeanFactory执行的时候，TestBean还没有被创建出来。

```java
@Configuration
@ComponentScan("com.scnu.beanFactoryPostProcessor")
public class MainConfig {

    @Bean
    public TestBean testBean(){
        return new TestBean();
    }

}
@Component
public class MyBeanFactoryPostProcessor implements BeanFactoryPostProcessor {

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        int count = beanFactory.getBeanDefinitionCount();
        System.out.println("当前beanFactory中bean定义的个数:" + count);
        System.out.println(Arrays.asList(beanFactory.getBeanDefinitionNames()));
        System.out.println("postProcessBeanFactory方法结束...");
    }
}
/*
当前beanFactory中bean定义的个数:9
[org.springframework.context.annotation.internalConfigurationAnnotationProcessor, org.springframework.context.annotation.internalAutowiredAnnotationProcessor, org.springframework.context.annotation.internalRequiredAnnotationProcessor, org.springframework.context.annotation.internalCommonAnnotationProcessor, org.springframework.context.event.internalEventListenerProcessor, org.springframework.context.event.internalEventListenerFactory, mainConfig, myBeanFactoryPostProcessor, testBean]
postProcessBeanFactory方法结束...
TestBean被创建...
容器创建完毕
*/
```

1.IOC容器创建对象
2.**invokeBeanFactoryPostProcessors(beanFactory);**执行BeanFactoryPostProcessors
     如何找到所有的BeanFactoryPostProcessor并执行它们的方法？
         1).在beanFactory中找到类型是BeanFactoryPostProcessor的组件,执行它们的方法
         2).在初始化Bean前面执行



## BeanDefinitionRegistryPostProcessor

**可以利用BeanDefinitionRegistryPostProcessor给容器中额外添加一些组件，注意了，基于注解的Spring，像@Bean标注的，要放入Spring容器，就是通过这个类来实现的！详细请看本文最上方。**

```java
BeanDefinitionRegistryPostProcessor优先于BeanFactoryPostProcessor执行,
 可以利用BeanDefinitionRegistryPostProcessor给容器中额外添加一些组件
原理:
 1).refresh()->invokeBeanFactoryPostProcessors(beanFactory);
 2).从容器中获取所有BeanDefinitionRegistryPostProcessor组件
     1.依次触发接口的方法
         for (BeanDefinitionRegistryPostProcessor postProcessor : postProcessors) {
             postProcessor.postProcessBeanDefinitionRegistry(registry);
         }
     2.上面的所有方法触发完成后
         // Now, invoke the postProcessBeanFactory callback of all processors handled so far.
         invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
         invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
 3).然后轮到BeanFactoryPostProcessor
     String[] postProcessorNames =
            beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);
      for (BeanFactoryPostProcessor postProcessor : postProcessors) {
         postProcessor.postProcessBeanFactory(beanFactory);
     }
```

```java
@Component
public class MyBeanDefinitionRegistryPostProcessor implements BeanDefinitionRegistryPostProcessor {
    //BeanDefinitionRegistry:bean定义信息的保存中心
    //以后BeanFactory就是按照BeanDefinitionRegistry里面保存的每一个bean定义信息创建bean实例
    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
        System.out.println("postProcessBeanDefinitionRegistry...bean的数量:" + registry.getBeanDefinitionCount());
        
        //自己注册一个BeanDefinition进去
        AbstractBeanDefinition beanDefinition = BeanDefinitionBuilder.rootBeanDefinition(TestBean.class).getBeanDefinition();
        registry.registerBeanDefinition("myTestBean",beanDefinition);
    }

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        System.out.println("postProcessBeanFactory..bean的数量:" + beanFactory.getBeanDefinitionCount());
    }
}
```

![image-20220122223703790](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220122223703790.png)