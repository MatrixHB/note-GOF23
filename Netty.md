**Netty是一个基于NIO的网络应用框架** ，Netty的所有IO操作都是异步非阻塞的（ServerSocketChannel注册到boss线程池的selector，selector是一个多路复用器，轮询accept事件，监听到accept之后建立socketChannel，即与客户端建立连接，同时注册到worker线程池的selector，轮询read事件和write事件）

（与BIO相比，仅当连接就绪或读写就绪时才开启线程，其他时间不出现线程阻塞占用CPU）



#### NIO编程

Server类

```java
public class NioServer {
    public static void main(String[] args) throws IOException {

        //1、创建一个通道，并设置为非阻塞模式
        ServerSocketChannel serverChannel = ServerSocketChannel.open();
        serverChannel.configureBlocking(false);

        //2、绑定监听的端口，监听客户端连接
        serverChannel.bind(new InetSocketAddress(8080));

        //3、获取一个选择器，把通道注册到选择器上，监听Accept事件
        Selector selector = Selector.open();
        serverChannel.register(selector, SelectionKey.OP_ACCEPT);

        //4、服务器轮询selector, selector如果有准备就绪的key，则可以进行处理
        while(selector.select() >0){
            Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();
            while(iterator.hasNext()){
                SelectionKey key =  iterator.next();
                handleInput(key);
                iterator.remove();
            }
        }

    }

    public static void handleInput(SelectionKey key) throws IOException{
        //监听到有新的客户端接入
        if(key.isAcceptable()){
            ServerSocketChannel serverChannel = (ServerSocketChannel)key.channel();
            //与客户端三次握手，建立连接通道
            SocketChannel channel = serverChannel.accept();
            //设置为非阻塞
            channel.configureBlocking(false);
            //注册到selector上，并监听读事件
            channel.register(key.selector(),SelectionKey.OP_READ);
        }
        //监听到客户端发来的可读消息
        else if(key.isReadable()){
            SocketChannel channel = (SocketChannel)key.channel();
            ByteBuffer buffer = ByteBuffer.allocate(1024);
            while (channel.read(buffer) >0) {
                buffer.flip();
                byte[] bytes = buffer.array();
                String msg = new String(bytes, "UTF-8");
                System.out.println(msg);
                buffer.clear();
            }
        }
    }
}
```

Client类

```java
public class NioClient {
    public static void main(String[] args) throws IOException {

        SocketChannel channel = SocketChannel.open(new InetSocketAddress(8080));
        channel.configureBlocking(false);

        Scanner sc = new Scanner(System.in);
        while(sc.hasNext()){
            String input = sc.next();
            if("end".equals(input))
                break;
            byte[] bytes = str.getBytes();
            ByteBuffer buffer = ByteBuffer.allocate(bytes.length);
            buffer.put(bytes);
            buffer.flip();
            channel.write(buffer);
            buffer.clear();
        }
        channel.close();       
    }
}
```



#### Netty编程

JDK NIO有ServerSocketChannel、SocketChannel、Selector、SelectionKey几个核心概念。

Netty提供了一个Channel接口统一了对网络的IO操作，其底层的IO通信是交给Unsafe接口实现，而Channel主要负责更高层次的read、write、flush、和ChannelPipeline、Eventloop等组件的交互，以及一些状态的展示。

每个Channel都持有一个ChannelPipeline，通过Channel的读写等IO操作实际上是交由ChannelPipeline处理的，而ChannelPipeline会持有一个ChannelHandlerContext链表，每个Context内部又包含一个ChannelHandler，使用者在Pipeline上添加handler负责逻辑处理，ChannelHandlerContext负责事件的传递流转。

**Server**

1、包括ServerHandler、ServerInitializer和Server启动类，ServerHandler类负责监听到accpet或read事件之后的处理操作，ServerInitializer类负责建立客户端连接后的初始化操作（ChannelPineline、Handler的构建等）

```java
public class ChatServerInitializer extends ChannelInitializer<SocketChannel> {
    @Override
    protected void initChannel(SocketChannel socketChannel) {
        ChannelPipeline pipeline = socketChannel.pipeline();
        pipeline.addLast("decoder",new StringDecoder());
        pipeline.addLast("encoder", new StringEncoder());
        pipeline.addLast("handler", new ChatServerHandler());
    }
}
```

2、Netty在服务端采用主从线程池的线程模型（boss线程池负责连接、登录、验证等事件，worker线程池负责读写事件和其他业务逻辑），都是采用NioEventLoopGroup线程池

3、一个channel会被分配到一个EventLoop（也就是一个Channel只对应一个线程），避免多线程竞争同步问题；一个EventLoop绑定了一个Selector，可以对应多个Channel，处理各个Channel的Handler操作；handler中一般不可以放置耗时的任务，如果有耗时的任务，可以将任务放入自定义的线程池中执行

4、ServerBootStrap启动

```java
ServerBootstrap sb = new ServerBootstrap();
sb.group(boss,worker)                                   //绑定两个线程池
    .channel(NioServerSocketChannel.class)              //指定使用的channel类
    .localAddress(8080)                                 //绑定监听的端口
    .childHandler(new ChatServerInitializer())          //绑定客户端连接时触发的初始化操作
    .childOption(ChannelOption.SO_KEEPALIVE, true);     //指定与客户端连接为长连接
//bind()方法负责绑定端口，并启动服务
ChannelFuture cf = sb.bind().sync();
```

5、由于Netty中的**所有IO操作都是异步的**，因此Netty为了解决调用者如何获取异步操作结果的问题而专门设计了**ChannelFuture**接口。sync()方法表示主线程阻塞（等待异步操作返回结果），addListener()方法用于在监听到异步操作完成之后进行后续处理（监听器模式） 

6、多人聊天采用ChannelGroup存放多个Channel，通过遍历实现消息的广播

**Client** 

包括ClientHandler、ClientInitializer和Client启动类

```java
NioEventLoopGroup group = new NioEventLoopGroup();
Bootstrap bootstrap = new Bootstrap();
bootstrap.group(group).channel(NioSocketChannel.class)
    .remoteAddress("127.0.0.1",8080)          //绑定连接的host和端口
    .handler(new ChatClientInitializer());
```



#### github上的CIM学习笔记

cim-forward-route做的几件事：用户注册、用户登录、服务的发现与更新、负载均衡、与Redis的交互（redisTemplate，包括注册登录时的保存和下线时的清除）

##### 一、依赖版本号的统一管理

在cim-server、cim-client、cim-forward-route等多个模块pom中添加此上层依赖

```xml
    <parent>
        <groupId>com.crossoverjie.netty</groupId>
        <artifactId>cim</artifactId>
        <version>1.0.0-SNAPSHOT</version>
    </parent>
```

版本号还可以用如下方式配置，方便统一更改

```xml
	<properties>
		<junit.version>4.12</junit.version>
		<netty.version>4.1.21.Final</netty.version>
		<logback.version>1.0.13</logback.version>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
	</properties>

	<dependencies>
			<dependency>
				<groupId>io.netty</groupId>
				<artifactId>netty-all</artifactId>
				<version>${netty.version}</version>
			</dependency>
	</dependencies>
```

##### 二、服务的注册与发现（zookeeper）

采用Redis保存服务注册信息的一个最大问题：不能在消费者这边实时更新服务注册信息，每次调用服务时都要去查询Redis

利用Zookeeper的原因：利用其**瞬时节点**的特性（创建znode的客户端与 Zookeeper 保持连接时节点存在，断开时则删除并会有相应的通知 ）

Zookeeper 是一个典型的**观察者模式**

- 由于瞬时节点的特点，消费者可以订阅瞬时节点的父节点。
- 当新增、删除节点时所有的瞬时节点也会自动更新。
- 更新时会给订阅者发起通知告诉最新的节点信息。

这样我们就可以实时获取服务节点的信息，同时也只需要在第一次获取列表时缓存到本地；也不需要频繁和 Zookeeper 产生交互，只用等待通知更新即可；并且不管应用什么原因节点 down 掉后也会在 Zookeeper 中删除该信息。

zookeeper的其他用途：

- 利用**数据变更发送通知**这一特性可以实现**统一配置中心**，再也不需要在每个服务中单独维护配置。
- 利用**瞬时有序节点**还可以实现分布式锁（排序在前的节点拿到锁，访问完资源后删除节点，发出变更通知）。

写一个ZKUtil的类，包含注册、监听等方法：

```java
public class ZkUtil{
    @Autowired
    ZKClient zkclient;
    
    //创建服务节点
    public void createNode(String path) {
        zkClient.createEphemeral(path);
    }
    
    //创建根节点
    public void createRootNode(String path){
        if(!zkClient.exists(path)){
            zkClient.createPersistent(path);
        }
    }
    
    //监听根节点，以便获得服务更新的通知
    public void subscribeEvent(String path){
        zkClient.subscribeChildChanges(path, new IZkChildListener(){
            @Override
            public void handleChildChange(String parentPath, List<String> currentChilds) {
                //监听到服务更新之后做什么
            }
        });
    }
}
```

（cim-server模块中）写一个专门用于在zookeeper上注册服务的线程，服务启动时开启此线程

```java
@SpringBootApplication
public class CIMServerApplication implements CommandLineRunner{
    
     @Override
	public void run(String... args) throws Exception {
         String addr = InetAddress.getLocalHost().getHostAddress();
		new Thread(new RegistryZK(addr, appConfiguration.getCimServerPort(),httpPort)).start();
    }    
}
```

##### 三、本地缓存服务列表

（cim-forward-route模块中）

采用guava的提供的LoadingCache，本质上是Map数据结构

RouteApplication在启动之后，就开启了线程ServerListListener，这个线程用于监听Zookeeper上/root根节点的ChildChanges，在收到zookeeper的更新通知后，本地缓存先清除，再更新

```java
    public void updateCache(List<String> currentChilds) {
        cache.invalidateAll();
        for (String currentChild : currentChilds) {
            String key = currentChild.split("-")[1];
            cache.put(key,key);
        }
    }
```

获取当前所有服务的列表

```java
    public List<String> getAll() {
        List<String> list = new ArrayList<>();
        for (Map.Entry<String, String> entry : cache.asMap().entrySet()) {
            list.add(entry.getKey());
        }
        return list;
    }
```

##### 四、简单的负载均衡

（cim-forward-route模块中）

```java
//轮询法
private AtomicLong index = new AtomicLong();

public String selectServer(){
    List<String> list = getAll();
    if(list.size() == 0){
        throw new RuntimeException("cim当前可用服务列表为空");
    }
    //多个客户端竞争，需要原子操作
    Long position = index.imcrementAndGet()%list.size();
    return list.get(position.intValue());
}
```

**五、路由的真正实现（REST）**

（cim-forward-route模块中的RouteController）

**用户登录的controller** 

```java 
    @RequestMapping(value = "login", method = RequestMethod.POST)
    @ResponseBody()
    public BaseResponse<CIMServerResVO> login(@RequestBody LoginReqVO loginReqVO) throws Exception {
        BaseResponse<CIMServerResVO> res = new BaseResponse();

        //登录校验
        StatusEnum status = accountService.login(loginReqVO);    //调用service层方法
        if (status == StatusEnum.SUCCESS) {
            String server = serverCache.selectServer();      //选择一台服务器
            String[] serverInfo = server.split(":");      //分成ip、port、httpport
            CIMServerResVO vo = new CIMServerResVO(serverInfo[0], Integer.parseInt(serverInfo[1]),Integer.parseInt(serverInfo[2]));
            res.setDataBody(vo);       
            accountService.saveRouteInfo(loginReqVO, server);        //保存路由信息    
        }
        res.setCode(status.getCode());
        res.setMessage(status.getMessage());
        return res;
    }
```

**私聊的controller** 

```java
    @RequestMapping(value = "p2pRoute", method = RequestMethod.POST)
    @ResponseBody()
//NULLBody是一个空对象，用来表示响应不需要DataBody
    public BaseResponse<NULLBody> p2pRoute(@RequestBody P2PReqVO p2pRequest) throws Exception {
        BaseResponse<NULLBody> res = new BaseResponse();
        try {
            //注意p2pRequest中包含发送者Id，接收者Id，消息msg
            //获取“Receiver”用户的路由信息（底层是调用redisTemplate查询）
            CIMServerResVO cimServerResVO = accountService.loadRouteRelatedByUserId(p2pRequest.getReceiveUserId());

            //将聊天消息进行路由（通过http传给相应的server去处理）
            String url = "http://" + cimServerResVO.getIp() + ":" + cimServerResVO.getHttpPort() + "/sendMsg" ;
            ChatReqVO chatVO =new ChatReqVO(p2pRequest.getReceiveUserId(),p2pRequest.getMsg()) ;
            accountService.pushMsg(url,p2pRequest.getUserId(),chatVO);      //推送给server

            res.setCode(StatusEnum.SUCCESS.getCode());
            res.setMessage(StatusEnum.SUCCESS.getMessage());
            
        }catch (CIMException e){
            res.setCode(e.getErrorCode());
            res.setMessage(e.getErrorMessage());
        }
        return res;
    }
```

**群聊的controller** 

不同于单台服务器的“群聊”（收到消息后直接遍历channel并群发），这里不同客户端路由到了不同的服务器，所以需要在路由做“群发”这件事

```java
@RequestMapping(value="groupRoute", method=RequestMethod.POST)
@ResponseBody()
public BaseResponse<NULLBody> groupRoute(@RequestBody ChatReqVO groupReqVO) throws Exception {
    BaseResponse<NULLBody> res = new BaseResponse();

    //获取所有待接收消息的用户的路由信息
    Map<Long, CIMServerResVO> serverResVOMap = accountService.loadRouteRelated();
    //逐一推送http消息到指定的服务器，过滤到发消息的用户自己
    for (Map.Entry<Long, CIMServerResVO> entry : serverResVOMap.entrySet()) {
        Long userId = entry.getKey();
        CIMServerResVO server = entry.getValue();
        if ( !userId.equals(groupReqVO.getUserId())){
            String url = "http://" + server.getIp() + ":" + server.getHttpPort() + "/sendMsg" ;
            ChatReqVO chatVO = new ChatReqVO(userId, groupReqVO.getMsg()) ;
            accountService.pushMsg(url,groupReqVO.getUserId(),chatVO);
        }
    }
    res.setCode(StatusEnum.SUCCESS.getCode());
    res.setMessage(StatusEnum.SUCCESS.getMessage());
    return res;
}
```

这里在loadRouteRelated方法中需要遍历所有UserId与服务实例的关系（也就是在Redis中遍历route_prefix前缀的KV）

```java
    //采用scan命令来遍历（比keys命令性能更好，不会因为数据量很大造成线程阻塞）
	RedisConnection connection = redisTemplate.getConnectionFactory().getConnection();
    ScanOptions options = ScanOptions.scanOptions()
        .match(ROUTE_PREFIX + "*")
        .build();
    Cursor<byte[]> scan = connection.scan(options);

    while (scan.hasNext()) {
        byte[] next = scan.next();
        String key = new String(next, StandardCharsets.UTF_8);
        
        long userId = Long.valueOf(key.split(":")[1]);
        String value = redisTemplate.opsForValue().get(key);      //用户对应的路由服务实例
        String[] server = value.split(":");
        CIMServerResVO cimServerResVO = new CIMServerResVO(server[0], Integer.parseInt(server[1]), Integer.parseInt(server[2]));
        routes.put(userId, cimServerResVO);
    }
```

**六、基于Netty的通信** 

（cim-server模块中）

服务端有对应的controller处理"/sendMsg"请求，然后执行与客户端的基于Netty和protoBuf的通信（ctx.writeAndFlush(...)在此处执行），将请求体中的Msg发送给指定用户或群发（请求体中的UserId）

##### 其他细节：

1、每个类都添加相应的日志

SpringBoot集成日志logback，使用很方便

```java
private static Logger logger = LoggerFactory.getLogger(xxx.class)
```

2、用枚举类型StatusEnum表示登录状态

```java
public enum StatusEnum {

    SUCCESS("9000", "成功"),

    VALIDATION_FAIL("3000", "参数校验失败"),

    FAIL("4000", "失败"),

    REPEAT_LOGIN("5000", "账号重复登录！"),

    OFF_LINE("7000", "你选择的账号不在线，请重新选择！"),

    ACCOUNT_NOT_MATCH("9100", "登录信息不匹配！"),

    REQUEST_LIMIT("6000", "请求限流");

    private final String code;
    private final String message;

    private StatusEnum(String code, String message) {
        this.code = code;
        this.message = message;
    }

    public String getCode() {        return code;    }

    public String getMessage() {        return message;    }
}
```

3、在Redis中保存用户账号、用户路由、已登录用户信息等，前两个均用K-V结构保存，但key以前缀区分（account_prefix、route_prefix），已登录用户信息用set结构保存，达到去重作用