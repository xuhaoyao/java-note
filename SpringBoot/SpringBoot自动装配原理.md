# SpringBoot自动装配原理

约定大于配置：

- 按约定编程，是一种软件设计范式，减少软件开发人员做决定的数量，获得简化的好处
- 约定好了Maven的结构目录，resources存放资源文件，src-main-java存放源代码，target存放编译生成的类
- 约定配置文件的后缀是properties或yml
- 约定好一些场景下的默认实现，比如要起一个Web应用，就需要Tomcat，那么引入web相关的starter即可，SpringBoot默认帮我们内嵌好一个Tomcat，装配好Spring MVC



- [SpringBoot自动装配原理](#springboot自动装配原理)
  - [自动装配的加速](#自动装配的加速)
    - [按需加载--条件装配](#按需加载--条件装配)
    - [proxyBeanMethods的加速](#proxybeanmethods的加速)
  - [定制化配置例子](#定制化配置例子)
  - [利用自动装配自己实现一个starter](#利用自动装配自己实现一个starter)
    - [包结构](#包结构)
    - [application.yml](#applicationyml)
    - [ThreadPoolProperties](#threadpoolproperties)
    - [ThreadPoolAutoConfiguration](#threadpoolautoconfiguration)
    - [META-INF/spring.factories](#meta-infspringfactories)
    - [测试自己的starter](#测试自己的starter)
  - [总结](#总结)

在没有使用Spring Boot之前

- 若想要使用Spring的事务控制，就需要自己往容器中导入一个事务管理器
- Spring MVC中，需要自己在web.xml中配置DispatcherServlet
- 若想要用文件上传功能，又要自己配一个文件上传解析器 StandardServletMultipartResolver 等等..

使用了Spring Boot

- 一个注解@SpringBootApplication，常规功能什么都配好了。
- 想使用什么场景，找到对应场景的启动器starter,引入后即可【@Conditional注解会生效，这时候自动装配就起作用】
- 自动装配可以简单理解为：**通过注解或者一些简单的配置就能在 Spring Boot 的帮助下实现某块功能。**



```java
@SpringBootConfiguration  //这个也是一个@Configuration
@EnableAutoConfiguration  //重点看这个,@EnableXXX,开启XXX,这里就开启了自动装配
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication{
    
}
```





```java
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration{
    
}
```

**1、@AutoConfigurationPackage**

自动配置包，指定了默认的包扫描规则

```java
@Import(AutoConfigurationPackages.Registrar.class)
public @interface AutoConfigurationPackage{  //给容器导入了一个组件,分析这个组件的作用
    
}

	/**
	 * ImportBeanDefinitionRegistrar to store the base package from the importing
	 * configuration.
	 * ImportBeanDefinitionRegistrar这个接口在《Spring的一些概念》有介绍,就是可以往容器中注册一些BeanDefinition,
	 * BeanDefinition最后会转换为Bean保存在IoC容器中
	 */
	static class Registrar implements ImportBeanDefinitionRegistrar, DeterminableImports {

		@Override
		public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
			register(registry, new PackageImports(metadata).getPackageNames().toArray(new String[0]));
		}

		@Override
		public Set<Object> determineImports(AnnotationMetadata metadata) {
			return Collections.singleton(new PackageImports(metadata));
		}

	}
```

利用Registrar将Boot主程序包下的所有组件注册进来，**为什么controller包下的@Controller我们不用配置包扫描也可以直接用？原因就在这里，@AutoConfigurationPackage这个注解的作用就是这个**

![image-20220128160044883](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220128160044883.png)



**2、@Import(AutoConfigurationImportSelector.class)**

```java
//ImportSelector的作用就是给容器中注册一些组件,返回的String[]数组中每一项就是要注册的组件的全类名
public class AutoConfigurationImportSelector implements ...{
    @Override
	public String[] selectImports(AnnotationMetadata annotationMetadata) {
		if (!isEnabled(annotationMetadata)) {
			return NO_IMPORTS;
		}
        //利用getAutoConfigurationEntry给容器中批量导入一些组件
		AutoConfigurationEntry autoConfigurationEntry = getAutoConfigurationEntry(annotationMetadata);
		return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations());
	}
}
```

![image-20220128161558582](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220128161558582.png)

```
getCandidateConfigurations方法利用工厂加载得到所有的组件【SpringFactoriesLoader】
	Map<String, List<String>> loadSpringFactories
从META-INF/spring.factories位置来加载一个文件
	默认扫描当前系统里面【所有】META-INF/spring.factories位置的文件
	文件里面写死了SpringBoot一启动要加载的所有配置类
```

![image-20220128161923764](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220128161923764.png)

![image-20220128162106035](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220128162106035.png)

```java
@Import(AutoConfigurationImportSelector.class){
    1、利用getAutoConfigurationEntry(annotationMetadata);给容器中批量导入一些组件
    2、调用List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes)获取到所有需	     要导入到容器中的配置类
    3、利用工厂加载 Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader)；得到所有的		  组件
    4、从META-INF/spring.factories位置来加载。
        Enumeration<URL> urls = classLoader.getResources("META-INF/spring.factories");
       默认扫描我们当前系统里面所有META-INF/spring.factories位置的文件
    5、加载的这些自动配置类不能全部生效,按照条件装配规则（@Conditional），最终会按需配置
}
```



## 自动装配的加速

### 按需加载--条件装配

“`spring.factories`中这么多配置，每次启动都要全部加载么？”

- 虽然自动配置启动的时候默认全部加载，按照条件装配原则（@Conditional这个注解）最终会按需配置

![image-20220128162655312](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220128162655312.png)

**如果要让它生效怎么做？**

```java
import org.aspectj.weaver.Advice;  //导入这个类就好了,spring-aspects里面就有

        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-aspects</artifactId>
        </dependency>
```

### proxyBeanMethods的加速

proxyBeanMethods = true,Full模式

proxyBeanMethods = false,Lite模式

- Full模式中，MainConfig在容器中是通过CGLIB动态代理的一个类，它在每次获取bean的时候,都会先去容器中检查是否有这个bean，有的话直接返回使用
- 因此调用pet()方法不会再new一个，而是直接从容器中返回
- 因此这个模式下配置速度较慢，如果配置类中没有组件依赖,推荐proxyBeanMethods = false,加快配置时候的速度
- 从源码里面也可以看到，上面那个Aop的自动配置类，都采用了false的写法【Lite模式】，这时候MainConfig不会被代理

```java
@Configuration(proxyBeanMethods = true) //Full模式
public class MainConfig {

    @Bean
    public User user() {
        User user = new User();
        user.setPet(pet());
        return user;
    }

    @Bean("tom")
    public Pet pet() {
        return new Pet();
    }


    @Bean
    public CharacterEncodingFilter myCharacterEncodingFilter() {
        return new CharacterEncodingFilter();
    }

}
```



## 定制化配置例子

- 直接自己@Bean替换底层的组件
- 去看这个组件是获取的配置文件什么值就去修改
- (在SpringBoot自动配置包下找对应的类,看prefix=什么,想配什么属性,看对应的属性叫什么)
  - xxxxxAutoConfiguration ---> 组件  ---> xxxxProperties里面拿值  ----> application.properties

![image-20220128170629250](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220128170629250.png)

![image-20220128170647923](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220128170647923.png)

**可以看到，默认编码是UTF8，想改成GBK怎么改？**

> 看下图，直接在一个SpringBoot的配置文件中写上server.servlet.encoding.charset=GBK即可

![image-20220128170829920](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220128170829920.png)



## 利用自动装配自己实现一个starter

### 包结构

![image-20220128202112009](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220128202112009.png)

### application.yml

```yaml
#自定义线程池配置
mythread:
  pool:
    core-pool-size: 5
    maximum-pool-size: 20
    keep-alive-time: 10
    queue-size: 10000
```

### ThreadPoolProperties

```java
@ConfigurationProperties(prefix = "mythread.pool")
public class ThreadPoolProperties {

    Integer corePoolSize = 5;
    Integer maximumPoolSize = 20;
    Integer keepAliveTime = 10;
    Integer queueSize = 10000;
    
    //getter and setter要写..
}
```

### ThreadPoolAutoConfiguration

```java
@Configuration
@ConditionalOnClass(ThreadPoolExecutor.class)    //配置线程池,必须要有这个类,由于jdk自带,所以肯定有
@EnableConfigurationProperties({ThreadPoolProperties.class}) //1.开启属性配置 2.将ThreadPoolProperties导入容器中
public class ThreadPoolAutoConfiguration {

    @Bean
    public ExecutorService pool(ThreadPoolProperties properties){
        return new ThreadPoolExecutor(
                properties.getCorePoolSize(),
                properties.getMaximumPoolSize(),
                properties.getKeepAliveTime(),
                TimeUnit.SECONDS,
                new LinkedBlockingQueue<>(properties.getQueueSize()),
                Executors.defaultThreadFactory(),
                new ThreadPoolExecutor.AbortPolicy()
        );
    }

}
```

### META-INF/spring.factories

在工程的resources包下创建`META-INF/spring.factories`文件

```java
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.scnu.config.ThreadPoolAutoConfiguration  
```



### 测试自己的starter

新建一个工程，引入自己的starter【引入一个场景，自己的线程池】

```xml
        <dependency>
            <groupId>com.scnu</groupId>
            <artifactId>theadpool-springboot-mystart</artifactId>
            <version>0.0.1-SNAPSHOT</version>
        </dependency>
```

![image-20220128203522824](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220128203522824.png)



## 总结

​	Spring Boot 通过`@EnableAutoConfiguration`开启自动装配，通过 SpringFactoriesLoader 最终加载`META-INF/spring.factories`中的自动配置类实现自动装配，自动配置类其实就是通过`@Conditional`按需加载的配置类，想要其生效必须引入`spring-boot-starter-xxx`包实现起步依赖 