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

**注意:** 环绕通知的方法需要返回目标方法执行之后的结果, 即调用 joinPoint.proceed(); 的返回值, 否则会出现空指针异常！！！

 