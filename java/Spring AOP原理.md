# Spring AOP原理

​	AOP(Aspect-Oriented Programming:面向切面编程)能够将那些与业务无关，却为业务模块所共同调用的逻辑或责任（例如事务处理、日志管理、权限控制等）封装起来，便于减少系统的重复代码，降低模块间的耦合度，并有利于未来的可拓展性和可维护性。

​	**减少重复代码，专注业务逻辑。**

## Spring AOP 和 AspectJ AOP 有什么区别？

> @Aspect、@Pointcut、@Before、@After 等注解都是来自于 AspectJ，但是功能的实现是纯 Spring AOP 自己实现的。
>
> **Spring AOP 属于运行时增强，而 AspectJ 是编译时增强。** Spring AOP 基于代理(Proxying)，而 AspectJ 基于字节码操作(Bytecode Manipulation)。
>
> **Spring AOP：**
>
> - 它基于动态代理来实现。默认地，如果使用接口的，用 JDK 提供的动态代理实现，如果没有接口，使用 CGLIB 实现。
>
> **AspectJ：**
>
> - AspectJ 出身也是名门，来自于 Eclipse 基金会，link：https://www.eclipse.org/aspectj
> - 属于静态织入，它是通过修改代码来实现的，它的织入时机可以是：
>   - Compile-time weaving：编译期织入，如类 A 使用 AspectJ 添加了一个属性，类 B 引用了它，这个场景就需要编译期的时候就进行织入，否则没法编译类 B。
>   - Post-compile weaving：也就是已经生成了 .class 文件，或已经打成 jar 包了，这种情况我们需要增强处理的话，就要用到编译后织入。
>   - **Load-time weaving**：指的是在加载类的时候进行织入，要实现这个时期的织入，有几种常见的方法。1、自定义类加载器来干这个，这个应该是最容易想到的办法，在被织入类加载到 JVM 前去对它进行加载，这样就可以在加载的时候定义行为了。2、在 JVM 启动的时候指定 AspectJ 提供的 agent：`-javaagent:xxx/xxx/aspectjweaver.jar`。
> - 因为 AspectJ 在实际代码运行前完成了织入，所以它生成的类是没有额外运行时开销的。

## Spring AOP例子分析

```java
1.导入aop模块:spring-aspects
2.定义一个业务逻辑类(MainCalculator),在业务逻辑运行的时候,打印日志(方法之前，方法结束)
3.定义一个日志切面类(LogAspects):切面类里面的方法需要动态感知MainCalculator.div运行
     通知方法
         前置通知(@Before):logStart,在目标方法div运行之前运行
         后置通知(@After): logAfter,在目标方法div结束之后运行,无论正常结束还是异常结束都运行
         返回通知(@AfterReturning): logReturn,在目标方法(div)正常返回之后运行
         异常通知(@AfterThrowing):  logException,在目标方法(div)出现异常以后运行
         环绕通知(@Around): 动态代理,手动推进目标方法运行(JoinPoint.proceed())
4.需要将切面类和业务逻辑类放入容器中,才能起作用
5.必须告诉Spring哪个是切面类(@Aspect)
6.给配置类中加上@EnableAspectJAutoProxy[开启基于注解的aop模式]
     功能等同于<aop:aspectj-autoproxy/>
核心三步:
     1.将业务逻辑组件和切面类都加入到容器中,告诉Spring哪个是切面类(@Aspect)
     2.在切面类上的每一个通知方法上都标注通知注解,告诉Spring何时何地运行(切入点表达式)
     3.开启基于注解的aop模式,@EnableAspectJAutoProxy
```

```java
@Aspect	//这是一个切面类
public class LogAspects {
    //抽取公共的切入点表达式
    //本类引用可以调pointCut()即可
    //其他类引用,写全名com.scnu.aop.LogAspects.pointCut()
    @Pointcut("execution(public int com.scnu.aop.MainCalculator.*(..))")
    public void pointCut(){}

    @Before("pointCut()")
    public void logStart(JoinPoint jp){
        Object[] objects = jp.getArgs();
        System.out.println("方法:" + jp.getSignature().getName() + "开始之前,打印一波日志,参数列表:" + Arrays.asList(objects));
    }

    @After("pointCut()")
    public void logAfter(JoinPoint jp){
        System.out.println("方法:" + jp.getSignature().getName() + "结束之后,打印一波日志");
    }

    @AfterReturning(value="pointCut()",returning = "result") //returning写的是什么名字,参数上就写一样的
    public Object logReturning(JoinPoint jp, Object result){
        System.out.println("方法"+jp.getSignature().getName()+"正常返回...结果是:" + result);
        return result;
    }

    @AfterThrowing(value="pointCut()",throwing = "e") //throwing写的是什么名字,参数上就写一样的，不然找不到
    public void logThrowing(JoinPoint jp,Exception e){
        System.out.println("方法"+jp.getSignature().getName()+"异常结束..." + e);
    }
}

```

```java
public class MainCalculator {
    public int div(int i,int j){
        return i / j;
    }
}
```

```java
//将业务逻辑组件和切面类都加入到容器中,告诉Spring哪个是切面类(@Aspect)
@Configuration
@EnableAspectJAutoProxy	//开启基于注解的aop模式
public class MainConfigOfAop {

    @Bean
    public MainCalculator mainCalculator(){
        return new MainCalculator();
    }

    @Bean
    public LogAspects logAspects(){
        return new LogAspects();
    }

}
```

测试代码

```java
@Test
public void testAOP(){
    ApplicationContext ac = new AnnotationConfigApplicationContext(MainConfigOfAop.class);
    MainCalculator calculator = ac.getBean(MainCalculator.class);
    System.out.println(calculator.div(6,3));
}
/*
方法:div开始之前,打印一波日志,参数列表:[6, 3]
方法:div结束之后,打印一波日志
方法div正常返回...结果是:2
2
*/
```



## 1.@EnableAspectJAutoProxy

> 小提示：(以后看见EnableXXX,就可分析它给容器注册了什么组件,分析组件什么时候工作,组件的功能)

<img src="https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220123172514066.png" alt="image-20220123172514066" style="zoom:67%;" />

注意看，快速导入了一个AspectJAutoProxyRegistrar，它其实是一个**ImportBeanDefinitionRegistrar**《Spring的一些概念》中有解释，就是可以手工注册一些BeanDefinition，这里主要是注册了一个AnnotationAwareAspectJAutoProxyCreator，它是AOP的核心类

**在refresh的invokeBeanFactoryPostProcessors(beanFactory)这个环节注册进入IoC容器。**

```java
@Import(AspectJAutoProxyRegistrar.class)
   利用AspectJAutoProxyRegistrar implements ImportBeanDefinitionRegistrar
                       (此接口用于手动给容器中注册bean)
   重写接口的registerBeanDefinitions方法，主要逻辑如下：
   refresh() -> invokeBeanFactoryPostProcessors(beanFactory)
   自定义给容器中注册beanDefinition:
          RootBeanDefinition beanDefinition = 
                 new RootBeanDefinition(AnnotationAwareAspectJAutoProxyCreator.class);
          ...
          registry.registerBeanDefinition("org.springframework.aop.config.internalAutoProxyCreator",                                                   beanDefinition);
	给容器中注册一个AnnotationAwareAspectJAutoProxyCreator(注解装配AspectJ自动代理创建器)
     它的名字是internalAutoProxyCreator(内部自动代理创建器)
```



## 2.AnnotationAwareAspectJAutoProxyCreator

### 继承结构图

这个注解装配AspectJ自动代理器其实就是一个BeanPostProcessor！那么我们就要关注它在bean的初始化前后干了什么！



![image-20220123173830893](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220123173830893.png)

**SmartInstantiationAwareBeanPostProcessor和BeanFactoryAware**

分析的时候，注意看，是AbstractAutoProxyCreator这个类首先继承他俩的，因此要从这里开始看，因为在这里可能就实现了相关方法。

**看源码的时候，从抽象开始，慢慢往具体的实现看**

给下面五个方法打断点。然后Debug

```
AbstractAutoProxyCreator.postProcessBeforeInstantiation  有后置处理器的逻辑
AbstractAutoProxyCreator.postProcessAfterInitialization
AbstractAutoProxyCreator.setBeanFactory
AbstractAdvisorAutoProxyCreator.setBeanFactory { initBeanFactory() }	这里又重写了setBeanFactory方法
AnnotationAwareAspectJAutoProxyCreator.initBeanFactory	这里又重写
```



## 3.创建AnnotationAwareAspectJAutoProxyCreator

AnnotationAwareAspectJAutoProxyCreator就是一个BeanPostProcessor

AnnotationAwareAspectJAutoProxyCreator的BeanDefinition会在invokeBeanFactoryPostProcessors(beanFactory)注册进去IoC容器。

【Debug过程得出的】创建流程如下：

```
1.传入配置类,创建ioc容器
2.注册配置类,调用refresh()刷新容器
2.5 invokeBeanFactoryPostProcessors(beanFactory)
	  调用AspectJAutoProxyRegistrar重写的接口方法
		使得AnnotationAwareAspectJAutoProxyCreator的BeanDefinition已经注册进IoC容器
3.registerBeanPostProcessors(beanFactory);注册bean的后置处理器,来拦截bean的创建
    1).先获取所有ioc容器已经定义的需要创建对象的BeanPostProcessor
        String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);
    2).给容器中加别的BeanPostProcessor
        beanFactory.addBeanPostProcessor(
        	new BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));
        注册BeanPostProcessorChecker，当BeanPostProcessor实例化期间创建bean时，即当一个bean不适合由所有BeanPostProcessor处理时，它会记录一条信息消息。
    3).First, register the BeanPostProcessors that implement PriorityOrdered.
    4).Next, register the BeanPostProcessors that implement Ordered.(AnnotationAwareAspectJAutoProxyCreator)
    5).Now, register all regular BeanPostProcessors.
    6). 由第四步,注册BeanPostProcessor,实际上就是创建它，并保存在容器中
        创建名称为internalAutoProxyCreator的BeanPostProcessor【AnnotationAwareAspectJAutoProxyCreator】
        1).创建Bean的实例
        2).populateBean(beanName, mbd, instanceWrapper); 属性赋值
            instanceWrapper就是AnnotationAwareAspectJAutoProxyCreator
            RootBeanDefinition mbd,就是上面一开始分析的
                            RootBeanDefinition beanDefinition = new 	  RootBeanDefinition(AnnotationAwareAspectJAutoProxyCreator.class);
            beanName就是internalAutoProxyCreator
        3).exposedObject = initializeBean(beanName, exposedObject, mbd);初始化bean
            1).invokeAwareMethods(beanName, bean); Aware接口的方法回调setBeanFactory
            2).wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
            3).invokeInitMethods(beanName, wrappedBean, mbd); 调用自定义初始化方法
            4).wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
        4).BeanPostProcessor(AnnotationAwareAspectJAutoProxyCreator)创建成功！
    7).把BeanPostProcessor注册到BeanFactory中【AnnotationAwareAspectJAutoProxyCreator】
        registerBeanPostProcessors(beanFactory, orderedPostProcessors);
```



在此之后，注册和创建AnnotationAwareAspectJAutoProxyCreator的过程结束，由于它是一个BeanPostProcessor，因此以后其他Bean想要注册进容器，首先得经过它，初始化前后都会调用BeanPostProcessor的两个接口方法，即拦截到Bean，在这里就可以增强Bean.



## 4.AnnotationAwareAspectJAutoProxyCreator的执行时机

`AnnotationAwareAspectJAutoProxyCreator ===> InstantiationAwareBeanPostProcessor`

```
finishBeanFactoryInitialization(beanFactory);
已经完成beanFactory初始化工作,创建剩下的单实例bean(如BeanPostProcessor就在之前创建了)
    1)、遍历获取容器中所有的bean,依次创建对象getBean(beanName)
    2)、创建bean
        结论:[AnnotationAwareAspectJAutoProxyCreator在所有bean创建之前
             会有一个拦截InstantiationAwareBeanPostProcessor,
             若可以代理的话它调用postProcessBeforeInstantiation()创建代理对象]
        1)、先从缓存中获取当前bean,如果能获取到,说明Bean之前被创建过,直接使用,否则再创建
        2)、createBean() 创建Bean，分两种情况
             [BeanPostProcessor是在Bean对象创建完成初始化前后调用的]
             [InstantiationAwareBeanPostProcessor是在创建Bean实例之前先尝试用后置处理器返回对象的]
                因此AnnotationAwareAspectJAutoProxyCreator会在任何bean创建之前先尝试返回bean的实例【代理】
            1)
                //Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
                Object bean = resolveBeforeInstantiation(beanName, mbdToUse)
                    bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName)
                    	拿到所有后置处理器,如果是InstantiationAwareBeanPostProcessor
                    		就执行postProcessBeforeInstantiation
					if (bean != null) {
						bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
					}
            2)
                if(bean == null)
                Object beanInstance = doCreateBean(beanName, mbdToUse, args); //真正的去创建一个Bean实例
```



## 5.创建AOP代理

<img src="https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220123222710344.png" alt="image-20220123222710344" style="zoom: 150%;" />

```java
分析例子：MainCalculator
AnnotationAwareAspectJAutoProxyCreator[InstantiationAwareBeanPostProcessor的作用]
 1.每个bean创建之前,都会调用接口的postProcessBeforeInstantiation()方法
     1).判断当前bean是否在advisedBean中(保存了所有需要增强的bean)
         Object cacheKey = getCacheKey(beanClass, beanName);
         this.advisedBeans.containsKey(cacheKey);
     2).判断当前bean是否是基础类型 Advice,AopInfrastructureBean,Pointcut,Advisor
         或者是否是切面@Aspect
     3).判断是否需要跳过(shouldSkip,就是上面的截图)
         获取候选的增强器(切面里面的通知方法 就是@Before,@After,@AfterReturning这些)
             List<Advisor> candidateAdvisors = findCandidateAdvisors();
             每一个封装的通知方法的增强器都是InstantiationModelAwarePointcutAdvisor【对于分析例子】
             这里判断增强器是否是advisor instanceof AspectJPointcutAdvisor(那么肯定就不是了)
第一步好像执行了个寂寞？bean没有被增强,而是正常创建，然后属性填充，然后调用初始化方法，然后！
```

下面这个方法，在AbstractAutoProxyCreator这个类调用。回去看看继承结构图。

![image-20220123223803480](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220123223803480.png)

看之前的例子分析，写的通知方法，就是一个个InstantiationModelAwarePointcutAdvisor

![image-20220123224546602](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220123224546602.png)

```java
AbstractAutoProxyCreator
2.增强Bean
    postProcessAfterInitialization【BeanPostProcessor接口的第二个方法,初始化后执行的，bean就在这里增强。】
        return wrapIfNecessary(bean, beanName, cacheKey){
            		// Create proxy if we have advice.
            1)获取当前bean的所有增强器(通知方法)
	                1)Object[] specificInterceptors = 
                			getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
	                2)获取到能在bean使用的增强器
	                3)给增强器排序(为了最后链式调用通知方法做准备,前置通知要放在最后面...)
	        2)保存当前bean在advisedBeans
	            this.advisedBeans.put(cacheKey, Boolean.TRUE);
            3)如果当前bean需要增强,那么创建它的代理对象
                1)Advisor[] advisors = buildAdvisors(beanName, specificInterceptors); 获取所有增强器(通知方法)
                2)proxyFactory.addAdvisors(advisors); 并保存到代理工厂中
                3)创建代理对象:Spring自动决定
                    JdkDynamicAopProxy(config);    有接口，jdk动态代理
                    ObjenesisCglibAopProxy(config);无接口,cglib动态代理
                4)给容器中返回当前组件使用cglib动态代理增强了的代理对象【因为MainCalculator没有实现接口】               
				5)以后容器中获取到的就是这个组件的代理对象,执行目标方法的时候,代理对象就会执行通知方法的流程
	     }
```



​	也就是说，如果一个Bean,被一个切面类切过了（即功能增强），那么会在这个Bean被创建之后，调用初始化方法完成后，由BeanPostProcessor的具体实现类AbstractAutoProxyCreator来增强这个Bean.以后从容器拿这个Bean,拿到的就是代理对象。



## 6.目标方法如何执行？【增强后的】

容器中保存了组件的代理对象(这里是cglib增强后的对象)这个对象里面保存了详细信息(增强器,目标对象等)

1).CglibAopProxy中的静态内部类DynamicAdvisedInterceptor.intercept() 拦截目标方法的执行

![image-20220124125735482](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220124125735482.png)

2).根据ProxyFactory获取将要执行的目标方法【div】的拦截器链

- List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
  - 注意看ArrayList，然后是目标方法的顺序【逆序】
  - 这里加了`@Around`的注解
  - ![image-20220126142214907](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220126142214907.png)
  
- 如果没有拦截器链【chain = null】，直接执行目标方法。

**那么，拦截器链是什么?如何获取的？**

- 拦截器链就是每一个通知方法又被包装为方法拦截器,利用MethodInterceptor机制

- 1.List<Object> interceptorList = new ArrayList<>(advisors.length); 保存所有拦截器

  - 一个默认的，和四个我们自己定义的增强器

- ```java
  2.遍历所有的增强器,将其转为Interceptor
  for (Advisor advisor : advisors) {
      // Add it conditionally.
      Interceptor[] interceptors = registry.getInterceptors(advisor);
  }
  ```

-     3.将增强器转为List<MethodInterceptor>
          如果是MethodInterceptor,直接加入集合中
          如果不是,使用AdvisorAdapter将增强器转为MethodInterceptor
          转换完成,返回return interceptors.toArray(new MethodInterceptor[0]);

**没有拦截器链的话，直接调用方法就返回了，如果有的话，进行下面步骤**

```java
// We need to create a method invocation... retVal就是方法返回的值
retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();
```

### 拦截器链的触发过程

    1)如果没有拦截器或者当前拦截器的索引和拦截器数组的长度-1一样
    	(this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1)
    	直接执行目标方法return invokeJoinpoint(){return method.invoke(target, args);}
    2)链式获取每一个拦截器,拦截器执行invoke方法,每一个拦截器等待下一个拦截器执行完成返回以后再来执行
    	拦截器链的机制，保证通知方法与目标方法的执行顺序

方法总览。

```java
//currentInterceptorIndex : 当前执行到拦截器链的哪个位置了
//interceptorsAndDynamicMethodMatchers: 保存了拦截方法
	@Override
	@Nullable
	public Object proceed() throws Throwable {
		// We start with an index of -1 and increment early.
		if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
			return invokeJoinpoint();
		}

		Object interceptorOrInterceptionAdvice =
				this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
		if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
			// Evaluate dynamic method matcher here: static part will already have
			// been evaluated and found to match.
			InterceptorAndDynamicMethodMatcher dm =
					(InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
			Class<?> targetClass = (this.targetClass != null ? this.targetClass : this.method.getDeclaringClass());
			if (dm.methodMatcher.matches(this.method, targetClass, this.arguments)) {
				return dm.interceptor.invoke(this);
			}
			else {
				// Dynamic matching failed.
				// Skip this interceptor and invoke the next in the chain.
				return proceed();
			}
		}
		else {
			// It's an interceptor, so we just invoke it: The pointcut will have
			// been evaluated statically before this object was constructed.
			return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
		}
	}
```

### AspectJAfterThrowingAdvice.invoke

若有异常，那么执行@AfterThrowing标注的方法，否则正常返回。

```java
	@Override
	public Object invoke(MethodInvocation mi) throws Throwable {
		try {
			return mi.proceed();
		}
		catch (Throwable ex) {
			if (shouldInvokeOnThrowing(ex)) {
				invokeAdviceMethod(getJoinPointMatch(), null, ex);
			}
			throw ex;
		}
	}
```

### AfterReturningAdviceInterceptor.invoke

若mi.proceed()没有出现异常，那么执行@AfterReturning标注的方法，否则不执行，而是把异常抛出去。

```java
	@Override
	public Object invoke(MethodInvocation mi) throws Throwable {
		Object retVal = mi.proceed();
		this.advice.afterReturning(retVal, mi.getMethod(), mi.getArguments(), mi.getThis());
		return retVal;
	}
```

### AspectJAfterAdvice.invoke

执行完目标方法后，一定会执行@After标注的方法，因为是finally

```java
	@Override
	public Object invoke(MethodInvocation mi) throws Throwable {
		try {
			return mi.proceed();
		}
		finally {
			invokeAdviceMethod(getJoinPointMatch(), null, null);
		}
	}
```

### AspectJAroundAdvice

```java
	@Override
	public Object invoke(MethodInvocation mi) throws Throwable {
		if (!(mi instanceof ProxyMethodInvocation)) {
			throw new IllegalStateException("MethodInvocation is not a Spring ProxyMethodInvocation: " + mi);
		}
		ProxyMethodInvocation pmi = (ProxyMethodInvocation) mi;
		ProceedingJoinPoint pjp = lazyGetProceedingJoinPoint(pmi);
		JoinPointMatch jpm = getJoinPointMatch(pmi);
		return invokeAdviceMethod(pjp, jpm, null, null);
	}
```

### MethodBeforeAdviceInterceptor.invoke

在目标方法执行前，执行@Before标注的方法。

```java
	@Override
	public Object invoke(MethodInvocation mi) throws Throwable {
		this.advice.before(mi.getMethod(), mi.getArguments(), mi.getThis());
		return mi.proceed();
	}
```

### 拦截器链遍历完毕，真正执行目标方法

```java
// We start with an index of -1 and increment early.
if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
    return invokeJoinpoint();
}
```

### 在此之后，一层一层的返回

### 链式调用图解

![image-20220124134244463](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220124134244463.png)

## 7.AOP总结

```
 1.@EnableAspectJAutoProxy开启AOP功能
 2.@EnableAspectJAutoProxy会给容器中注册一个组件AnnotationAwareAspectJAutoProxyCreator
 3.AnnotationAwareAspectJAutoProxyCreator是一个后置处理器
 4.容器创建流程:
     1.registerBeanPostProcessors(beanFactory);注册后置处理器,创建AnnotationAwareAspectJAutoProxyCreator
     2.finishBeanFactoryInitialization(beanFactory);初始化剩下的单实例bean
         1)创建业务逻辑组件和切面组件
         2)AnnotationAwareAspectJAutoProxyCreator拦截组件的创建过程
         3)组件创建完之后,判断组件是否需要增强,
             是:切面的通知方法,包装成增强器(Advisor);给业务逻辑组件创建一个代理对象
 5.执行目标方法
```



## 8.Spring AOP 5的改动

Spring 5.2.7.RELEASE之后，拦截器链改动了

![image-20220126142548143](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220126142548143.png)

### 方法总览，与之前的没有变化，就是拦截器链的顺便变了

```java
	@Override
	@Nullable
	public Object proceed() throws Throwable {
		// We start with an index of -1 and increment early.
		if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
			return invokeJoinpoint();
		}

		Object interceptorOrInterceptionAdvice =
				this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
		if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
			// Evaluate dynamic method matcher here: static part will already have
			// been evaluated and found to match.
			InterceptorAndDynamicMethodMatcher dm =
					(InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
			Class<?> targetClass = (this.targetClass != null ? this.targetClass : this.method.getDeclaringClass());
			if (dm.methodMatcher.matches(this.method, targetClass, this.arguments)) {
				return dm.interceptor.invoke(this);
			}
			else {
				// Dynamic matching failed.
				// Skip this interceptor and invoke the next in the chain.
				return proceed();
			}
		}
		else {
			// It's an interceptor, so we just invoke it: The pointcut will have
			// been evaluated statically before this object was constructed.
			return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
		}
	}
```



### AspectJAroundAdvice

```java
	@Override
	public Object invoke(MethodInvocation mi) throws Throwable {
		if (!(mi instanceof ProxyMethodInvocation)) {
			throw new IllegalStateException("MethodInvocation is not a Spring ProxyMethodInvocation: " + mi);
		}
		ProxyMethodInvocation pmi = (ProxyMethodInvocation) mi;
		ProceedingJoinPoint pjp = lazyGetProceedingJoinPoint(pmi);
		JoinPointMatch jpm = getJoinPointMatch(pmi);
		return invokeAdviceMethod(pjp, jpm, null, null);
	}
```

```java
    @Around("pointCut()")
    public Object logAround(ProceedingJoinPoint jp) throws Throwable {
        System.out.println("@Around环绕通知...开始");
        Object retVal = jp.proceed();   //执行到这里然后继续调用拦截器链
        System.out.println("@Around环绕通知...结束");
        return retVal;
    }
```

### MethodBeforeAdviceInterceptor

```java
	@Override
	public Object invoke(MethodInvocation mi) throws Throwable {
		this.advice.before(mi.getMethod(), mi.getArguments(), mi.getThis()); //执行@Before的方法
		return mi.proceed();
	}
```

### AspectJAfterAdvice

```java
	@Override
	public Object invoke(MethodInvocation mi) throws Throwable {
		try {
			return mi.proceed();  
		}
		finally {
			invokeAdviceMethod(getJoinPointMatch(), null, null);
		}
	}
```

### AfterReturningAdviceInterceptor

```java
	@Override
	public Object invoke(MethodInvocation mi) throws Throwable {
		Object retVal = mi.proceed();
		this.advice.afterReturning(retVal, mi.getMethod(), mi.getArguments(), mi.getThis());
		return retVal;
	}
```

### AspectJAfterThrowingAdvice

```java
	@Override
	public Object invoke(MethodInvocation mi) throws Throwable {
		try {
			return mi.proceed();
		}
		catch (Throwable ex) {
			if (shouldInvokeOnThrowing(ex)) {
				invokeAdviceMethod(getJoinPointMatch(), null, ex);
			}
			throw ex;
		}
	}
```

### 上面的拦截方法调用之后，执行目标方法

```java
// We start with an index of -1 and increment early.
if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
    return invokeJoinpoint();
}
```

## 9.AOP4和5的执行顺序对比

![image-20220126143711171](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220126143711171.png)

### Spring5的调用顺序

Spring4没必要看了，知道一下就可

```java
around(){
    doSome();
    {before(); after(); afterReturning(); afterThrowing();}
    doSome();  //若有异常,不会执行,除非自己捕捉异常
}
before(){
    do@Before();
    return mi.proceed();
}
after(){
    try{
        return mi.proceed();
    }
    finally{
        do@Atfer();
    }
}
afterReturning(){
    retVal = mi.proceed();
    do@AtferReturning();
    return retVal;
}
afterThrowing() throws Throwable{
    try{
        return mi.proceed();
    }
    catch(Throwable t){
        do@AfterThrowing();
        throw t;
    }
}
```



