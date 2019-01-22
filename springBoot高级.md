### 1 springBoot与缓存

缓存的必要性：1）热点数据，避免频繁访问数据库；2）临时性数据，一段时间后失效，例如验证码

#### **1、JSR07缓存规范** 

由J2EE 提出，现在用的比较少

定义了五个核心接口：CachingProvider、CacheManager、Cache（类似map数据结构，一个cache仅由一个CacheManager管理）、Entry（存储在cache中的键值对）、Expiry（Entry的有效期）

#### **2、spring缓存抽象**

基本接口与注解

|                | 功能                                                         |
| -------------- | ------------------------------------------------------------ |
| Cache          | 缓存接口，定义缓存操作，实现有：RedisCache、ConcurrentMapCache等 |
| CacheManager   | 缓存管理器，管理各种缓存（Cache）组件                        |
| @Cacheable     | 针对方法配置，根据方法的请求参数对其结果进行缓存             |
| @CacheEvict    | 清空缓存                                                     |
| @CachePut      | 更新缓存，保证方法被调用，又希望结果被缓存                   |
| @EnableCaching | 开启基于注解的缓存                                           |
| KeyGenerator   | 缓存数据时key生成的策略                                      |
| serialize      | 缓存数据时value序列化策略                                    |

**@Cacheable注解的属性**

```
*  CacheName/value：指定存储缓存组件的名字
*  key：指定缓存数据使用的key。默认是使用方法参数的值，键值对为“方法参数-方法返回值”
        或编写Spel表达式：#a0/#p0         （第一个方法参数）
                         #root.args[0]   （第一个方法参数）
                         #root.mathodName    （方法名字）
*  keyGenerator：key的生成器，自己可以指定key生成器（或使用自定义keyGenerator）
        key/keyGendertor二选一使用
*  cacheManager：指定Cache管理器，或者cacheReslover指定获取解析器
*  condition：指定符合条件的情况下，才缓存；
        例如condition="#a0>1"，即第一个参数大于1的条件下才缓存
*  unless：否定缓存，unless指定的条件为true，方法的返回值就不会被缓存，可以获取到结果进行判断
*  sync：是否使用异步模式，sync为ture时不支持unless       
```

**spring缓存原理** ：

1、CacheAutoConfiguration，导入了多个缓存组件，其中只有SimpleCacheConfiguration生效

2、在SimpleCacheConfiguration配置类中，向容器中注册了ConcurrentMapCacheManager组件

3、在ConcurrentMapCacheManager可以创建和获取ConcurrentMapCache，作用是将数据保存在ConcurrentMap中

```java
public Cache getCache(String name) {
        Cache cache = (Cache)this.cacheMap.get(name);    //每个Cache都有一个对应的名字
        if (cache == null && this.dynamic) {
            synchronized(this.cacheMap) {
                cache = (Cache)this.cacheMap.get(name);
                if (cache == null) {
                    cache = this.createConcurrentMapCache(name);
                    this.cacheMap.put(name, cache);
                }
            }
        }
        return cache;
}
```

**运行流程**：

1、方法运行之前，使用CacheManager按照cacheName或value所指定的名字获取Cache，第一次获取如果不存在相应cache会自己创建

2、去Cache中查找缓存的内容，使用一个key，默认就是方法的参数；

key是使用KeyGenerator生成的，默认使用SimpleKeyGenerator生成key，没有参数 key=new SimpleKey()，如果有一个参数 key=参数值，如果多个参数 key=new SimpleKey(params)；

3、如果没有查到缓存就调用目标方法

4、将目标方法返回的结果，放回缓存中

**@CachePut** ：

注意CachePut若要同步缓存中原有的键值对，指定的key要跟缓存中的key一致（@CachePut默认的key也是方法参数）

**@CacheEvict** :

key=   指定要清除哪个键的数据

allEntries = true   指定清除这个缓存中的所有数据

beforeInvocation=true    方法执行前清空缓存，默认是false，如果程序异常，就不会清除缓存

**@Caching** : 可组合@Cacheable、@CachePut、@CacheEvict

**@CacheConfig**：注解在service类上，可将方法注解的属性统一起来，例如@CacheConfig(CacheNames="XXX")

#### 3、整合Redis

**在docker上安装redis** 

```shell
#拉取redis镜像
sudo docker pull redis
#启动redis
sudo docker run -d -p 6379:6379 --name redis01 redis
```

**引入starter**

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

**配置**

```
spring.redis.host=10.15.194.44
```

Redis的常用五大数据类型

String【字符串】、List【列表】、Set【集合】、Hash【散列】、ZSet【有序集合】

引入starter后，autoConfiguration会导入两个组件：**StringRedisTemplate和RedisTemplate**，前者k-v都是字符串，后者k-v都是对象（两个组件类似于JdbcTemplate）

```
stringRedisTemplate.opsForValue()  --String
stringRedisTemplate.opsForList()  --List
stringRedisTemplate.opsForSet()  --Set
stringRedisTemplate.opsForHash()  --Hash
stringRedisTemplate.opsForZset()  -Zset
```

**编写测试类**

```java
@RunWith(SpringRunner.class)
@SpringBootTest(classes = SpringbootdemoApplication.class)    //不添加此属性会报错
public class SpringbootdemoApplicationTests {

    @Autowired
    RedisTemplate redisTemplate;

    @Autowired
    BusDataDaoMyBatis busDataDaoMyBatis;

    @Test
    public void test01() {
        BusData busData = busDataDaoMyBatis.queryById(2);
        redisTemplate.opsForList().leftPush(2, busData);
    }
}
```

**redis保存对象时需要序列化** ，默认以jdk原生序列化机制保存

```java
public class Employee implements Serializable {
```

采用json序列化机制，添加**自定义的RedisTemplate** 

```java
@Configuration
public class MyRedisConfig {

    public RedisTemplate<String, BusData> busRedisTemplate(RedisConnectionFactory redisConnectionFactory) throws UnknownHostException {
        RedisTemplate<String, BusData> template = new RedisTemplate();
        template.setConnectionFactory(redisConnectionFactory);
        //key仍按照字符串序列化机制
        template.setKeySerializer(RedisSerializer.string());
        //value按照对象json序列化机制
        Jackson2JsonRedisSerializer<BusData> serializer = new Jackson2JsonRedisSerializer<BusData>(BusData.class);
        template.setValueSerializer(serializer);
        return template;
    }
}
```

测试类中加载的是自定义的RedisTemplate

```java
 @Autowired
 RedisTemplate<String, BusData> busRedisTemplate;
```

#### 4、Redis作为缓存

引入redis的starter后，容器中默认生效的是**RedisCacheManager**，原来默认的SimpleCacheManager不生效

RedisCacheManager为我们创建**RedisCache**作为缓存组件，RedisCache通过操作Redis来缓存数据

**自定义RedisCacheManager**从而自定义key-value序列化机制（会覆盖默认配置的RedisCacheManager）

**springBoot1.5到2.0，自定义RedisCacheManager的方式发生变化**

springBoot1.5

```java
@Bean
public RedisCacheManager redisCacheManager(RedisTemplate<String, BusData> busRedisTemplate){
    RedisCacheManager cacheManager = new RedisCacheManager(busRedisTemplate);
    cacheManager.setUsePrefix(true);   //缓存时自动使用cacheNames作为前缀
    return cacheManager;
}
```

springBoot2.0

```java
@Bean
public RedisCacheManager redisCacheManager(RedisConnectionFactory redisConnectionFactory){
        Jackson2JsonRedisSerializer<BusData> serializer = new Jackson2JsonRedisSerializer<BusData>(BusData.class);

        //配置key-value序列化机制
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()     .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(RedisSerializer.string()))      .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(serializer));
        //用builer.build的方式创建RedisCacheManager
        RedisCacheManager cacheManager = RedisCacheManager.builder(redisConnectionFactory).cacheDefaults(config).build();
        return cacheManager;
}
```



### 2 SpringBoot与消息

#### 1、消息

**消息队列的作用**：

1）异步处理（比如用户注册先写入数据库，再写入消息队列之后，就可以响应用户，而发送注册邮件和注册短信的步骤可以从消息队列取出，异步完成）

2）应用解耦（比如订单系统和库存系统的解耦，订单系统将订单信息写入消息队列，库存系统从消息队列中拉取信息再做更新）

3）流量削峰（秒杀业务中的用户请求的限流，先抢占消息队列中的座位，业务再慢慢处理）

**两种模式**：

1）点对点式：消息代理将发送者的消息放入队列（queue）中，一个或多个接收者从队列中获取消息，消息取出一个没一个

2）发布订阅模式：发送者的消息发送到主题（topic），多个接收者监听这个主题，在消息到达时同时接收到消息

**两种消息代理规范** ：

1）JMS：J2EE提出，ActiveMQ是JMS实现

2）AMQP：兼容JMS，RabbitMQ是AMQP实现

|              | JMS                                                          | AMQP                                                         |
| ------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 定义         | Java API                                                     | 网络线级协议                                                 |
| 跨平台       | 否                                                           | 是                                                           |
| 跨语言       | 否                                                           | 是                                                           |
| Model        | (1)、Peer-2-Peer (2)、Pub/Sub                                | (1)、direct exchange (2)、fanout exchange (3)、topic exchange (4)、headers exchange  后三种都是pub/sub ，只是路由机制做了更详细的划分 |
| 支持消息类型 | TextMessage、MapMessage、 ByteMessage、StreamMessage、 ObjectMessage、Message | 由于跨语言平台，所以基于byte[]，通常需要序列化               |

**SpringBoot的支持** 

spring-jms提供了对JMS的支持

spring-rabbit提供了对AMQP的支持

需要创建ConnectionFactory的实现来连接消息代理

提供JmsTemplate、RabbitTemplate来**发送**消息

@JmsListener、@RabbitListener注解在方法上，可以**监听**消息代理发布的消息

@EnableJms、@EnableRabbit**开启支持**

JmsAutoConfiguration、RabbitAutoConfiguratio为springBoot的**自动配置**

#### 2、RabbitMQ核心概念

**Message** ：消息头和消息体组成，消息体是不透明的，而消息头上则是由一系列的可选属性组成，属性：路由键【routing-key】、优先级【priority】、指出消息可能需要持久性存储【delivery-mode】

**Publisher** ：消息的生产者，也是一个向交换器发布消息的客户端应用程序

<u>**Exchange**</u> ：交换器，用来接收生产者发送的消息并将这些消息路由给服务器中的队列（JMS没有）

Exchange的4中类型：direct【默认】点对点，fanout、topic和headers属于发布订阅，不同类型的Exchange转发消息的策略有所区别（headers用不到）

1）direct：根据路由键直接匹配队列，一对一

2）fanout：不经过路由键，直接发送给所有队列，广播模式

3）topic：根据路由键模糊匹配（通配符#和*），多对多

**Queue** ：消息队列，用来保存消息直到发送给消费者，它是消息的容器，也是消息的终点，一个消息可投入一个或多个队列，消息一直在队列里面，等待消费者连接到这个队列将数据取走。

<u>**Binding**</u> ：绑定，队列和交换器之间的关联，多对多关系（JMS没有）

**Connection** ：网络连接，例如TCP连接

**Channel**：信道，多路复用连接中的一条独立的双向数据流通道，信道是建立在真实的TCP连接之内的虚拟连接，AMQP命令都是通过信道发送出去的。发布和接收消息都是信道，减少TCP的开销，复用一条TCP连接。

**Consumer** ：消息的消费者，表示一个从消息队列中取得消息的客户端应用程序

**VirtualHost ** ：虚拟的小型rabbitMQ服务器，相互隔离，拥有独立的队列、交换器、绑定和权限机制

**Broker** ：表示消息队列服务器实体，可包含多个VirtualHost

#### 3、RabbitMQ整合

1）在docker上安装rabbitmq及其管理器（可浏览器访问）

```shell
sudo docker pull rabbitmq:3.7.8-management
```

2）在浏览器上打开rabbitmq管理界面（端口号15672），创建exchange、queue和binding

**3）引入RabbitMQ的启动器**

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-amqp</artifactId>
        </dependency>
```

引入start后，RabbitAutoConfiguration会帮我们自动导入RabbitProperties类（prefix="spring.rabbitmq"），以及自动配置ConnectionFactory、RabbitTemplate（发送和接收消息）、AmqpAdmin（管理功能组件）

4）配置

```properties
spring.rabbitmq.host=10.15.194.44
spring.rabbitmq.username=guest
spring.rabbitmq.password=guest
```

5）编写测试类

```java
    @Autowired
    RabbitTemplate rabbitTemplate;
    
    @Test
    public void mqProducerTest(){
        BusData busData = busDataDaoMyBatis.queryById(2);
        //convertAndSend(exchange, routing key, object)
        rabbitTemplate.convertAndSend("cxlab.direct","rdsgy.busdata",busData);
    }

    @Test
    public void mqConsumerTest(){
        //receiveAndConvert(queue)
        BusData busData = (BusData) rabbitTemplate.receiveAndConvert("rdsgy.busdata");
        System.out.println(busData.toString());
    }
```

**6）更改序列化方式**

默认是jdk原生序列化方式，可以自定义MessageConverter更改为json序列化

```java
@Configuration
public class MyAmqpConfig {

    @Bean
    public MessageConverter messageConverter(){
        return new Jackson2JsonMessageConverter();
    }
}
```

**7）监听消息** 

```java
//注意，同时要在springBootApplication上注解@EnableRabbit  
    @RabbitListener(queues = "rdsgy.busdata")    //一监听到消息就执行此方法
    public void receive(BusData busdata){
        .....
    }
```

8）用**AmqpAdmin**创建、删除Exchange\queue\binding

```java
@Autowired
AmqpAdmin amqpAdmin;

public void createExchange(){
    amqpAdmin.declareExchange(new TopicExchange("cxlab.topic"));
}
public void createQueue(){
    amqpAdmin.declareQueue(new Queue("rdsgy.busdata",true));
}
public void createBind(){
    amqpAdmin.declareBinding(new Binding("rdsgy.busdata", Binding.DestinationType.QUEUE, "cxlab.topic", "rdsgy.#", null));
}
public void deleteExchange(){
    amqpAdmin.deleteExchange("cxlab.topic");
}
```



### 3 SpringBoot与检索

ElasticSearch是一个基于Lucene的搜索服务器，提供了一个分布式多用户能力的全文搜索引擎，用java开发，其增删改查基于Restful Web接口

在docker上运行ES

```shell
#ES是基于JVM运行的，占用内存需要限制
#9200是ES与外部通信端口，9300是ES节点之间通信端口
sudo docker run -e ES_JAVA_OPTS="-Xms256m -Xmx256m" -d -p 9200:9200 -p 9300:9300 --name ES01 elasticsearch
```

ElasticSearch面向文档，json作为序列化格式

**基本概念**：索引-->类型-->文档-->属性（类比于：数据库-->表-->记录-->字段）

#### 入门测试示例

1、插入数据

```json
PUT /megacorp/employee/1
{
    "first_name" : "John",
    "last_name" :  "Smith",
    "age" :        25,
    "about" :      "I love to go rock climbing",
    "interests": [ "sports", "music" ]
}
```

2、检索文档    GET+URI (index+type+id)

```json
GET /megacorp/employee/1
```

3、轻量搜索（ *query-string* 参数 ）

```json
GET /megacorp/employee/_search   (返回所有employee)
```

```
GET /megacorp/employee/_search?q=last_name:Smith
```

4、查询表达式【match查询】

```json
GET /megacorp/employee/_search
{
    "query" : {
        "match" : {
            "last_name" : "Smith"
        }
    }
}
```

5、【match+filter】

```json
GET /megacorp/employee/_search
{
    "query" : {
        "bool": {
            "must": {
                "match" : {
                    "last_name" : "smith" 
                }
            },
            "filter": {
                "range" : {
                    "age" : { "gt" : 30 } 
                }
            }
        }
    }
}
```

这部分是一个range过滤器 ， 它能找到年龄大于 30 的文档，其中 gt 表示大于(great than) 

6、相关性

```json
GET /megacorp/employee/_search
{
    "query" : {
        "match" : {
            "about" : "rock climbing"
        }
    }
}
```

返回结果为

```json
{
   "hits": {
      "total":      2,
      "max_score":  0.16273327,
      "hits": [
         {
            "_score":         0.16273327, 
            "_source": {
               "first_name":  "John",
               "last_name":   "Smith",
               "age":         25,
               "about":       "I love to go rock climbing",
               "interests": [ "sports", "music" ]
            }
         },
         {
            "_score":         0.016878016, 
            "_source": {
               "first_name":  "Jane",
               "last_name":   "Smith",
               "age":         32,
               "about":       "I like to collect rock albums",
               "interests": [ "music" ]
            }
         }
      ]
   }
}
```

ES能在全文属性上搜索 并返回 相关性最强的结果，区别于传统数据库的要么匹配要么不匹配

但如果查询改成短语搜索就只返回第一个结果【match_phrase】

```json
GET /megacorp/employee/_search
{
    "query" : {
        "match_phrase" : {
            "about" : "rock climbing"
        }
    }
}
```

springBoot默认支持两种技术来和ES交互：jest和springData ElasticSearch

#### springBoot整合Jest

引入jest的工具包

```xml
	    <dependency>
            <groupId>io.searchbox</groupId>
            <artifactId>jest</artifactId>
            <version>5.3.3</version>
        </dependency>
```

引入后JestAutoConfiguration生效，还需要配置Jest相关的properties

```properties
spring.elasticsearch.jest.uris=http://10.15.194.44:9200
```

接下来可以利用JestClient与服务器的9200端口进行http通信

创建实体类，可以在某个属性上注解**@JestId**，自动作为文档Id

**编写测试类**

```java
@Autowired
JestClient jestClient;

@Test
public void saveTest() {
    //给Es中索引（保存）一个文档
    Article article = new Article();
    ...
    //构建一个索引，执行它就会保存到Es
    Index index = new Index.Builder(article).index("megacrop").type("article").build();
    try {
        jestClient.execute(index);
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

```java
@Test
public void search(){
    //查询表达式
    String json ="{\n" +
        "    \"query\":{\n" +
        "        \"match\":{\n" +
        "            \"content\":\"hello\"\n" +
        "        }\n" +
        "    }\n" +
        "}";
    //构建搜索操作
    Search search = new Search.Builder(json).addIndex("megacrop").addType("article").build();
    try {
        SearchResult result = jestClient.execute(search);
        System.out.println(result.getJsonString());
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

#### springBoot整合SpringData ElasticSearch

引入启动器

```xml
       <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
        </dependency>>
```

**注意SpringDataES的版本要和ES的版本适配** ，否则可能连接报错

| spring data elasticsearch | elasticsearch |
| ------------------------- | ------------- |
| 3.1.x                     | 6.2.2         |
| 3.0.x                     | 5.5.0         |
| 2.1.x                     | 2.4.0         |
| 2.0.x                     | 2.2.0         |

配置相关的properties

```properties
spring.data.elasticsearch.cluster-name = elasticsearch
spring.data.elasticsearch.cluster-nodes= 10.15.194.44:9300
```

接下来，操作ES有两种方式：**ElasticsearchRepositry**和**ElasticsearchTemplate**



### 4 springBoot与分布式

#### 1、整合dubbo与zookeeper

**dubbo的本质是远程服务调用（RPC）的分布式框架**（告别web service中的WSDL，以服务者和消费者的方式在dubbo上注册）

dubbo能做到透明的远程方法调用，就像调用本地方法一样，只需要简单配置

服务自动注册与发现，不需要写死服务提供方的地址，注册中心**基于接口名**查询服务提供者的IP地址

##### springBoot整合示例

1）docker上安装zookeeper作为注册中心

```shell
docker pull registry.docker-cn.com/library/zookeeper  #官方镜像加速
docker run --name zk01 --restart always -d -p 2181:2181 zookeeper
```

注意：dubbo 2.6 以前的版本引入 `zkclient` 操作 zookeeper 

dubbo 2.6 及以后的版本引入 `curator` 操作 zookeeper 

**2）Provider模块** 

2.1）引入dubbo和zkclient的相关依赖

```xml
<dependency>
    <groupId>com.alibaba.boot</groupId>
    <artifactId>dubbo-spring-boot-starter</artifactId>
    <version>0.1.0</version>
</dependency>

<dependency>
    <groupId>com.github.sgroschupf</groupId>
    <artifactId>zkclient</artifactId>
    <version>0.1</version>
</dependency>
```

**dubbo 2.6以前的版本，引入zkclient操作zookeeper**  

**dubbo 2.6及以后的版本，引入curator操作zookeeper** 

2.2）编写服务接口及其实现类

```java
public interface TicketService {
    public String getTicket();
}
```

```java
@Component        //注册到spring容器中
@Service         //注意这里的service注解是dubbo包下的，用于提供者暴露服务
public class TicketServiceImp implements TicketService {
    @Override
    public String getTicket() {
        ....
    }
}
```

2.3）配置application.yml

```yaml
dubbo:
  application:
    name: provider-ticket
  registry:
    address: zookeeper://10.15.194.44:2181      #注册中心地址
  scan:
    base-packages: service             #扫描哪个包中的服务类
server:
  port: 9001                  #服务提供者运行占用的端口
```

2.4）启动服务提供者，并一直处于运行状态

**3）Consumer模块**

3.1）引入dubbo和zkclient的依赖

3.2）将TickerService的接口复制过来（包名、全类名相同）（具体实现类不复制）

3.3）编写服务调用类

```java
@Service               //这里的service注解是spring包下的，是spring容器中的service组件
public class UserService {

    @Reference        //该注解用于客户端引用服务
    TicketService ticketService;

    public void hello(){
        String ticket = ticketService.getTicket();
        ...
    }
}
```

3.4）配置application.yml

```yaml
dubbo:
  application:
    name: comsumer-ticket
  registry:
    address: zookeeper://192.168.179.131:2181      #注册中心地址
```

3.5）编写测试类，看是否能调用Provider中的TicketServiceImp

**注意：**

**1）、dubbo版本和zookeeper客户端版本不一致会导致注册失败**

**2）、如果provider和consumer之间互传对象，该对象的类一定要序列化，即实现serializable接口**

**3）、在springBootApplication类上注解@EnableDubbo，主要是开启包扫描**

<u>**springBoot整合dubbo的三种方式** ：</u>

1）、导入dubbo的starter，在application.properties中配置属性，【使用@Service暴露服务，使用@Reference引用服务】，在application类上注解@EnableDubbo

2）、**如果想要精确到method级别配置**，保留dubbo的xml配置文件，仍导入dubbo的starter，在application类上使用@ImportResource(location="xxx.xml")使配置文件生效

3）、除xml配置外，也可以采用**java配置方式**，dubbo提供了API用于java配置，同时使用@Service暴露服务，使用@Reference引用服务，在application类上注解@EnableDubbo(scanBasePackages="  ")

```java
import com.alibaba.dubbo.config.ServiceConfig；
import com.alibaba.dubbo.config.registryConfig；

//将一个个组件配置手动加到容器中，dubbo再去扫描这些组件
@Configuration
public class DubboConfiguration {

    @Bean
    public RegistryConfig registryConfig() {
        RegistryConfig registryConfig = new RegistryConfig();
        registryConfig.setAddress("zookeeper://127.0.0.1:2181");
        registryConfig.setClient("curator");
        return registryConfig;
    }
    
    /*
     *<dubbo:service interface="service.UserService" ref="UserServiceImpl"></dubbo:service>
     */
    @Bean
    public ServiceConfig<UserService> userServiceConfig(UserService userService){
    	ServiceConfig<UserService> serviceConfig = new ServiceConfig<>();
    	serviceConfig.setInterface(UserService.class);
    	serviceConfig.setRef(userSerice);      //userService这个对象在容器中，这里会自动注入
    	return serviceConfig;
    }
}
```

#### 2、Dubbo的xml配置

采用spring（非springBoot）整合，推荐采用xml的Schema配置方式，需要在xml文件中加入dubbo名称空间

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
       http://code.alibabatech.com/schema/dubbo http://code.alibabatech.com/schema/dubbo/dubbo.xsd">
    
<!--http://code.alibabatech.com/schema/dubbo是key,
    http://code.alibabatech.com/schema/dubbo/dubbo.xsd是value-->
```

启动IOC容器（ClassPathXmlApplicationContext）时，如果找不到配置xml文件，可能原因是这个文件没有自动生成到target文件夹中

**schema配置官方说明文档**：http://dubbo.apache.org/zh-cn/docs/user/references/xml/introduction.html

dubbo.method.timeout 设置超时时间

dubbo.method.retrice 设置失败后重试次数

dubbo.service.version 设置服务版本号 （对应的是dubbo.reference.version）

dubbo.reference.stub 服务接口的客户端本地代理类，用于在客户端执行本地逻辑，例如本地缓存、权限验证等，该本地代理类的构造函数必须传入远程代理对象

**dubbo配置覆盖关系** ：

相同属性，例如timeout，方法级(dubbo:method)优先，接口级(dubbo:reference / dubbo:service)次之，全局配置(dubbo:consumer / dubbo:provider)再次之

若级别相同，消费方优先，提供方次之

#### 3、Dubbo原理

**dubbo的运作流程**：

1）服务容器（Container）负责启动、加载和运行服务提供者（Provider）

2）Provider在启动时，向注册中心（Registry）注册自己提供的服务

3）服务消费者（Consumer）在启动时，向注册中心订阅自己需要的服务

4）注册中心返回Provider的地址列表给Consumer，如果有变更，将基于长连接推送变更数据给consumer

5）consumer从地址列表中，基于软负载均衡算法，选择一台Provider进行调用，如果失败则更换一台

6）Provider和Consumer在内存中累计调用次数和时间，每分钟发送一次统计数据给监控中心

**dubbo原理分析**：

1）将客户端和服务端的代理（Stub）、网络通信等环节封装成PRC框架

2）基于Netty实现通信

3）Dubbo框架设计原理图（service层 --> RPC层 --> Remoting层）： http://dubbo.apache.org/zh-cn/docs/dev/design.html

4）**dubbo标签解析**：IOC容器启动后，在解析spring配置文件过程中，DubboNamespaceHandler会创建dubbo标签解析器，即DubboBeanDefinitionParser，并调用其中的parse方法来解析application、registry等标签

5）**Provider服务暴露**：如果解析出ServiceBean对象，在IOC刷新完成后，会触发onApplicationEvent方法 --> 执行export()即服务暴露  --> 在ServiceConfig中执行doExportUrlsFor1Protocol()方法 --> Dubbo和Registry分别有各自的protocol，执行protocol.export(invoker)，分别完成服务的开启、服务在注册中心上的注册。

```xml
    <dubbo:registry protocol="zookeeper" address="10.15.194.44:2181"></dubbo:registry>
    <dubbo:protocol name="dubbo" port="20880"></dubbo:protocol>
```

注册表中保存了每个服务的Url地址和相应的执行器invoker

6）**Consumer服务引用**：解析出ReferenceBean对象，执行ReferenceConfig中的init()方法 --> 执行createProxy()方法 -->  执行invoker=refProtocol.refer()方法 --> DubboProtocol负责获取与Provider的连接，RegistryProtocol负责从注册中心获取服务的invoker，并返回 --> invoker代理返回到ReferenceBean

```xml
<!--声明想要调用的服务接口，生成代理对象-->
    <dubbo:reference interface="service.UserService" id="UserService"></dubbo:reference>
```

7）**服务调用** ：满眼都是invoker，invoker实现了真正的服务调用 http://dubbo.apache.org/zh-cn/docs/dev/implementation.html

#### 4、Dubbo高可用

1）zookeeper宕机：全部注册中心宕机后，消费者和服务者仍能通过**本地缓存**通信（直接调用过一次即可）

即使没有注册中心，也能通过**dubbo直连**实现服务调用

```java
//直接告诉调用方法的地址，不通过注册中心查询
@Reference(url = "localhost:20882" )
```

2）**负载均衡**：

基于权重的随机机制（dubbo默认使用）、基于权重的轮询机制、最少活跃数机制（选择响应时间最快的那台）、一致性哈希机制

```java
//负载均衡改为轮询方式
@Reference(loadbalance = "roundrobin" )
```

权重调节可以在@Service(weight="")中设定，也可以在管理控制台上调节

3）**服务降级**：在服务器压力剧增情况下，对某些服务或页面采取不处理或简单处理的策略，从而释放服务器资源保证核心业务正常运作

在控制台上对某个消费者“屏蔽”，屏蔽后不发起远程调用，直接在客户端返回null

在控制台上对某个消费者“容错”，则发起调用失败后会返回null，不抛异常

4）**集群容错** ：

fail-over：失败后自动切换，设置重试次数

fail-fast：只发起一次调用，失败即报错，用于非幂等性操作（例如新增记录）

fail-safe：调用失败后直接忽略，用于写入日志等操作

fallback：调用失败后，后台记录失败请求，定时重发，用于必须成功的操作（例如消息通知）

forking：并行调用多个服务，一个成功即返回，用于实时性要求较高的读操作，占用资源较多

broadcast：调用所有该服务的提供者，任意一个失败即报错，用于广播所有提供者更新缓存或日志等本地信息

**整合Hystrix**: 导入spring-cloud-starter-netflix-hystrix

#### 5、整合spring cloud

与dubbo不同，spring cloud是一个**分布式整体解决方案**，具有五大开发常用组件

- 服务注册中心 ——Netflix Eureka
- 负载均衡——Netflix Ribbon
- 断路器——Netflix Hystrix （调用链条上某一环调用失败就及时断开）
- 服务网关——Netflix Zuul   （过滤请求）
- 分布式配置——SpringCloud Config

springcloud包含了注册中心，不需要额外的zookeeper

##### springBoot整合示例

**1）注册中心**

1.1）新建模块，选择添加Cloud Discovery -->  Eureka Server 

1.2）配置application.yml

```yaml
server:
  port: 8761           #默认端口8761
eureka:
  instance:
    hostname: eureka-server      #注册中心的主机名
  client:
    register-with-eureka: false     #注册中心不把自己注册到euraka上
    fetch-registry: false           #注册中心本身不从euraka上获取服务的注册信息
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

1.3）在主程序类上注解**@EnableEurekaServer**

1.4）启动程序后，在浏览器可访问localhost:8761，查看eureka界面

**2）Provider模块**

2.1）新建模块，选择添加Cloud discovery --> Eureka Discovery

2.2）配置 application.yml

```yaml
server:
  port: 8001
spring:
  application:
    name: provider-ticket
eureka:
  instance:
    prefer-ip-address: true    #注册服务时使用ip地址进行注册
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

2.3）编写服务类和用于访问的controller

```java
@Service
public class TicketService {
    public String getTicket(){
        ...
    }
}
```

```java
@RestController
public class TicketController {
    @Autowired
    TicketService ticketService;

    @GetMapping("/ticket")
    public String getTicket(){
        return ticketService.getTicket();
    }
}
```

同一个服务可以通过更改端口，在eureka上注册两个实例

**3）consumer模块**

3.1）新建模块，选择添加Cloud discovery --> Eureka Discovery

3.2）配置application.yml

```yaml
spring:
  application:
    name: consumer-ticket
server:
  port: 9001
eureka:
  instance:
    prefer-ip-address: true
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

3.3）编写controller实现远程方法调用

```java
@RestController
public class UserController {
    @Autowired
    RestTemplate restTemplate;
    
    @GetMapping("/buy")
    public String buyTicket(){
        String s = restTemplate.getForObject("http://PROVIDER-TICKET/ticket", String.class);
        ...
    }
}
```

3.4）主程序中开启服务发现

```java
@EnableDiscoveryClient          //开启发现服务功能
@SpringBootApplication
public class ConsumerTicketApplication {

    public static void main(String[] args) {
        SpringApplication.run(ConsumerTicketApplication.class, args);
    }

    @LoadBalanced      //使用负载均衡机制
    @Bean
    public RestTemplate restTemplate(){
        return new RestTemplate();
    }
}
```

浏览器中访问localhost:9001/buy即可调用远程方法



### 5、springBoot与任务

#### 1、异步任务

例如发送邮件、数据同步等任务

@Async注解在方法上，@EnableAsync注解在application或configuration上开启异步功能

#### 2、定时任务

例如每天凌晨分析当天日志的任务

@Scheduled注解在方法上，指定好cron属性

cron属性的格式：秒、分、时、日、月、周几（例如cron="0 * * * * MON-FRI" 是指周一至周五的每一时每一分都执行此任务；cron="0 0 0,6,12,18 * * 6-7"是指周六日的0,6,12,18这四个整点执行此任务）

@EnableScheduling注解在application上，开启定时任务功能

#### 3、邮件任务

加入spring-boot-starter-mail依赖

配置属性

```properties
spring.mail.username = 435856474@qq.com
#qq邮箱提供的第三方登录授权码
spring.mail.password = XXX       
spring.mail.host = smtp.qq.com
#开启SSL服务，安全连接
spring.mail.properties.mail.smtp.ssl.enabled = true
```

编写任务

```java
@Autowired
JavaMailSenderImpl mailSender;

public void mailTest() throws Exception{
        MimeMessage mimeMessage = mailSender.createMimeMessage();
        MimeMessageHelper helper = new MimeMessageHelper(mimeMessage, true);
        //邮件设置
        helper.setSubject("邮件测试");
        helper.setText("<b style='color:red'>这是一封测试邮件，请忽略</b>",true);
        helper.setTo("yibinhaha@163.com");
        helper.setFrom("435856474@qq.com");
        //添加附件
        helper.addAttachment("1.txt",new File("C:\\GY-data\\IeeeIsland.txt"));
        mailSender.send(mimeMessage);
}
```

=======
### 1 springBoot与缓存

缓存的必要性：1）热点数据，避免频繁访问数据库；2）临时性数据，一段时间后失效，例如验证码

#### **1、JSR07缓存规范** 

由J2EE 提出，现在用的比较少

定义了五个核心接口：CachingProvider、CacheManager、Cache（类似map数据结构，一个cache仅由一个CacheManager管理）、Entry（存储在cache中的键值对）、Expiry（Entry的有效期）

#### **2、spring缓存抽象**

基本接口与注解

|                | 功能                                                         |
| -------------- | ------------------------------------------------------------ |
| Cache          | 缓存接口，定义缓存操作，实现有：RedisCache、ConcurrentMapCache等 |
| CacheManager   | 缓存管理器，管理各种缓存（Cache）组件                        |
| @Cacheable     | 针对方法配置，根据方法的请求参数对其结果进行缓存             |
| @CacheEvict    | 清空缓存                                                     |
| @CachePut      | 更新缓存，保证方法被调用，又希望结果被缓存                   |
| @EnableCaching | 开启基于注解的缓存                                           |
| KeyGenerator   | 缓存数据时key生成的策略                                      |
| serialize      | 缓存数据时value序列化策略                                    |

**@Cacheable注解的属性**

```
*  CacheName/value：指定存储缓存组件的名字
*  key：指定缓存数据使用的key。默认是使用方法参数的值，键值对为“方法参数-方法返回值”
        或编写Spel表达式：#a0/#p0         （第一个方法参数）
                         #root.args[0]   （第一个方法参数）
                         #root.mathodName    （方法名字）
*  keyGenerator：key的生成器，自己可以指定key生成器（或使用自定义keyGenerator）
        key/keyGendertor二选一使用
*  cacheManager：指定Cache管理器，或者cacheReslover指定获取解析器
*  condition：指定符合条件的情况下，才缓存；
        例如condition="#a0>1"，即第一个参数大于1的条件下才缓存
*  unless：否定缓存，unless指定的条件为true，方法的返回值就不会被缓存，可以获取到结果进行判断
*  sync：是否使用异步模式，sync为ture时不支持unless       
```

**spring缓存原理** ：

1、CacheAutoConfiguration，导入了多个缓存组件，其中只有SimpleCacheConfiguration生效

2、在SimpleCacheConfiguration配置类中，向容器中注册了ConcurrentMapCacheManager组件

3、在ConcurrentMapCacheManager可以创建和获取ConcurrentMapCache，作用是将数据保存在ConcurrentMap中

```java
public Cache getCache(String name) {
        Cache cache = (Cache)this.cacheMap.get(name);    //每个Cache都有一个对应的名字
        if (cache == null && this.dynamic) {
            synchronized(this.cacheMap) {
                cache = (Cache)this.cacheMap.get(name);
                if (cache == null) {
                    cache = this.createConcurrentMapCache(name);
                    this.cacheMap.put(name, cache);
                }
            }
        }
        return cache;
}
```

**运行流程**：

1、方法运行之前，使用CacheManager按照cacheName或value所指定的名字获取Cache，第一次获取如果不存在相应cache会自己创建

2、去Cache中查找缓存的内容，使用一个key，默认就是方法的参数；

key是使用KeyGenerator生成的，默认使用SimpleKeyGenerator生成key，没有参数 key=new SimpleKey()，如果有一个参数 key=参数值，如果多个参数 key=new SimpleKey(params)；

3、如果没有查到缓存就调用目标方法

4、将目标方法返回的结果，放回缓存中

**@CachePut** ：

注意CachePut若要同步缓存中原有的键值对，指定的key要跟缓存中的key一致（@CachePut默认的key也是方法参数）

**@CacheEvict** :

key=   指定要清除哪个键的数据

allEntries = true   指定清除这个缓存中的所有数据

beforeInvocation=true    方法执行前清空缓存，默认是false，如果程序异常，就不会清除缓存

**@Caching** : 可组合@Cacheable、@CachePut、@CacheEvict

**@CacheConfig**：注解在service类上，可将方法注解的属性统一起来，例如@CacheConfig(CacheNames="XXX")

#### 3、整合Redis

**在docker上安装redis** 

```shell
#拉取redis镜像
sudo docker pull redis
#启动redis
sudo docker run -d -p 6379:6379 --name redis01 redis
```

**引入starter**

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

**配置**

```
spring.redis.host=10.15.194.44
```

Redis的常用五大数据类型

String【字符串】、List【列表】、Set【集合】、Hash【散列】、ZSet【有序集合】

引入starter后，autoConfiguration会导入两个组件：**StringRedisTemplate和RedisTemplate**，前者k-v都是字符串，后者k-v都是对象（两个组件类似于JdbcTemplate）

```
stringRedisTemplate.opsForValue()  --String
stringRedisTemplate.opsForList()  --List
stringRedisTemplate.opsForSet()  --Set
stringRedisTemplate.opsForHash()  --Hash
stringRedisTemplate.opsForZset()  -Zset
```

**编写测试类**

```java
@RunWith(SpringRunner.class)
@SpringBootTest(classes = SpringbootdemoApplication.class)    //不添加此属性会报错
public class SpringbootdemoApplicationTests {

    @Autowired
    RedisTemplate redisTemplate;

    @Autowired
    BusDataDaoMyBatis busDataDaoMyBatis;

    @Test
    public void test01() {
        BusData busData = busDataDaoMyBatis.queryById(2);
        redisTemplate.opsForList().leftPush(2, busData);
    }
}
```

**redis保存对象时需要序列化** ，默认以jdk原生序列化机制保存

```java
public class Employee implements Serializable {
```

采用json序列化机制，添加**自定义的RedisTemplate** 

```java
@Configuration
public class MyRedisConfig {

    public RedisTemplate<String, BusData> busRedisTemplate(RedisConnectionFactory redisConnectionFactory) throws UnknownHostException {
        RedisTemplate<String, BusData> template = new RedisTemplate();
        template.setConnectionFactory(redisConnectionFactory);
        //key仍按照字符串序列化机制
        template.setKeySerializer(RedisSerializer.string());
        //value按照对象json序列化机制
        Jackson2JsonRedisSerializer<BusData> serializer = new Jackson2JsonRedisSerializer<BusData>(BusData.class);
        template.setValueSerializer(serializer);
        return template;
    }
}
```

测试类中加载的是自定义的RedisTemplate

```java
 @Autowired
 RedisTemplate<String, BusData> busRedisTemplate;
```

#### 4、Redis作为缓存

引入redis的starter后，容器中默认生效的是**RedisCacheManager**，原来默认的SimpleCacheManager不生效

RedisCacheManager为我们创建**RedisCache**作为缓存组件，RedisCache通过操作Redis来缓存数据

**自定义RedisCacheManager**从而自定义key-value序列化机制（会覆盖默认配置的RedisCacheManager）

**springBoot1.5到2.0，自定义RedisCacheManager的方式发生变化**

springBoot1.5

```java
@Bean
public RedisCacheManager redisCacheManager(RedisTemplate<String, BusData> busRedisTemplate){
    RedisCacheManager cacheManager = new RedisCacheManager(busRedisTemplate);
    cacheManager.setUsePrefix(true);   //缓存时自动使用cacheNames作为前缀
    return cacheManager;
}
```

springBoot2.0

```java
@Bean
public RedisCacheManager redisCacheManager(RedisConnectionFactory redisConnectionFactory){
        Jackson2JsonRedisSerializer<BusData> serializer = new Jackson2JsonRedisSerializer<BusData>(BusData.class);

        //配置key-value序列化机制
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()     .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(RedisSerializer.string()))      .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(serializer));
        //用builer.build的方式创建RedisCacheManager
        RedisCacheManager cacheManager = RedisCacheManager.builder(redisConnectionFactory).cacheDefaults(config).build();
        return cacheManager;
}
```



### 2 SpringBoot与消息

#### 1、消息

**消息队列的作用**：

1）异步处理（比如用户注册先写入数据库，再写入消息队列之后，就可以响应用户，而发送注册邮件和注册短信的步骤可以从消息队列取出，异步完成）

2）应用解耦（比如订单系统和库存系统的解耦，订单系统将订单信息写入消息队列，库存系统从消息队列中拉取信息再做更新）

3）流量削峰（秒杀业务中的用户请求的限流，先抢占消息队列中的座位，业务再慢慢处理）

**两种模式**：

1）点对点式：消息代理将发送者的消息放入队列（queue）中，一个或多个接收者从队列中获取消息，消息取出一个没一个

2）发布订阅模式：发送者的消息发送到主题（topic），多个接收者监听这个主题，在消息到达时同时接收到消息

**两种消息代理规范** ：

1）JMS：J2EE提出，ActiveMQ是JMS实现

2）AMQP：兼容JMS，RabbitMQ是AMQP实现

|              | JMS                                                          | AMQP                                                         |
| ------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 定义         | Java API                                                     | 网络线级协议                                                 |
| 跨平台       | 否                                                           | 是                                                           |
| 跨语言       | 否                                                           | 是                                                           |
| Model        | (1)、Peer-2-Peer (2)、Pub/Sub                                | (1)、direct exchange (2)、fanout exchange (3)、topic exchange (4)、headers exchange  后三种都是pub/sub ，只是路由机制做了更详细的划分 |
| 支持消息类型 | TextMessage、MapMessage、 ByteMessage、StreamMessage、 ObjectMessage、Message | 由于跨语言平台，所以基于byte[]，通常需要序列化               |

**SpringBoot的支持** 

spring-jms提供了对JMS的支持

spring-rabbit提供了对AMQP的支持

需要创建ConnectionFactory的实现来连接消息代理

提供JmsTemplate、RabbitTemplate来**发送**消息

@JmsListener、@RabbitListener注解在方法上，可以**监听**消息代理发布的消息

@EnableJms、@EnableRabbit**开启支持**

JmsAutoConfiguration、RabbitAutoConfiguratio为springBoot的**自动配置**

#### 2、RabbitMQ核心概念

**Message** ：消息头和消息体组成，消息体是不透明的，而消息头上则是由一系列的可选属性组成，属性：路由键【routing-key】、优先级【priority】、指出消息可能需要持久性存储【delivery-mode】

**Publisher** ：消息的生产者，也是一个向交换器发布消息的客户端应用程序

<u>**Exchange**</u> ：交换器，用来接收生产者发送的消息并将这些消息路由给服务器中的队列（JMS没有）

Exchange的4中类型：direct【默认】点对点，fanout、topic和headers属于发布订阅，不同类型的Exchange转发消息的策略有所区别（headers用不到）

1）direct：根据路由键直接匹配队列，一对一

2）fanout：不经过路由键，直接发送给所有队列，广播模式

3）topic：根据路由键模糊匹配（通配符#和*），多对多

**Queue** ：消息队列，用来保存消息直到发送给消费者，它是消息的容器，也是消息的终点，一个消息可投入一个或多个队列，消息一直在队列里面，等待消费者连接到这个队列将数据取走。

<u>**Binding**</u> ：绑定，队列和交换器之间的关联，多对多关系（JMS没有）

**Connection** ：网络连接，例如TCP连接

**Channel**：信道，多路复用连接中的一条独立的双向数据流通道，信道是建立在真实的TCP连接之内的虚拟连接，AMQP命令都是通过信道发送出去的。发布和接收消息都是信道，减少TCP的开销，复用一条TCP连接。

**Consumer** ：消息的消费者，表示一个从消息队列中取得消息的客户端应用程序

**VirtualHost ** ：虚拟的小型rabbitMQ服务器，相互隔离，拥有独立的队列、交换器、绑定和权限机制

**Broker** ：表示消息队列服务器实体，可包含多个VirtualHost

#### 3、RabbitMQ整合

1）在docker上安装rabbitmq及其管理器（可浏览器访问）

```shell
sudo docker pull rabbitmq:3.7.8-management
```

2）在浏览器上打开rabbitmq管理界面（端口号15672），创建exchange、queue和binding

**3）引入RabbitMQ的启动器**

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-amqp</artifactId>
        </dependency>
```

引入start后，RabbitAutoConfiguration会帮我们自动导入RabbitProperties类（prefix="spring.rabbitmq"），以及自动配置ConnectionFactory、RabbitTemplate（发送和接收消息）、AmqpAdmin（管理功能组件）

4）配置

```properties
spring.rabbitmq.host=10.15.194.44
spring.rabbitmq.username=guest
spring.rabbitmq.password=guest
```

5）编写测试类

```java
    @Autowired
    RabbitTemplate rabbitTemplate;
    
    @Test
    public void mqProducerTest(){
        BusData busData = busDataDaoMyBatis.queryById(2);
        //convertAndSend(exchange, routing key, object)
        rabbitTemplate.convertAndSend("cxlab.direct","rdsgy.busdata",busData);
    }

    @Test
    public void mqConsumerTest(){
        //receiveAndConvert(queue)
        BusData busData = (BusData) rabbitTemplate.receiveAndConvert("rdsgy.busdata");
        System.out.println(busData.toString());
    }
```

**6）更改序列化方式**

默认是jdk原生序列化方式，可以自定义MessageConverter更改为json序列化

```java
@Configuration
public class MyAmqpConfig {

    @Bean
    public MessageConverter messageConverter(){
        return new Jackson2JsonMessageConverter();
    }
}
```

**7）监听消息** 

```java
//注意，同时要在springBootApplication上注解@EnableRabbit  
    @RabbitListener(queues = "rdsgy.busdata")    //一监听到消息就执行此方法
    public void receive(BusData busdata){
        .....
    }
```

8）用**AmqpAdmin**创建、删除Exchange\queue\binding

```java
@Autowired
AmqpAdmin amqpAdmin;

public void createExchange(){
    amqpAdmin.declareExchange(new TopicExchange("cxlab.topic"));
}
public void createQueue(){
    amqpAdmin.declareQueue(new Queue("rdsgy.busdata",true));
}
public void createBind(){
    amqpAdmin.declareBinding(new Binding("rdsgy.busdata", Binding.DestinationType.QUEUE, "cxlab.topic", "rdsgy.#", null));
}
public void deleteExchange(){
    amqpAdmin.deleteExchange("cxlab.topic");
}
```



### 3 SpringBoot与检索

ElasticSearch是一个基于Lucene的搜索服务器，提供了一个分布式多用户能力的全文搜索引擎，用java开发，其增删改查基于Restful Web接口

在docker上运行ES

```shell
#ES是基于JVM运行的，占用内存需要限制
#9200是ES与外部通信端口，9300是ES节点之间通信端口
sudo docker run -e ES_JAVA_OPTS="-Xms256m -Xmx256m" -d -p 9200:9200 -p 9300:9300 --name ES01 elasticsearch
```

ElasticSearch面向文档，json作为序列化格式

**基本概念**：索引-->类型-->文档-->属性（类比于：数据库-->表-->记录-->字段）

#### 入门测试示例

1、插入数据

```json
PUT /megacorp/employee/1
{
    "first_name" : "John",
    "last_name" :  "Smith",
    "age" :        25,
    "about" :      "I love to go rock climbing",
    "interests": [ "sports", "music" ]
}
```

2、检索文档    GET+URI (index+type+id)

```json
GET /megacorp/employee/1
```

3、轻量搜索（ *query-string* 参数 ）

```json
GET /megacorp/employee/_search   (返回所有employee)
```

```
GET /megacorp/employee/_search?q=last_name:Smith
```

4、查询表达式【match查询】

```json
GET /megacorp/employee/_search
{
    "query" : {
        "match" : {
            "last_name" : "Smith"
        }
    }
}
```

5、【match+filter】

```json
GET /megacorp/employee/_search
{
    "query" : {
        "bool": {
            "must": {
                "match" : {
                    "last_name" : "smith" 
                }
            },
            "filter": {
                "range" : {
                    "age" : { "gt" : 30 } 
                }
            }
        }
    }
}
```

这部分是一个range过滤器 ， 它能找到年龄大于 30 的文档，其中 gt 表示大于(great than) 

6、相关性

```json
GET /megacorp/employee/_search
{
    "query" : {
        "match" : {
            "about" : "rock climbing"
        }
    }
}
```

返回结果为

```json
{
   "hits": {
      "total":      2,
      "max_score":  0.16273327,
      "hits": [
         {
            "_score":         0.16273327, 
            "_source": {
               "first_name":  "John",
               "last_name":   "Smith",
               "age":         25,
               "about":       "I love to go rock climbing",
               "interests": [ "sports", "music" ]
            }
         },
         {
            "_score":         0.016878016, 
            "_source": {
               "first_name":  "Jane",
               "last_name":   "Smith",
               "age":         32,
               "about":       "I like to collect rock albums",
               "interests": [ "music" ]
            }
         }
      ]
   }
}
```

ES能在全文属性上搜索 并返回 相关性最强的结果，区别于传统数据库的要么匹配要么不匹配

但如果查询改成短语搜索就只返回第一个结果【match_phrase】

```json
GET /megacorp/employee/_search
{
    "query" : {
        "match_phrase" : {
            "about" : "rock climbing"
        }
    }
}
```

springBoot默认支持两种技术来和ES交互：jest和springData ElasticSearch

#### springBoot整合Jest

引入jest的工具包

```xml
	    <dependency>
            <groupId>io.searchbox</groupId>
            <artifactId>jest</artifactId>
            <version>5.3.3</version>
        </dependency>
```

引入后JestAutoConfiguration生效，还需要配置Jest相关的properties

```properties
spring.elasticsearch.jest.uris=http://10.15.194.44:9200
```

接下来可以利用JestClient与服务器的9200端口进行http通信

创建实体类，可以在某个属性上注解**@JestId**，自动作为文档Id

**编写测试类**

```java
@Autowired
JestClient jestClient;

@Test
public void saveTest() {
    //给Es中索引（保存）一个文档
    Article article = new Article();
    ...
    //构建一个索引，执行它就会保存到Es
    Index index = new Index.Builder(article).index("megacrop").type("article").build();
    try {
        jestClient.execute(index);
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

```java
@Test
public void search(){
    //查询表达式
    String json ="{\n" +
        "    \"query\":{\n" +
        "        \"match\":{\n" +
        "            \"content\":\"hello\"\n" +
        "        }\n" +
        "    }\n" +
        "}";
    //构建搜索操作
    Search search = new Search.Builder(json).addIndex("megacrop").addType("article").build();
    try {
        SearchResult result = jestClient.execute(search);
        System.out.println(result.getJsonString());
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

#### springBoot整合SpringData ElasticSearch

引入启动器

```xml
       <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
        </dependency>>
```

**注意SpringDataES的版本要和ES的版本适配** ，否则可能连接报错

| spring data elasticsearch | elasticsearch |
| ------------------------- | ------------- |
| 3.1.x                     | 6.2.2         |
| 3.0.x                     | 5.5.0         |
| 2.1.x                     | 2.4.0         |
| 2.0.x                     | 2.2.0         |

配置相关的properties

```properties
spring.data.elasticsearch.cluster-name = elasticsearch
spring.data.elasticsearch.cluster-nodes= 10.15.194.44:9300
```

接下来，操作ES有两种方式：**ElasticsearchRepositry**和**ElasticsearchTemplate**



### 4 springBoot与分布式

#### 1、整合dubbo与zookeeper

**dubbo的本质是远程服务调用（RPC）的分布式框架**（告别web service中的WSDL，以服务者和消费者的方式在dubbo上注册）

dubbo能做到透明的远程方法调用，就像调用本地方法一样，只需要简单配置

服务自动注册与发现，不需要写死服务提供方的地址，注册中心**基于接口名**查询服务提供者的IP地址

##### springBoot整合示例

1）docker上安装zookeeper作为注册中心

```shell
docker pull registry.docker-cn.com/library/zookeeper  #官方镜像加速
docker run --name zk01 --restart always -d -p 2181:2181 zookeeper
```

注意：dubbo 2.6 以前的版本引入 `zkclient` 操作 zookeeper 

dubbo 2.6 及以后的版本引入 `curator` 操作 zookeeper 

**2）Provider模块** 

2.1）引入dubbo和zkclient的相关依赖

```xml
<dependency>
    <groupId>com.alibaba.boot</groupId>
    <artifactId>dubbo-spring-boot-starter</artifactId>
    <version>0.1.0</version>
</dependency>

<dependency>
    <groupId>com.github.sgroschupf</groupId>
    <artifactId>zkclient</artifactId>
    <version>0.1</version>
</dependency>
```

**dubbo 2.6以前的版本，引入zkclient操作zookeeper**  

**dubbo 2.6及以后的版本，引入curator操作zookeeper** 

2.2）编写服务接口及其实现类

```java
public interface TicketService {
    public String getTicket();
}
```

```java
@Component        //注册到spring容器中
@Service         //注意这里的service注解是dubbo包下的，用于提供者暴露服务
public class TicketServiceImp implements TicketService {
    @Override
    public String getTicket() {
        ....
    }
}
```

2.3）配置application.yml

```yaml
dubbo:
  application:
    name: provider-ticket
  registry:
    address: zookeeper://10.15.194.44:2181      #注册中心地址
  scan:
    base-packages: service             #扫描哪个包中的服务类
server:
  port: 9001                  #服务提供者运行占用的端口
```

2.4）启动服务提供者，并一直处于运行状态

**3）Consumer模块**

3.1）引入dubbo和zkclient的依赖

3.2）将TickerService的接口复制过来（包名、全类名相同）（具体实现类不复制）

3.3）编写服务调用类

```java
@Service               //这里的service注解是spring包下的，是spring容器中的service组件
public class UserService {

    @Reference        //该注解用于客户端引用服务
    TicketService ticketService;

    public void hello(){
        String ticket = ticketService.getTicket();
        ...
    }
}
```

3.4）配置application.yml

```yaml
dubbo:
  application:
    name: comsumer-ticket
  registry:
    address: zookeeper://192.168.179.131:2181      #注册中心地址
```

3.5）编写测试类，看是否能调用Provider中的TicketServiceImp

**注意：**

**1）、dubbo版本和zookeeper客户端版本不一致会导致注册失败**

**2）、如果provider和consumer之间互传对象，该对象的类一定要序列化，即实现serializable接口**

**3）、在springBootApplication类上注解@EnableDubbo，主要是开启包扫描**

<u>**springBoot整合dubbo的三种方式** ：</u>

1）、导入dubbo的starter，在application.properties中配置属性，【使用@Service暴露服务，使用@Reference引用服务】，在application类上注解@EnableDubbo

2）、**如果想要精确到method级别配置**，保留dubbo的xml配置文件，仍导入dubbo的starter，在application类上使用@ImportResource(location="xxx.xml")使配置文件生效

3）、除xml配置外，也可以采用**java配置方式**，dubbo提供了API用于java配置，同时使用@Service暴露服务，使用@Reference引用服务，在application类上注解@EnableDubbo(scanBasePackages="  ")

```java
import com.alibaba.dubbo.config.ServiceConfig；
import com.alibaba.dubbo.config.registryConfig；

//将一个个组件配置手动加到容器中，dubbo再去扫描这些组件
@Configuration
public class DubboConfiguration {

    @Bean
    public RegistryConfig registryConfig() {
        RegistryConfig registryConfig = new RegistryConfig();
        registryConfig.setAddress("zookeeper://127.0.0.1:2181");
        registryConfig.setClient("curator");
        return registryConfig;
    }
    
    /*
     *<dubbo:service interface="service.UserService" ref="UserServiceImpl"></dubbo:service>
     */
    @Bean
    public ServiceConfig<UserService> userServiceConfig(UserService userService){
    	ServiceConfig<UserService> serviceConfig = new ServiceConfig<>();
    	serviceConfig.setInterface(UserService.class);
    	serviceConfig.setRef(userSerice);      //userService这个对象在容器中，这里会自动注入
    	return serviceConfig;
    }
}
```

#### 2、Dubbo的xml配置

采用spring（非springBoot）整合，推荐采用xml的Schema配置方式，需要在xml文件中加入dubbo名称空间

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
       http://code.alibabatech.com/schema/dubbo http://code.alibabatech.com/schema/dubbo/dubbo.xsd">
    
<!--http://code.alibabatech.com/schema/dubbo是key,
    http://code.alibabatech.com/schema/dubbo/dubbo.xsd是value-->
```

启动IOC容器（ClassPathXmlApplicationContext）时，如果找不到配置xml文件，可能原因是这个文件没有自动生成到target文件夹中

**schema配置官方说明文档**：http://dubbo.apache.org/zh-cn/docs/user/references/xml/introduction.html

dubbo.method.timeout 设置超时时间

dubbo.method.retrice 设置失败后重试次数

dubbo.service.version 设置服务版本号 （对应的是dubbo.reference.version）

dubbo.reference.stub 服务接口的客户端本地代理类，用于在客户端执行本地逻辑，例如本地缓存、权限验证等，该本地代理类的构造函数必须传入远程代理对象

**dubbo配置覆盖关系** ：

相同属性，例如timeout，方法级(dubbo:method)优先，接口级(dubbo:reference / dubbo:service)次之，全局配置(dubbo:consumer / dubbo:provider)再次之

若级别相同，消费方优先，提供方次之

#### 3、Dubbo原理

**dubbo的运作流程**：

1）服务容器（Container）负责启动、加载和运行服务提供者（Provider）

2）Provider在启动时，向注册中心（Registry）注册自己提供的服务

3）服务消费者（Consumer）在启动时，向注册中心订阅自己需要的服务

4）注册中心返回Provider的地址列表给Consumer，如果有变更，将基于长连接推送变更数据给consumer

5）consumer从地址列表中，基于软负载均衡算法，选择一台Provider进行调用，如果失败则更换一台

6）Provider和Consumer在内存中累计调用次数和时间，每分钟发送一次统计数据给监控中心

**dubbo原理分析**：

1）将客户端和服务端的代理（Stub）、网络通信等环节封装成PRC框架

2）基于Netty，**Netty是一个基于NIO的网络应用框架** ，Netty的所有IO操作都是异步非阻塞的（ServerSocketChannel注册到boss的selector，selector是一个多路复用器，轮询accept事件，监听到accept之后建立socketChannel，即与客户端建立连接，同时注册到worker的selector，轮询read事件和write事件）

（与BIO相比，仅当连接就绪或读写就绪时才开启线程，其他时间不出现线程阻塞占用CPU）

3）Dubbo框架设计原理图（service层 --> RPC层 --> Remoting层）： http://dubbo.apache.org/zh-cn/docs/dev/design.html

4）**dubbo标签解析**：IOC容器启动后，在解析spring配置文件过程中，DubboNamespaceHandler会创建dubbo标签解析器，即DubboBeanDefinitionParser，并调用其中的parse方法来解析application、registry等标签

5）**Provider服务暴露**：如果解析出ServiceBean对象，在IOC刷新完成后，会触发onApplicationEvent方法 --> 执行export()即服务暴露  --> 在ServiceConfig中执行doExportUrlsFor1Protocol()方法 --> Dubbo和Registry分别有各自的protocol，执行protocol.export(invoker)，分别完成服务的开启、服务在注册中心上的注册。

```xml
    <dubbo:registry protocol="zookeeper" address="10.15.194.44:2181"></dubbo:registry>
    <dubbo:protocol name="dubbo" port="20880"></dubbo:protocol>
```

注册表中保存了每个服务的Url地址和相应的执行器invoker

6）**Consumer服务引用**：解析出ReferenceBean对象，执行ReferenceConfig中的init()方法 --> 执行createProxy()方法 -->  执行invoker=refProtocol.refer()方法 --> DubboProtocol负责获取与Provider的连接，RegistryProtocol负责从注册中心获取服务的invoker，并返回 --> invoker代理返回到ReferenceBean

```xml
<!--声明想要调用的服务接口，生成代理对象-->
    <dubbo:reference interface="service.UserService" id="UserService"></dubbo:reference>
```

7）**服务调用** ：满眼都是invoker，invoker实现了真正的服务调用 http://dubbo.apache.org/zh-cn/docs/dev/implementation.html

#### 4、Dubbo高可用

1）zookeeper宕机：全部注册中心宕机后，消费者和服务者仍能通过**本地缓存**通信（直接调用过一次即可）

即使没有注册中心，也能通过**dubbo直连**实现服务调用

```java
//直接告诉调用方法的地址，不通过注册中心查询
@Reference(url = "localhost:20882" )
```

2）**负载均衡**：

基于权重的随机机制（dubbo默认使用）、基于权重的轮询机制、最少活跃数机制（选择响应时间最快的那台）、一致性哈希机制

```java
//负载均衡改为轮询方式
@Reference(loadbalance = "roundrobin" )
```

权重调节可以在@Service(weight="")中设定，也可以在管理控制台上调节

3）**服务降级**：在服务器压力剧增情况下，对某些服务或页面采取不处理或简单处理的策略，从而释放服务器资源保证核心业务正常运作

在控制台上对某个消费者“屏蔽”，屏蔽后不发起远程调用，直接在客户端返回null

在控制台上对某个消费者“容错”，则发起调用失败后会返回null，不抛异常

4）**集群容错** ：

fail-over：失败后自动切换，设置重试次数

fail-fast：只发起一次调用，失败即报错，用于非幂等性操作（例如新增记录）

fail-safe：调用失败后直接忽略，用于写入日志等操作

fallback：调用失败后，后台记录失败请求，定时重发，用于必须成功的操作（例如消息通知）

forking：并行调用多个服务，一个成功即返回，用于实时性要求较高的读操作，占用资源较多

broadcast：调用所有该服务的提供者，任意一个失败即报错，用于广播所有提供者更新缓存或日志等本地信息

**整合Hystrix**: 导入spring-cloud-starter-netflix-hystrix

#### 5、整合spring cloud

与dubbo不同，spring cloud是一个**分布式整体解决方案**，具有五大开发常用组件

- 服务注册中心 ——Netflix Eureka
- 负载均衡——Netflix Ribbon
- 断路器——Netflix Hystrix （调用链条上某一环调用失败就及时断开）
- 服务网关——Netflix Zuul   （过滤请求）
- 分布式配置——SpringCloud Config

springcloud包含了注册中心，不需要额外的zookeeper

##### springBoot整合示例

**1）注册中心**

1.1）新建模块，选择添加Cloud Discovery -->  Eureka Server 

1.2）配置application.yml

```yaml
server:
  port: 8761           #默认端口8761
eureka:
  instance:
    hostname: eureka-server      #注册中心的主机名
  client:
    register-with-eureka: false     #注册中心不把自己注册到euraka上
    fetch-registry: false           #注册中心本身不从euraka上获取服务的注册信息
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

1.3）在主程序类上注解**@EnableEurekaServer**

1.4）启动程序后，在浏览器可访问localhost:8761，查看eureka界面

**2）Provider模块**

2.1）新建模块，选择添加Cloud discovery --> Eureka Discovery

2.2）配置 application.yml

```yaml
server:
  port: 8001
spring:
  application:
    name: provider-ticket
eureka:
  instance:
    prefer-ip-address: true    #注册服务时使用ip地址进行注册
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

2.3）编写服务类和用于访问的controller

```java
@Service
public class TicketService {
    public String getTicket(){
        ...
    }
}
```

```java
@RestController
public class TicketController {
    @Autowired
    TicketService ticketService;

    @GetMapping("/ticket")
    public String getTicket(){
        return ticketService.getTicket();
    }
}
```

同一个服务可以通过更改端口，在eureka上注册两个实例

**3）consumer模块**

3.1）新建模块，选择添加Cloud discovery --> Eureka Discovery

3.2）配置application.yml

```yaml
spring:
  application:
    name: consumer-ticket
server:
  port: 9001
eureka:
  instance:
    prefer-ip-address: true
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

3.3）编写controller实现远程方法调用

```java
@RestController
public class UserController {
    @Autowired
    RestTemplate restTemplate;
    
    @GetMapping("/buy")
    public String buyTicket(){
        String s = restTemplate.getForObject("http://PROVIDER-TICKET/ticket", String.class);
        ...
    }
}
```

3.4）主程序中开启服务发现

```java
@EnableDiscoveryClient          //开启发现服务功能
@SpringBootApplication
public class ConsumerTicketApplication {

    public static void main(String[] args) {
        SpringApplication.run(ConsumerTicketApplication.class, args);
    }

    @LoadBalanced      //使用负载均衡机制
    @Bean
    public RestTemplate restTemplate(){
        return new RestTemplate();
    }
}
```

浏览器中访问localhost:9001/buy即可调用远程方法



### 5、springBoot与任务

#### 1、异步任务

例如发送邮件、数据同步等任务

@Async注解在方法上，@EnableAsync注解在application或configuration上开启异步功能

#### 2、定时任务

例如每天凌晨分析当天日志的任务

@Scheduled注解在方法上，指定好cron属性

cron属性的格式：秒、分、时、日、月、周几（例如cron="0 * * * * MON-FRI" 是指周一至周五的每一时每一分都执行此任务；cron="0 0 0,6,12,18 * * 6-7"是指周六日的0,6,12,18这四个整点执行此任务）

@EnableScheduling注解在application上，开启定时任务功能

#### 3、邮件任务

加入spring-boot-starter-mail依赖

配置属性

```properties
spring.mail.username = 435856474@qq.com
#qq邮箱提供的第三方登录授权码
spring.mail.password = XXX       
spring.mail.host = smtp.qq.com
#开启SSL服务，安全连接
spring.mail.properties.mail.smtp.ssl.enabled = true
```

编写任务

```java
@Autowired
JavaMailSenderImpl mailSender;

public void mailTest() throws Exception{
        MimeMessage mimeMessage = mailSender.createMimeMessage();
        MimeMessageHelper helper = new MimeMessageHelper(mimeMessage, true);
        //邮件设置
        helper.setSubject("邮件测试");
        helper.setText("<b style='color:red'>这是一封测试邮件，请忽略</b>",true);
        helper.setTo("yibinhaha@163.com");
        helper.setFrom("435856474@qq.com");
        //添加附件
        helper.addAttachment("1.txt",new File("C:\\GY-data\\IeeeIsland.txt"));
        mailSender.send(mimeMessage);
}
```

