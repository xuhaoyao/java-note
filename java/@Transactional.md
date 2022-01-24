# @Transactional

在要开启事务的方法上使用`@Transactional`注解即可

```java
@Transactional(rollbackFor = Exception.class)
public void save() {
  ......
}
```

我们知道 Exception 分为运行时异常 RuntimeException 和非运行时异常。在`@Transactional`注解中如果不配置`rollbackFor`属性,那么事务只会在遇到`RuntimeException`的时候才会回滚,加上`rollbackFor=Exception.class`,可以让事务在遇到非运行时异常时也回滚。

`@Transactional` 注解一般可以作用在`类`或者`方法`上。

- **作用于类**：当把`@Transactional` 注解放在类上时，表示所有该类的 public 方法都配置相同的事务属性信息。
- **作用于方法**：若类配置了`@Transactional`，方法也配置了`@Transactional`，方法的事务会覆盖类的事务配置信息。



## 事务失效的场景

[一口气说出 6种 @Transactional 注解失效场景 (qq.com)](https://mp.weixin.qq.com/s?__biz=Mzg2OTA0Njk0OA==&mid=2247486483&idx=2&sn=77be488e206186803531ea5d7164ec53&chksm=cea243d8f9d5cacecaa5c5daae4cde4c697b9b5b21f96dfc6cce428cfcb62b88b3970c26b9c2&token=816772476&lang=zh_CN#rd)



## 源码分析例子

环境搭建:
1.导入相关依赖
     数据源，数据库驱动，Spring-jdbc模块
2.配置数据源和JdbcTemplate操作数据
3.给方法上标注@Transactional表示当前方法是一个事务方法
4.@EnableTransactionManagement开启基于注解的事务管理功能
5.配置事务管理器管理事务

```java
@Configuration
@EnableTransactionManagement	//开启基于注解的事务管理功能
@ComponentScan("com.scnu.tx")
public class TxConfig {

    @Bean
    public DataSource dataSource(){
        DruidDataSource dataSource = new DruidDataSource();
        dataSource.setUsername("root");
        dataSource.setPassword("333");
        dataSource.setUrl("jdbc:mysql://localhost:3306/springdb");
        dataSource.setDriverClassName("com.mysql.jdbc.Driver");
        return dataSource;
    }

    @Bean
    public JdbcTemplate jdbcTemplate(){
        //Spring对Configuration类会特殊处理,给容器中加组件的方法,多次调用都是从容器中找组件
        JdbcTemplate jdbcTemplate = new JdbcTemplate(dataSource());
        return jdbcTemplate;
    }

    @Bean
    public TransactionManager transactionManager(){	//事务管理器
        return new DataSourceTransactionManager(dataSource());
    }
}
```

```java
@Service
public class UserServiceImpl implements UserService {

    @Autowired
    private UserDao userDao;
	
    //加上这个注解,出了异常之后就会回滚,前提是要配置好事务管理器，开启基于注解的事务管理功能@EnableTransactionManagement
    @Transactional
    public void insertUser(){
        userDao.insertUser();
        int i = 1/0;
    }

}
@Repository
public class UserDao {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    public void insertUser(){
        String sql = "insert into t_user(username,password) values(?,?)";
        String username = UUID.randomUUID().toString().substring(0,5);
        String pwd = UUID.randomUUID().toString().substring(0,5);
        jdbcTemplate.update(sql,username,pwd);
    }
}

```



## 源码分析

### @EnableTransactionManagement

利用@Import({TransactionManagementConfigurationSelector.class})给容器中注入组件

- AutoProxyRegistrar：加一个BeanPostProcessor,给Bean增强【事务方面】
- ProxyTransactionManagementConfiguration【事务增强方面的功能由它来做，用它来解析@Transactional】
- 也就是说，@EnableTransactionManagement不配置的话，肯定没有事务功能！

<img src="https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220124154612838.png" alt="image-20220124154612838" style="zoom:67%;" />

```java
TransactionManagementConfigurationSelector{
	@Override
	protected String[] selectImports(AdviceMode adviceMode) {
		switch (adviceMode) {
			case PROXY:		//导入两个组件
				return new String[] {AutoProxyRegistrar.class.getName(),
						ProxyTransactionManagementConfiguration.class.getName()};
			case ASPECTJ:
				return new String[] {determineTransactionAspectClass()};
			default:
				return null;
		}
	}
}
```

#### AutoProxyRegistrar

```java
public class AutoProxyRegistrar implements ImportBeanDefinitionRegistrar{
    @Override
	public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry){
        ...
            if (mode == AdviceMode.PROXY) {
                AopConfigUtils.registerAutoProxyCreatorIfNecessary(registry);  //看这里
                if ((Boolean) proxyTargetClass) {
                    AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
                    return;
                }
            }
    }
}
```

会给容器中注册一个InfrastructureAdvisorAutoProxyCreator

- 分析AOP的时候，开启AOP，会给容器注册AnnotationAwareAspectJAutoProxyCreator
- 而开启事务，会给容器注册一个InfrastructureAdvisorAutoProxyCreator
- 它们都是BeanPostProcessor
- 那么大致原理就出来了
  - **InfrastructureAdvisorAutoProxyCreator利用后置处理器机制在对象创建以后,包装对象,返回一个代理对象(增强器),代理对象执行方法利用拦截器链进行调用(与aop分析的情况类似)**

#### ProxyTransactionManagementConfiguration

给容器中注册事务增强器`BeanFactoryTransactionAttributeSourceAdvisor`，完成事务应做的功能。

- 事务增强器要用事务注解的信息，AnnotationTransactionAttributeSource解析@Transactional
- 事务拦截器，TransactionInterceptor【它是一个MethodInterceptor,分析AOP的时候，这个接口就是一个个的增强方法】

##### 解析@Transactional这个注解

AnnotationTransactionAttributeSource用来解析@Transactional()里面的各种参数propagation,readOnly,rollbackFor这些

<img src="https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220124160031876.png" alt="image-20220124160031876"  />

<img src="https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220124160110918.png" alt="image-20220124160110918"  />

##### TransactionInterceptor

注意看第二步，这就是为什么要手动往容器中加入一个事务管理器的原因。

注意看第三步，可以知道@Transactional是一个环绕注解@Around

```java
TransactionInterceptor,它是一个MethodInterceptor,在注入IOC容器前先保存了事务注解的信息,事务管理器
在目标方法执行的时候:
    执行拦截器链
    事务拦截器:
        1)先获取事务的相关属性
            // If the transaction attribute is null, the method is non-transactional.
            TransactionAttributeSource tas = getTransactionAttributeSource();
        2)再获取TransactionManager,如果事先没有添加任何指定的transactionManager,那么从容器里面拿
            defaultTransactionManager = this.beanFactory.getBean(TransactionManager.class);
        3)执行目标方法
            如果异常,获取到事务管理器,利用事务管理器回滚操作
            Object retVal;
            try {
                // This is an around advice: Invoke the next interceptor in the chain.
                // This will normally result in a target object being invoked.
                //这里面的代码很复杂,拿数据库的连接,设置自动提交,开启新事务等等
                retVal = invocation.proceedWithInvocation();  
             }
            catch (Throwable ex) {
               // 如果出了异常,在这里会回滚
               // target invocation exception
                completeTransactionAfterThrowing(txInfo, ex){
                    txInfo.getTransactionManager().rollback(txInfo.getTransactionStatus());
                }
                throw ex;
             }
            finally {
               cleanupTransactionInfo(txInfo);
            }
            //没有异常的话在这里提交
            commitTransactionAfterReturning(txInfo){
                txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
            }
            return retVal;
```