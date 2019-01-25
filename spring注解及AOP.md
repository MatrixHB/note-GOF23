#### 组件注册

bean的xml注册方式

```xml
<bean id="person" class="entities.Person">
    <property name="age" value="18"></property>
    <property name="name" value="zhangsan"></property>
</bean>
```

获取容器及其组件的时候

```java
public void test(){
    ApplicationContext context = new ClassPathXmlApplicationContext("bean.xml");
    Person bean = (Person)context.getBean("person");
}
```

**@Configuration**  告诉Spring这是一个配置类

**@Bean** 在配置类中的方法上添加该注解，给容器中注册一个bean，id默认是方法名

用此注解配置方法，则获取容器及组件的代码改为

```java
public void test(){
    ApplicationContext context = new AnnotationConfigApplicationContext(SpringConfig.class);
    Person bean = (Person)context.getBean("person");
}
```

**@ConponentScan** 包扫描，该包以下的类只要标注了@Controller、@Service、@Repository、@Component就会自动添加到容器中

```xml
<context:component-scan base-package="webmvc"></context:component-scan>
```

```java
//制定扫描组件的规则（排除某个注解的扫描）
@ComponentScan(value="webmvc",excludeFilters={
    @Filter(type=FilterType.ANNOTATION,classes={Controller.class})
})
//只包含某个注解的扫描，注意禁用掉默认的useDefaultFilters
@ComponentScan(value="webmvc",includeFilters={
    @Filter(type=FilterType.ANNOTATION,classes={Controller.class})
},useDefaultFilters = flase)
```

**@Scope**  注解在@Bean上，取值可以是prototype或者singleton

```java
@Scope("prototype")
//1、单实例时，默认ioc容器启动时就会创建对象放进容器中，以后直接获取bean即可
//2、多实例时，获取bean时才会创建对象
```

**@Lazy**  注解在@Bean上，懒加载，针对单实例bean，ioc容器启动时不创建对象，第一次获取时才创建

**@Conditional**  按条件注册bean，制定判断条件是否符合的类

```java
import org.springframework.context.annotation.Condition;
import org.springframework.context.annotation.ConditionContext;
import org.springframework.core.env.Environment;
import org.springframework.core.type.AnnotatedTypeMetadata;

public class LinuxCondition implements Condition {
    @Override
    public boolean matches(ConditionContext conditionContext, AnnotatedTypeMetadata annotatedTypeMetadata) {
        Environment environment = conditionContext.getEnvironment();
        String os = environment.getProperty("os.name");
        if(os.contains("linux")){
            return true;
        }
        return false;
    }
}

//在某个bean上注解，表示在linux环境下才注册该组件
@Condition({LinuxCondition.class})
```

**@Import**  注解在配置类上，快速给容器中导入一个组件，id默认是组件的全类名

```java
@Import(Color.class)
public class SpringConfig{
    @Bean
    ...
}
```

**总结组件注册的四种方式**：

1）、包扫描+组件标注（@Controller/@Service/@Repository/@Component），必要时加过滤规则

2）、@Bean（适合导入第三方包里面的组件）

3）、@Import（快速导入组件）

4）、使用FactoryBean（默认获取的是工厂bean调用getObject方法创建的对象）



#### 属性赋值

**@Value**  可以是基本数值、#{}表达式、或${}取出配置文件中的值

**@PropertySource**  读取外部配置文件（.properties或.yml文件）

```java
@PropertySource(value={"classpath:/person.properties"})
@Configuration
public class SpringConfig{
    ...
}
```



#### 自动装配

Spring利用依赖注入（DI），完成对IOC容器中各个组件的依赖关系赋值

**@Autowired**  默认优先按照类型去容器中寻找对应的组件，如果找到多个相同类型的组件，再按照属性的名称作为id去容器中茶盏

```java
@Autowired(required=false)
//没有在容器中找到组件，则不装配，不报错
```

**@Qualifier("xxx")**  配合@Autowired，指定需要装配的组件的id，而不是按照属性名

**@Primary("xxx")**  配合@Autowired，在容器中有多个相同类型bean时，首选某个进行装配

Spring 还支持**@Resource**和**@Inject**进行自动装配，这两个是Java规范的注解，不是spring注解

```java
//指定名称的装配
@Resource(name="xxxx")
```

@Inject使用时需要依赖javax.inject包，与@Autowired功能一样

@Autowired除了可以标注属性，还可以**标注在方法、构造器、参数上** 

```java
@Autowired
public void setPerson(Person person){
    this.person = person;
}
//Spring容器调用方法，完成赋值
```

自定义组件想要使用Spring容器**底层的一些组件**，可以实现**xxxAware接口** （ApplicationContextAware、BeanFactoryAware等）

```java
/*
 * 可以在某些情况下主动访问容器，主动注入bean
 */
public class SpringContextHolder implements ApplicationContextAware{
    private static ApplicationContext context;
    
    @Override
	public void setApplicationContext(ApplicationContext applicationContext) {
		context = applicationContext;
	}
    
    public static <T> T getBean(Class<T> c){
        return context.getBean(c);
    }
}
```

**@Profile** 根据当前环境（开发环境、测试环境、生产环境），动态地激活和切换一系列bean

可以标注在组件上，也可以标注在配置类上

```java
@Profile("test")    //指定组件在测试环境下才能注册到容器中
@Bean
public DataSource dataSourceTest(){
    ComboPooledDataSource dataSource = new ComboPooledDataSource();
    ...
    return dataSource;
}

@Profile("dev")
@Bean
public DataSource dataSourceDev(){    ...   }

@Profile("prod")
@Bean
public DataSource dataSourceProd(){    ...   }

//如何指定哪个环境被激活？
//1、命令行参数    在VMoptions中添加 -Dspring.profiles.active=dev
//2、代码方式，如下
applicationContext.getEnvironment().setActiveProfiles("dev");
applicationContext.register(SpringConfig.class);
applicationcontext.refresh();
```



#### Bean生命周期

bean的生命周期一般包含“创建--初始化--销毁”

Spring只帮我们管理单例Bean的完整生命周期，对于prototype的bean只负责创建好交给使用者

1）通过**@Bean**指定使用自定义的bean初始化和销毁方法

```java
@Bean(initMethod="init",destroyMethod="destroy")
//init和destroy两个方法写在组件的类中
//何时调用初始化方法？在创建好bean，并且赋值好之后
//何时调用销毁方法？对于单实例，在容器关闭时；对于多实例，容器不管理这个bean，不会调用销毁方法
```

2）Bean直接实现**InitializingBean**和**DisposableBean**接口，重写afterPropertiesSet()方法和destroy()方法

3）采用JSR规范注解 **@PostConstruct**和 **@PreDestroy**，分别在bean属性赋值之后和销毁之前执行方法（标注在方法上）

4）**BeanPostProcessor接口**，提供两个方法，分别在bean初始化之前和之后调用

```java
//按以下顺序执行
populateBean();     //属性赋值
{
    applyBeanPostProcessorBeforeInitialization();    //初始化之前
    invokeInitMethods();
    applyBeanPostProcessorAfterInitialization();     //初始化之后
}
```



#### 动态代理

抽取重复代码进行封装，如日志操作、验证操作等。

待修改的冗余事物代码：

```java
public void updateCustomer(Customer customer){
    try{
        //开启事务（重复代码）
        TransactionManager.beginTransaction();     
        //执行更新操作
        CustomerDao.updateCustomer(customer);
        //提交事务（重复代码）
        TransactionManager.commit();
    }catch(Exception e){
        //回滚事务（重复代码）
        TransactionManager.rollback();
        throw new RuntimeException(e);
    }
}
```

**代理**：在不改变源码的基础上，对已有方法进行增强（类似一个歌手及其经纪人）

**静态代理**：写一个专门的代理类，传入被代理对象，并在调用被代理对象方法前后添加增强代码

**动态代理**：不用专门去写代理类，完全交给工具去生成代理对象

##### 第一类：基于接口的动态代理（JDK提供）

​	要求：被代理类至少实现一个接口

​	创建代理对象的方法：Proxy.newProxyInstance(ClassLoader, Class[], InvocationHandler);

​	参数含义：ClassLoader 类加载器，和被代理对象使用同一个类加载器，一般写为~.getClass().getClassLoader()；Class[] 字节码数组，即被代理类实现的接口（代理类和被代理类具有相同行为），一般写为~.getClass().getInterfaces()；InvocationHandler 是一个接口，用于提供“增强”代码，即代理是要做什么事情，一般要写一个该接口的实现类，可以是匿名内部类

​	需要重写的invoke方法：具有拦截功能，被代理对象的所有方法被调用时会经过此方法

​	invoke方法参数：method 当前执行的方法，通过method.getName()可以获取方法名；args 当前执行方法的参数，args[0]、args[1]等

```java
//比如被代理类BusDataDaoImpl实现了接口BusDataDao
//BusDataDaoImpl类中有个方法是public void deleteBusData(Integer busNumber)
final BusDataDao busDataDaoImpl = new BusDataDaoImpl();
BusDataDao busDataDaoProxy = (BusDataDao)Proxy.newProxyInstance(busDataDaoImpl.getClass().getClassLoader(),busDataDaoImpl.getClass().getInterfaces(), new InvocationHandler() {
           @Override
           public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                   //此处写增强代码
                   Object ret = null;
                   Integer id = (Integer)args[0];
                   if("deleteBusData".equals(method.getName())){
                       if(id<2000 && id>0){
                           ret = method.invoke(busDataDaoImpl, id);   //执行被代理对象的方法
                       }
                   }
                   return ret;
           }
});
busDataDaoProxy.deleteBusData(1500);   //以代理的身份执行原有方法
```

##### 第二类：基于子类的动态代理（第三方CGLib提供）

​	要求：被代理类不能被final修饰；导入第三方库cglib的依赖

​	创建代理对象的方法：Enhancer.create(Class, Callback)

​	参数含义：Class 被代理对象的字节码；Callback 是说明如何代理的一个接口（常使用该接口的子接口MethodInterceptor），一般要写一个该接口的实现类（匿名内部类）

```java
BusDataDaoImpl busDataDaoProxy = (BusDataDaoImpl)Enhancer.create(busDataDaoImpl.getClass(),new MethodIntercetor(){
           @Override
           public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
                   //此处写增强代码
                   ...
           }
})
```

**动态代理应用案例**：连接池在调用conn.close()方法时，会拦截此方法，不直接关闭该连接，而是将连接放回池中

原来的事务代码可以用动态代理实现，如下

```java 
final CustomerService cs = new CustomerServiceImpl();
CustomerService csProxy = (CustomerService)Proxy.newProxyInstance(CustomerServiceImpl.getClass().getClassLoader(),CustomerServiceImpl.getClass().getInterfaces(), new InvocationHandler() {
           @Override
           public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
               try{
                   //开启事务
                   TransactionManager.beginTransaction();     
                   //执行方法
                   Object ret = method.invoke(cs,args);
        		  //提交事务
                   TransactionManager.commit();
                   return ret;
               }catch(Exception e){
                   //回滚事务
                   TransactionManager.rollback();
                   throw new RuntimeException(e);
               }finally{
                   //关闭资源
               }
           }
});
csProxy.updateCustomer(customer);
```



#### Spring AOP

Spring AOP根据目标类是否实现了接口，来决定采用Proxy还是cglib来实现动态代理

**开发阶段（我们做的）：** 

​	1、编写核心业务代码（Controller、Service、Dao）

​	2、把公用代码抽取出来，制作成通知

​	3、在配置文件中，声明切入点与通知间的关系，即切面

**运行阶段（Spring做的）：** 

Spring监控切入点方法的执行，一旦切入点方法被执行，开启动态代理模式，创建目标对象的代理对象，在方法的对应位置将通知的相应功能织入，完成完整的逻辑代码运行

**AOP术语：**

JoinPoint 连接点：被拦截到的点，在Spring中指的是方法

PointCut 切入点：JoinPoint中，需要被增强的JoinPoint是切入点

Advice 通知/增强：拦截到JoinPoint之后要做的事情（增强代码写在哪个类里面，这个类就是Advice），包括前置通知、后置通知、异常通知、最终通知、环绕通知等

Target 目标对象

Weaving 织入：把增强应用到目标对象来创建新的代理对象的过程，spring采用动态代理织入，而AspectJ采用编译期织入或类加载期织入

Proxy 代理对象

Aspect 切面：切入点和通知的结合

#### 基于xml的AOP配置：

 	1、将通知类的bean交给spring管理

```xml
<bean id="loggerAdvice" class="util.logger"></bean>
```

​	2、导入aop名称空间，使用aop:config开始配置

```xml
<beans xmlns="http://www.springframework.org/schema/beans"  
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"  
        xmlns:aop="http://www.springframework.org/schema/aop"  
        xsi:schemaLocation="http://www.springframework.org/schema/beans  
              http://www.springframework.org/schema/beans/spring-beans.xsd   
              http://www.springframework.org/schema/aop   
              http://www.springframework.org/schema/aop/spring-aop.xsd >  
```

​	3、使用aop:aspect配置切面，order为织入顺序，ref指定通知bean的id

​	4、配置通知的类型（aop:before、aop:after、aop:after-returning、aop:after-throwing、aop:around），指定增强的方法何时执行，method为通知类中的增强方法名，pointcut用于指定切入点表达式

```xml
<aop:config>
    <aop:aspect id="loggerAspect" order="1" ref="loggerAdvice">
        <aop:before method="printLog" pointcut="execution(public void dao.impl.BusDataDaoImpl.deleteBusData(java.lang.Integer busNumber))"></aop:before>
    </aop:aspect>
</aop:config>
```

**切入点表达式**

​	1、全匹配写法：execution(访问修饰符 返回值 包名.类名.方法名(参数列表) )，参数列表的引用类型必须是包名.类名

​	2、全通配写法： 访问修饰符可省略；返回值用通配符 * 表示任意返回值；包名可以使用通配符 *，但有几个包就要写几个 *，也可以用 .. 表示当前包及其子包；类名和方法名都可以使用通配符 *；参数列表在有参情况下可以用 *表示任意类型，也可以用 .. 表示有参无参均可，且任意参数类型

```xml
	<!--所有方法都为切入点-->
	<aop:before method="printLog" pointcut="execution(* *..*.*(..))"></aop:before>
	<!--service层的方法为切入点-->
	<aop:after method="printLog" pointcut="execution(* service.impl.*.*(..))"></aop:after>
```

​	3、通用的切入点

```xml
<!--通用的aop:pointcut必须写在aop:aspect之前-->
<aop:pointcut id="pt1" expression="execution(* service.impl.*.*(..))"></aop:pointcut>
<aop:aspect id="loggerAspect" order="1" ref="loggerAdvice">
        <aop:before method="printLog" pointcut-ref="pt1"></aop:before>
</aop:aspect>
```

**通知类型**

​	aop:before 前置通知：切入点方法执行之前

​	aop:after-returning  后置通知：切入点方法正常执行之后

​	aop:after-throwing  异常通知：切入点方法产生异常之后，与后置通知永远只能执行一个

​	aop:after  最终通知：无论切入点方法是否正常执行，都在之后执行 

​	aop:around 环绕通知：需要在环绕通知的方法中明确指定切入点方法的调用，spring提供了ProceedingJoinPoint接口作为环绕通知的方法参数，该接口中有一个方法proceed()，相当于method.invoke方法

```java
//环绕通知的优点：可以在代码中手动控制通知何时执行
public Object aroundAdvice(ProceedingJoinPoint pjp) {
    try{
        //前置通知
        TransactionManager.beginTransaction();     
        //执行方法
        Object ret = pjp.proceed();
        //后置通知
        TransactionManager.commit();
        return ret;
    }catch(Throwable e){
        //异常通知
        TransactionManager.rollback();
        throw new RuntimeException(e);
    }finally{
        //最终通知
    }
}
```

```xml
<!--配置环绕通知-->
<aop:pointcut id="pt1" expression="execution(* dao.impl.*.*(..))"></aop:pointcut>
<aop:aspect id="transactionAspect" ref="transactionAdvice">
        <aop:around method="aroundAdvice" pointcut-ref="pt1"></aop:around>
</aop:aspect>
```

#### 基于注解的aop配置：

```java
@Component     //通知组件加入容器中
@Aspect     //注解配置切面
public class Logger {

    //注解配置切入点
    @Pointcut("execution(* dao.*.*(..))")
    public void pt1(){}

    //注解配置环绕通知
    @Around("pt1()")
    public void aroundLogger(ProceedingJoinPoint pjp){
        try {
            System.out.println("这是一个前置通知");
            pjp.proceed();
            System.out.println("这是一个后置通知");
        } catch (Throwable throwable) {
            System.out.println("这是一个异常通知");
            throwable.printStackTrace();
        } finally {
            System.out.println("这是一个最终通知");
        }
    }
}
```

在非springBoot环境下，还需要在xml文件中开启spring对注解aop的支持

```xml
<aop:aspectj-autoproxy/>
```

或者在配置类上注解@EnableAspectJAutoProxy



#### AOP实现分布式事务

(结合github的Raincat项目学习) 

自定义注解@TxTransaction

```java
//分布式事务发起者
	@TxTransaction
	public String save() {
        Order order = new Order();
        order.setCreateTime(new Date());
        //...
        orderService.save(order);       //订单事务（参与者）(并且这里用了dubbo远程调用)

        Stock stock = new Stock();
        stock.setCreateTime(new Date());
        //...
        stockService.save(stock);       //库存事务（参与者）

        return "success";
    }
```

通过IDEA的Find Usage功能可以发现，@TxManager这个注解在一个切面类中被处理

```java
@Aspect
public abstract class AbstractTxTransactionAspect {

    private TxTransactionInterceptor txTransactionInterceptor;

    public void setTxTransactionInterceptor(final TxTransactionInterceptor txTransactionInterceptor) {
        this.txTransactionInterceptor = txTransactionInterceptor;
    }

    //切入点为注解了@TxTransaction的所有方法
    @Pointcut("@annotation(org.dromara.raincat.core.annotation.TxTransaction)")
    public void txTransactionInterceptor() {
    }

    //拦截了方法后，增加环绕型的通知，使方法以分布式事务执行
    @Around("txTransactionInterceptor()")
    public Object interceptTxTransaction(final ProceedingJoinPoint pjp) throws Throwable {
        return txTransactionInterceptor.interceptor(pjp);
    }
```

##### 一、拦截，增强事务

跟踪源码发现，执行`txTransactionInterceptor.interceptor(pjp)`的内部，会执行到以下代码

```java
public Object invoke(final String transactionGroupId, final ProceedingJoinPoint point) throws Throwable {        
    ...
        //在factroyOf方法中，会根据这个事务是发起者还是参与者（是否有TxGroupId）创建不同类的对象，发起者则返回StartTxTransactionHandler.class，参与者则返回ActorTxTransactionHandler.class
		final Class css = txTransactionFactoryService.factoryOf(info);
   	 	final TxTransactionHandler txTransactionHandler =
                (TxTransactionHandler) SpringBeanUtils.getInstance().getBean(css);
    	return txTransactionHandler.handler(point, info);
}
```

在`txTransactionHandler.handler(point, info)`中，如果是StartTxTransactionHandler.class，会执行`saveTxTransactionGroup` 方法，内部将执行`ctx.writeAndFlush(heartBeat)`方法，即利用Netty通信发送给TxManager（协调者）一个消息（包含标识码为CREATE_GROUP），告诉TxManager创建一个事务组

```java
    public Boolean saveTxTransactionGroup(final TxTransactionGroup txTransactionGroup) {
        HeartBeat heartBeat = new HeartBeat();
        heartBeat.setAction(NettyMessageActionEnum.CREATE_GROUP.getCode());  //创建事务组标识码
        heartBeat.setTxTransactionGroup(txTransactionGroup);
        final Object object = nettyClientMessageHandler.sendTxManagerMessage(heartBeat);
        //object为null，可能出现TxManager宕机、Netty通信异常等
        if (Objects.nonNull(object)) {
            return (Boolean) object;
        }
        return false;
    }
```

```java
//TxManager对消息的处理
	case CREATE_GROUP:
        ...
        //这里saveTxTransactionGroup内部将把事务组信息写入Redis
        success = txManagerService.saveTxTransactionGroup(heartBeat.getTxTransactionGroup());
        ctx.writeAndFlush(buildSendMessage(heartBeat.getKey(), success));
        break;
```

在`txTransactionHandler.handler(point, info)`中，如果是ActorTxTransactionHandler.class，会开启一个新线程，执行`addTxTransaction` 方法，同样将利用Netty发送给TxManager一个包含标识码ADD_TRANSACTION的消息，告诉TxManager添加一个事务到相应事务组

```java
    public Boolean addTxTransaction(final String txGroupId, final TxTransactionItem txTransactionItem) {
        HeartBeat heartBeat = new HeartBeat();
        heartBeat.setAction(NettyMessageActionEnum.ADD_TRANSACTION.getCode());   //添加事务标识码
        TxTransactionGroup txTransactionGroup = new TxTransactionGroup();
        txTransactionGroup.setId(txGroupId);
        txTransactionGroup.setItemList(Collections.singletonList(txTransactionItem));
        heartBeat.setTxTransactionGroup(txTransactionGroup);
        final Object object = nettyClientMessageHandler.sendTxManagerMessage(heartBeat);
        if (Objects.nonNull(object)) {
            return (Boolean) object;
        }
        return false;
    }
```

```java
//TxManager对消息的处理
	case ADD_TRANSACTION:
        final List<TxTransactionItem> itemList = txTransactionGroup.getItemList();
        if (CollectionUtils.isNotEmpty(itemList)) {
            String modelName = ctx.channel().remoteAddress().toString();  
            final TxTransactionItem item = itemList.get(0);
            item.setModelName(modelName);       //消息来自于哪个模块                 
            //item中的其他信息，status、txGroupId等均在发送端就设定好了
            success = txManagerService.addTxTransaction(txTransactionGroup.getId(), item);
            ctx.writeAndFlush(buildSendMessage(hb.getKey(), success));
        }
        break;
```

客户端（也就是各个微服务模块）和服务端（TxManager）的Netty通信相应的Handler分别写在NettyClientMessageHandler和NettyServerMessageHandler

##### 二、执行原始方法、预提交

参与者执行完`addTxTransaction` 方法会返回一个boolean success，然后

```java
final String waitKey = IdWorkerUtils.getInstance().createTaskKey();   
//waitKey很重要，用来标识要唤醒哪一个线程
final BlockTask waitTask = BlockTaskHelper.getInstance().getTask(waitKey);
...
if (success) {
    //发起调用
    final Object res = point.proceed();
    //唤醒主线程，使事务发起者可以继续往下走
    task.signal();
    ...
    //线程阻塞，等待TxManager将事务组全部事务进行提交检验后，再被唤醒
    waitTask.await();
    ...
}
```

两个参与者线程阻塞之后，发起者这边`point.proceed()`执行完毕了，于是发起者向TxManager发起预提交的请求，告诉TxManager检验几个事务是否都可以提交

```java
final Boolean commit = txManagerMessageService.preCommitTxTransaction(groupId);
```

```java
    public Boolean preCommitTxTransaction(final String txGroupId) {
        HeartBeat heartBeat = new HeartBeat();
        heartBeat.setAction(NettyMessageActionEnum.PRE_COMMIT.getCode());
        TxTransactionGroup txTransactionGroup = new TxTransactionGroup();
        txTransactionGroup.setStatus(TransactionStatusEnum.PRE_COMMIT.getCode());
        txTransactionGroup.setId(txGroupId);
        heartBeat.setTxTransactionGroup(txTransactionGroup);
        final Object object = nettyClientMessageHandler.sendTxManagerMessage(heartBeat);
        if (Objects.nonNull(object)) {
            return (Boolean) object;
        }
        return false;
    }
```

```java
//TxManager对消息的处理
	case PRE_COMMIT:
        ctx.writeAndFlush(buildSendMessage(heartBeat.getKey(), true));
        txTransactionExecutor.preCommit(txTransactionGroup.getId());
        break;
```

TxManager如果通知可以提交，则

```java
    if (commit) {
        //提交事务
        platformTransactionManager.commit(transactionStatus);
        LOGGER.info("发起者提交本地事务");

        //用一个异步线程，通知txManager完成事务
        CompletableFuture.runAsync(() -> txManagerMessageService.asyncCompleteCommit(groupId, 	waitKey, TransactionStatusEnum.COMMIT.getCode(), res));

    } 
```

##### 三、ScheduleFuture

每个线程在task.await()即阻塞之前，都会先开一个线程，用于返回错误、无返回、网络异常、超时的情况下仍能唤醒线程

例如`NettyClientMessageHandler.sendTxManagerMessage()`方法中，在sendTask阻塞之前，先创建一个指定延迟时间之后启用的ScheduledFuture，使sendTask不至于永久阻塞，创建方法为`ScheduledExecutorService.schedule(Runnable command, long delay, TimeUnit unit)`  

```java
ctx.writeAndFlush(heartBeat);
final ScheduledFuture<?> schedule =
    ctx.executor().schedule(() -> {
    if (!sendTask.isNotify()) {
        ...
        sendTask.signal();
    }
}, txConfig.getDelayTime(), TimeUnit.SECONDS);
//发送线程在此等待，等txManager是否正确返回（正确返回则唤醒），返回错误或者无返回通过上面的调度线程唤醒
sendTask.await();
```

在参与者线程阻塞（等待提交）之前，也有类似的处理，防止TxManager宕机或网络异常导致事务永远无法提交

```java
final ScheduledFuture<?> scheduledFuture = txTransactionThreadPool.multiScheduled(() -> {
    if (!waitTask.isNotify()) {
        //如果waitTask超时了，就去获取事务组的状态
        final int transactionGroupStatus = txManagerMessageService.findTransactionGroupStatus(info.getTxGroupId());
        if (TransactionStatusEnum.PRE_COMMIT.getCode() == transactionGroupStatus
            || TransactionStatusEnum.COMMIT.getCode() == transactionGroupStatus) {
            //如果事务组是预提交，或者是提交状态
            //表明事务组是成功的，这时候就算超时也应该去提交
            waitTask.setAsyncCall(objects -> TransactionStatusEnum.COMMIT.getCode());
            waitTask.signal();
        } else {
            waitTask.setAsyncCall(objects -> NettyResultEnum.TIME_OUT.getCode());
            waitTask.signal();
        }
    }
}, info.getWaitMaxTime());

//参与者线程在此阻塞，超时则通过上面调度线程唤醒，唤醒之后根据status决定是提交还是回滚
waitTask.await();
final Integer status = (Integer) waitTask.getAsyncCall().callBack();
if (TransactionStatusEnum.COMMIT.getCode() == status) {
    //提交事务
    platformTransactionManager.commit(transactionStatus);
} else {
    //回滚事务
    platformTransactionManager.rollback(transactionStatus);
}
```

这里的await()和signal()实质上是使用java.util.concurrent.locks.Condition类的await()和signal()方法，一般是配合Lock一起使用（wait()和notify()则是配合synchronized关键字一起使用）

```java
	public BlockTask() {
        lock = new ReentrantLock();
        condition = lock.newCondition();
        notify = false;
    }

	public void signal() {
        try {
            lock.lock();
            notify = true;
            condition.signal();
        } finally {
            lock.unlock();
        }
    }

    public void await() {
        try {
            lock.lock();
            if(!isNotify()) {
                condition.await();
            }
        } finally {
            lock.unlock();
        }
    }
```

##### 四、参与者线程唤醒

当Txmanager收到发起者发来的PRE_COMMIT消息时，执行`txTransactionExecutor.preCommit`方法，其内部除了修改redis中的事务组状态，还会执行到一个`doCommit()`方法，该方法用于给客户端发送回一个心跳（包含标识码COMPLETE_COMMIT），表示“好的，那你们提交吧”

```java
heartBeat.setAction(NettyMessageActionEnum.COMPLETE_COMMIT.getCode());
```

客户端收到之后（NettyClientMessageHandler中的channelRead()方法）

```java
    case COMPLETE_COMMIT:
        notify(heartBeat);
        break;
```

```java
    private void notify(final HeartBeat heartBeat) {
        final List<TxTransactionItem> txTransactionItems =
                heartBeat.getTxTransactionGroup().getItemList();
        if (CollectionUtils.isNotEmpty(txTransactionItems)) {
            final TxTransactionItem item = txTransactionItems.get(0);
            //根据item中的TaskKey去寻找相应的阻塞中的线程，从而去唤醒它
            final BlockTask task = BlockTaskHelper.getInstance().getTask(item.getTaskKey());
            task.setAsyncCall(objects -> item.getStatus());
            task.signal();
        }
    }
```

**总结：** 

实现一个2pc中间件，具体功能在raincat-core、raincat-common、raincat-manager等模块，对于实现业务（微服务）的部分，只需要添加相应注解**@TxTransaction** 和指定属性（事务最大等待时间等），开放接口非常简单；TxManager还需要进一步实现高可用，包括集群部署、负载均衡、Netty路由等