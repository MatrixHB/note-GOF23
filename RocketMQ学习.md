### RocketMQ

（Notify/MetaQ）分布式消息中间件

作用：应用解耦、异步调用、流量削峰、分布式最终一致性

源码核心模块：namesrv、client、broker、store、remoting

突出优势：能保证严格的消息顺序、提供丰富的消息拉取模式、消费者水平扩展能力、亿级消息堆积能力

使用示例：源码中的example模块，同步producer、异步producer、推consumer、拉consumer



### namesrv

作用：1、管理一些KV配置信息；2、管理一些topic、broker的注册信息

**主要组件:**

1、DefaultRequestProcessor：处理Namesrv接收到的请求（processRequest()方法），根据不同的RequestCode调用不同的方法，比如对KV配置表的增删查、对路由信息的增删改查

2、KVConfigManager

3、RouteInfoManager（最重要的管理组件，保存所有的路由信息，包括topicQueueTable、brokerAddrTable、clusterAddrTable、brokerLiveTable、filterServerTable）

4、Processor线程池（processRequest()方法）、scheduled线程（负责定时任务）

**启动过程：**

1、由NamesrvStartup 负责解析命令行的一些参数到各种 Config 对象中（NamesrvConfig/NettyServerConfig等），如果命令行参数中带有配置文件的路径，也会从配置文件中读取配置到各种 Config 对象中，然后**初始化并启动NamesrvController** 

```java
final NamesrvController controller = new NamesrvController(namesrvConfig, nettyServerConfig);
return controller;
```

2、由NamesrvController**初始化并启动各个组件**，包括按照配置创建NettyServer、注册requestProcessor、启动NettyServer、启动各种scheduled任务（NettyRemotingServer启动后就可以接收请求了，其中最主要的请求就是处理路由信息相关的请求，例如broker的注册请求）

```java
public boolean initialize() {
    this.kvConfigManager.load();
    this.remotingServer = new NettyRemotingServer(this.nettyServerConfig, this.brokerHousekeepingService);
    this.remotingExecutor = Executors.newFixedThreadPool( nettyServerConfig.getServerWorkerThreads(), new ThreadFactoryImpl("RemotingExecutorThread_"));
    this.registerProcessor();
    this.scheduledExecutorService.scheduleAtFixedRate(() → {
        ...
    },5, 10, TimeUnit.SECONDS);
}
```

**路由信息的管理：** 

Netty服务端接收到请求后，回调请求处理程序DefaultRequestProcessor，根据请求类型RequestCode，例如注册Broker或者新建Topic请求，来更新RouteInfoManager路由信息

```java
public RemotingCommand processRequest(ChannelHandlerContext ctx,
                                      RemotingCommand request) throws RemotingCommandException {
    switch (request.getCode()) {
        case RequestCode.PUT_KV_CONFIG:
            return this.putKVConfig(ctx, request);
        case RequestCode.GET_KV_CONFIG:
            return this.getKVConfig(ctx, request);
        case RequestCode.DELETE_KV_CONFIG:
            return this.deleteKVConfig(ctx, request);
        case RequestCode.REGISTER_BROKER:
            ...
        case RequestCode.UNREGISTER_BROKER:
            return this.unregisterBroker(ctx, request);
        case RequestCode.GET_ROUTEINTO_BY_TOPIC:
            return this.getRouteInfoByTopic(ctx, request);
        case RequestCode.GET_BROKER_CLUSTER_INFO:
            return this.getBrokerClusterInfo(ctx, request);
        case RequestCode.GET_ALL_TOPIC_LIST_FROM_NAMESERVER:
            return getAllTopicListFromNameserver(ctx, request);
        ...
        default:
            break;
    }
}
```

如果请求码是REGISTER_BROKER，则会执行RouteInfoManager类中的registerBroker()方法（写之前先加锁writeLock），同时返回成功或失败的response

如果请求码是GET_ROUTEINTO_BY_TOPIC，则会执行RouteInfoManager类中的pickupTopicRouteData，先加锁readLock，然后执行`List<QueueData> queueDataList = this.topicQueueTable.get(topic);` 

**心跳检查：** 

使用BrokerHouseKeepingService来处理broker是否存活，如果broker失效、异常或者关闭，则将broker从RouteInfoManager路由信息中移除，同时将与该broker相关的topic信息也一起删。

由于BrokerHouseKeepingService实现了ChannelEventListener接口，因此NameServer的remotingServer在启动时，会专门启动一个线程用于监听Channel的失效、异常或者关闭等的事件队列，当事件队列里面有新事件时，则取出事件并判断事件的类型，然后调用BrokerHouseKeepingService对应的方法来处理该事件。

```java
//NettyRemotingServer类中
public void start() {
    ...
        if (this.channelEventListener != null) {
            this.nettyEventExecutor.start();
        }
    ...
}
class NettyEventExecutor extends ServiceThread {
    public void run() {
        final ChannelEventListener listener = NettyRemotingAbstract.this.getChannelEventListener();
        ...
        NettyEvent event = this.eventQueue.poll(3000, TimeUnit.MILLISECONDS);
        if (event != null && listener != null) {
            switch (event.getType()) {
                case IDLE:
                    listener.onChannelIdle(event.getRemoteAddr(), event.getChannel());
                    break;
                case CLOSE:
                    listener.onChannelClose(event.getRemoteAddr(), event.getChannel());
                    break;
                case CONNECT:
                    listener.onChannelConnect(event.getRemoteAddr(), event.getChannel());
                    break;
                case EXCEPTION:
                    listener.onChannelException(event.getRemoteAddr(), event.getChannel());
                    break;
                default:
                    break;
            }
        }
    }
}
```

```java
//BrokerHouseKeepingService类中
    public void onChannelClose(String remoteAddr, Channel channel) {
        this.namesrvController.getRouteInfoManager().onChannelDestroy(remoteAddr, channel);
    }

    public void onChannelException(String remoteAddr, Channel channel) {
        this.namesrvController.getRouteInfoManager().onChannelDestroy(remoteAddr, channel);
    }

    public void onChannelIdle(String remoteAddr, Channel channel) {
        this.namesrvController.getRouteInfoManager().onChannelDestroy(remoteAddr, channel);
    }
```



### remoting

基于Netty的负责网络通信的模块

一个 Reactor 主线程负责监听 TCP 连接请求，建立好连接后丢给 Reactor 线程池，它负责将建立好连接的 socket 注册到 selector 上去（这里EventLoopGroup有两种选择，Nio和Epoll，可配置），然后监听真正的网络数据；拿到网络数据后，再丢给 Worker 线程池。 

Worker 拿到网络数据后，就交给 Pipeline，从 Head 到 Tail 一个个 Handler 的走下去（包括NettyEncoder、NettyDecoder、IdelStateHandler、NettyConnectManageHandler、NettyServerHandler等） 

**Producer、Consumer启动的是remotingClient，broker、nameSrv启动的是remotingServer**

**传输的消息为RemotingCommand**

RocketMQ的通信协议格式共分为四个部分:

- 消息总长度（四个字节存储，占用一个int类型）
- 序列化类型&消息头长度（占用一个int类型，第一个字节表示序列化类型，后三个字节表示消息头长度）
- 消息头数据
- 消息主体数据

**消息的编码：** 

```java
//Encoder负责把RemotingCommand转化为按照通信协议编码的ByteBuf
public class NettyEncoder extends MessageToByteEncoder<RemotingCommand> {
    @Override
    public void encode(ChannelHandlerContext ctx, RemotingCommand remotingCommand, ByteBuf out)
        throws Exception {
        ByteBuffer header = remotingCommand.encodeHeader();
        out.writeBytes(header);
        byte[] body = remotingCommand.getBody();
        if (body != null) {
            out.writeBytes(body);
        }
    }
}
```

```java
//Remoting类中，编码消息头部的方法    
	public ByteBuffer encodeHeader(final int bodyLength) {
        
        int length = 4;

        byte[] headerData;
        headerData = this.headerEncode();

        length += headerData.length;
        length += bodyLength;
        ByteBuffer result = ByteBuffer.allocate(4 + length - bodyLength);

        result.putInt(length);
        result.put(markProtocolType(headerData.length, serializeTypeCurrentRPC));
        result.put(headerData);
        result.flip();
        return result;
    }

    public static byte[] markProtocolType(int source, SerializeType type) {
        byte[] result = new byte[4];

        result[0] = type.getCode();
        result[1] = (byte) ((source >> 16) & 0xFF);
        result[2] = (byte) ((source >> 8) & 0xFF);
        result[3] = (byte) (source & 0xFF);
        return result;
    }
```

**消息的解码：** 

用到了Netty提供的LengthFieldBasedFrameDecoder，适用于携带长度字段的协议，可屏蔽粘包拆包问题

```java
public class NettyDecoder extends LengthFieldBasedFrameDecoder {

     /**
     * @param maxFrameLength  帧的最大长度
     * @param lengthFieldOffset length字段偏移的地址
     * @param lengthFieldLength length字段所占的字节长
     * @param lengthAdjustment 满足lengthAdjustment + length取值 = 除去length字段之后剩下的字节数
     * @param initialBytesToStrip 解析时候跳过多少个字节
     **/
    public NettyDecoder(int maxFrameLength, int lengthFieldOffset, int lengthFieldLength, int lengthAdjustment, int initialBytesToStrip) {
        super(maxFrameLength, lengthFieldOffset, lengthFieldLength, lengthAdjustment, initialBytesToStrip);         //调用父类构造方法
    }

    @Override
    public Object decode(ChannelHandlerContext ctx, ByteBuf in) throws Exception {
        ByteBuf frame = null;
        try {
            frame = (ByteBuf) super.decode(ctx, in);   //去除消息长度字段，把消息提取出来
            if (null == frame) {
                return null;
            }
            ByteBuffer byteBuffer = frame.nioBuffer();
            return RemotingCommand.decode(byteBuffer);
        } catch (Exception e) {
            ...
        } finally {
            if (null != frame) {
                frame.release();
            }
        }
        return null;
    }
}
```

```java
//Remoting类中，解码消息的方法    
	public static RemotingCommand decode(final ByteBuffer byteBuffer) {
        int length = byteBuffer.limit();   //消息长度（4 + headerLength + bodyLength）
        int oriHeaderLen = byteBuffer.getInt();    //取消息前四个字节
        int headerLength = getHeaderLength(oriHeaderLen);   //实质上是 oriHeaderLen & 0xFFFFFF

        byte[] headerData = new byte[headerLength];
        byteBuffer.get(headerData);

        RemotingCommand cmd = headerDecode(headerData, getProtocolType(oriHeaderLen));

        int bodyLength = length - 4 - headerLength;
        byte[] bodyData = null;
        if (bodyLength > 0) {
            bodyData = new byte[bodyLength];
            byteBuffer.get(bodyData);
        }
        cmd.body = bodyData;
        
        return cmd;
    }
```



### store

**消息存储特点：**

1、消息主题及元数据存储在**CommitLog**中；

2、**ConsumeQueue**存储消息在CommitLog中的位置信息（offset、size、tag），一个topic和一个queueId对应一个ConsumeQueue 

3、每次读取消息时，先读取consumeQueue，再通过consumeQueue去commitLog中拿到消息主体

4、这里consumeQueue借鉴了Kafka的patition（Kafka队列），每个topic可以有多个partition，每个partition都是独立的物理文件，消息直接从里面读写；但rocketmq对其进行了改良，consumeQueue只存储很少的数据，消息主体都是通过commitLog进行读写（优点：**单个队列轻量化**，减少磁盘竞争，不会出现因为访问队列增加而导致IOWait增加的情况；缺点：先读consumeQueue再读commitLog，且对于commitLog而言是顺序写但随机读，增加了时间开销）

**如何克服性能缺点？** 

1、MappedByteBuffer: CommitLog和ConsumeQueue都是磁盘文件，利用NIO的FileChannel模型将物理文件映射到缓冲区，提高读写速度

2、page cache: 将磁盘部分文件缓存到内存中，尽可能命中page cache，减少IO读操作

**消息写入流程：** 

1、设置消息存储时间戳和body CRC，向MappedFileQueue获取一个映射文件，返回Queue中最后一个

```java
DefaultMessageStore# putMessage(MessageExtBrokerInner msg)
---> CommitLog# putMessage(final MessageExtBrokerInner msg)   //下面的步骤基本都在这个方法内部完成
---> MappedFileQueue# getLastMappedFile()
```

2、CommitLog加锁，更新消息存储时间戳

```java
putMessageLock.lock();
long beginLockTimestamp = this.defaultMessageStore.getSystemClock().now();
msg.setStoreTimestamp(beginLockTimestamp);
```

3、如果获取映射文件为空或已写满，则通过AllocateMappedFileService创建映射文件

```java
if (null == mappedFile || mappedFile.isFull()) {
    mappedFile = this.mappedFileQueue.getLastMappedFile(0); //调用的是有参方法，可以创建MappedFile
}
---> MappedFileQueue# getLastMappedFile(final long startOffset, boolean needCreate)
---> AllocateMappedFileService# putRequestAndReturnMappedFile(String nextFilePath, String nextNextFilePath, int fileSize)
```

4、向映射文件中写入消息，写入MappedFile类中的Buffer，返回PUT_OK后，释放锁

```java
result = mappedFile.appendMessage(msg, this.appendMessageCallback);
switch (result.getStatus()) {
    case PUT_OK:
        break;
    ...
}
putMessageLock.unlock();
```

5、消息刷盘、落盘（前面只是将消息放到CommitLog引用的MappedFile中，还需要持久化到磁盘）

```java
handleDiskFlush(result, putMessageResult, msg);
```

**映射文件MappedFile**

MappedFile类是对MappedByteBuffer的封装，MappedByteBuffer是java NIO引入的高性能文件读写方案，将文件映射到虚拟内存（非堆区），如果文件较大可以分段进行映射。RocketMQ使用该类实现数据从内存到磁盘的持久化，对于commitlog、consumeQueue的读写都是通过MappedFileQueue实现的

**关键方法：**

- appendMessage: 插入消息到MappedFile，并返回插入结果.
- selectMappedBuffer: 返回指定位置的内存映射，用于读取数据.

```java
public AppendMessageResult appendMessagesInner(final MessageExt messageExt, final AppendMessageCallback cb) {
    //断言，表达式为true则继续往下执行，否则抛出AssertionError
    assert messageExt != null;
    assert cb != null;

    int currentPos = this.wrotePosition.get();   //获取写到哪一个位置

    if (currentPos < this.fileSize) {          //currentPos小于文件尺寸才能写入
        //获取需要写入的缓冲区，选择mappedByteBuffer还是writeBuffer与选择的落盘方式有关
        ByteBuffer byteBuffer = writeBuffer != null ? writeBuffer.slice() : this.mappedByteBuffer.slice();
        byteBuffer.position(currentPos);
        AppendMessageResult result = null;
        //调用AppendMessageCallback来执行msg到字节缓冲区的写入，doAppend()的具体实现与消息格式有关
        if (messageExt instanceof MessageExtBrokerInner) {
            result = cb.doAppend(this.getFileFromOffset(), byteBuffer, this.fileSize - currentPos, (MessageExtBrokerInner) messageExt);
        } else if (messageExt instanceof MessageExtBatch) {
            result = cb.doAppend(this.getFileFromOffset(), byteBuffer, this.fileSize - currentPos, (MessageExtBatch) messageExt);
        } else {
            return new AppendMessageResult(AppendMessageStatus.UNKNOWN_ERROR);
        }
        this.wrotePosition.addAndGet(result.getWroteBytes());    //更新已写到的位置
        this.storeTimestamp = result.getStoreTimestamp();
        return result;
    }
    return new AppendMessageResult(AppendMessageStatus.UNKNOWN_ERROR);
}
```

```java
//返回从pos到 pos + size的内存映射
public SelectMappedBufferResult selectMappedBuffer(int pos, int size) {
    int readPosition = getReadPosition();   //获取当前有效数据的最大位置
    if ((pos + size) <= readPosition) {    //内存映射的最大位置必须小于readPosition
        if (this.hold()) {    //引用计数
            ByteBuffer byteBuffer = this.mappedByteBuffer.slice();  
            byteBuffer.position(pos);    
            ByteBuffer byteBufferNew = byteBuffer.slice();  // 复制一个byteBuffer(与原byteBuffer共享数据, 只是指针位置独立)
            byteBufferNew.limit(size);
            //获取目标数据
            return new SelectMappedBufferResult(this.fileFromOffset + pos, byteBufferNew, size, this);
        }
    }
    return null;
}
```

**偏移量：** 每个commitLog文件的大小为1G，第一个commitLog起始偏移量为0，则第二个的起始偏移量为1G的字节数，根据全局偏移量可知道消息在哪一个commitLog且文件内的相对偏移量是多少



### Provider

**消息发送流程：**

获取主题路由信息（TopicRouteInfo）--> 队列负载 --> 发送到broker服务端 --> broker实现消息存储

**具体程序流：** 

**1）、**`producer.send(msg);` 

`defaultMQProducerImpl.send(msg);`

**2）、**`TopicPublishInfo topicPublishInfo = this.tryToFindTopicPublishInfo(msg.getTopic());` **获取主题路由信息**

2.1、`mQClientFactory.updateTopicRouteInfoFromNameServer(topic);`从NameSrv上获取topic对应的路由信息，如果没有找到topic，会使用默认的自动创建topic（*MixAll.AUTO_CREATE_TOPIC_KEY_TOPIC* ）

（broker.topic.TopicConfigManager中有自动创建topic的配置）

2.2、`topicRouteData = this.mQClientAPIImpl.getTopicRouteInfoFromNameServer(topic, 1000 * 3);` 

2.3、通过Netty发送给NameSrv，请求码为*GET_ROUTEINTO_BY_TOPIC*

```java
RemotingCommand request = RemotingCommand.createRequestCommand(RequestCode.GET_ROUTEINTO_BY_TOPIC, requestHeader);
RemotingCommand response = this.remotingClient.invokeSync(null, request, timeoutMillis);
//这里地址参数传入null，默认会和Namesrv建立通信
```

2.4 在NameSrv端（是一个RemotingServer），接收请求并处理

```java
switch (request.getCode()) {
    ...
    case RequestCode.GET_ROUTEINTO_BY_TOPIC:
        return this.getRouteInfoByTopic(ctx, request);
```

**3）、**`MessageQueue mqSelected = this.selectOneMessageQueue(topicPublishInfo, lastBrokerName);` **选择一个队列**

轮询法负载均衡（一个topic有多个队列，该topic的消息会均匀分布到各个队列）

```java
    int index = tpInfo.getSendWhichQueue().getAndIncrement();
    for (int i = 0; i < tpInfo.getMessageQueueList().size(); i++) {
        int pos = Math.abs(index++) % tpInfo.getMessageQueueList().size();
        if (pos < 0)
            pos = 0;
        MessageQueue mq = tpInfo.getMessageQueueList().get(pos);
        ...
    }
```

**4）、**`sendResult = this.sendKernelImpl(msg, mq, communicationMode, sendCallback, topicPublishInfo, timeout);` **执行消息的发送**

4.1、`sendResult = this.mQClientFactory.getMQClientAPIImpl().sendMessage(...)` 

4.2、`request = RemotingCommand.createRequestCommand(RequestCode.SEND_MESSAGE, requestHeader);` 

4.3、`RemotingCommand response = this.remotingClient.invokeSync(addr, request, timeoutMillis);` 通过Netty发送给broker，请求码为*SEND_MESSAGE*

**5）、在broker端（RemotingServer）处理消息** 

（SendProcessor类中的sendMessage()方法）

5.1 、`SendMessageRequestHeader requestHeader = parseRequestHeader(request);` 解析请求头

5.2、`response = this.sendMessage(ctx, request, mqtraceContext, requestHeader);`

5.3、`putMessageResult = this.brokerController.getMessageStore().putMessage(msgInner);` 存储消息



### broker

基本功能：消息堆积；生产与消费解耦（点对点、一对多）

高级功能：消息去重；顺序消息

**启动过程：** 

1、将初始化参数注入对应的Config类中，根据Config创建brokerController

```Java
final BrokerController controller = new BrokerController(
                brokerConfig,
                nettyServerConfig,
                nettyClientConfig,
                messageStoreConfig);
//broker自身配置：包括根目录, namesrv地址, broker的IP和名称, 消息队列数, 收发消息线程池数等参数
//netty启动配置：包括netty监听端口, 工作线程数, 网络配置等参数.
//存储层配置：包括存储根目录, CommitLog配置, 持久化策略配置等参数.
```

2、brokerController执行initialize()方法，加载多个组件，同时启动Netty服务端，注册多个消息处理器，初始化线程池用来做消息的并发处理 

```java
public boolean initialize() throws CloneNotSupportedException {
    //加载topicConfigManager组件，用于管理broker中存储的所有topic的配置
    boolean result = this.topicConfigManager.load();
	//加载consumerOffsetManager组件，用于管理Consumer的消费进度
    result = result && this.consumerOffsetManager.load();
    //管理订阅组
    result = result && this.subscriptionGroupManager.load();
    //管理消息过滤器
    result = result && this.consumerFilterManager.load();

    if (result) {
        try {
            //加载messageStore组件
            this.messageStore = new DefaultMessageStore(this.messageStoreConfig, this.brokerStatsManager, this.messageArrivingListener, this.brokerConfig);
            ...
        } catch (IOException e) {
            result = false;
            log.error("Failed to initialize", e);
        }
    }
    result = result && this.messageStore.load();
    
    if (result) {
        //加载Netty服务端，并初始化一些线程池、定时任务等
        this.remotingServer = new NettyRemotingServer(this.nettyServerConfig, this.clientHousekeepingService);
        ...
        //注册SendMessageProcessor、QueryMessageProcessor、ClientManageProcessor、ConsumerManageProcessor、EndTransactionProcessor等处理器，用于处理provider或consumer的请求
        this.registerProcessor();
        ...
    }
    return result;
}
```

3、brokerController执行start()方法，启动messageStore、remotingServer、clientHouseKeepingService、brokerStatsManager等组件

**过期文件清理**

broker在启动时，messageStore组件会加入一个定时任务

```java
this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
    @Override
    public void run() {
        DefaultMessageStore.this.cleanFilesPeriodically();    //定期清理文件
    }
}, 1000 * 60, this.messageStoreConfig.getCleanResourceInterval(), TimeUnit.MILLISECONDS);
```

```java
    private void cleanFilesPeriodically() {
        this.cleanCommitLogService.run();
        this.cleanConsumeQueueService.run();
    }
```

```java
public void run() {
    try {
        this.deleteExpiredFiles();    //尝试删除过期文件
        this.redeleteHangedFile();    //由于被其他线程引用在第一阶段未被删除的文件，在这里重试删除
    } catch (Throwable e) {
        ...
    }
}
```

**重复消息**

rocketMQ为了追求高性能，并不保证做到"Exactly Only Once"性能，也就是需要在业务层面实现消息去重，即消费消息时的幂等性



### PullConsumer

（example模块的simple包中）

consumer准备拉取broker中的消息时：

`pullBlockIfNotFound()` --> `DefaultMQPullConsumerImpl.pullSyncImpl()` --> `PullAPIWrapper.pullKernelImpl()`   --> `MQClientAPIImpl.pullMessage()` --> `RemotingClient.invokeSync(brokerAddr, request, timeOutMillis)` 

此时，consumer就将拉取消息的请求发送给了broker

broker在BrokerController.initialize()方法中会注册PullMessageProcessor来处理Pull Message请求

```java
this.remotingServer.registerProcessor(RequestCode.PULL_MESSAGE, this.pullMessageProcessor, this.pullMessageExecutor);
this.pullMessageProcessor.registerConsumeMessageHook(consumeMessageHookList);
```



### 分布式事务

RocketMQ 事务消息设计，主要是为了解决 Producer 端的消息发送与本地事务执行的原子性问题。RocketMQ 利用了broker 与 producer 端的双向通信能力，使得 broker 天生可以作为一个事务协调者存在；而 RocketMQ 本身提供的存储机制，则为事务消息提供了持久化能力；RocketMQ 的高可用机制以及可靠消息设计，则为事务消息在系统在发生异常时，依然能够保证事务的最终一致性达成。 

**主要流程：** 

1. 事务发起方（producer）首先发送 prepare 消息到 broker。
2. 在发送 prepare 消息成功后，producer执行本地事务。
3. 根据本地事务执行结果发送 commit 或者是 rollback 到broker。
4. broker处理消息，如果消息是 rollback，MQ 将删除该 prepare 消息不进行下发；如果是 commit 消息，MQ 将会把这个消息发送给 consumer 端。（这一步在broker.processor.EndTransactionProcessor类中实现，如果收到commit消息，实质上是执行`brokerController.getMessageStore().putMessage(msg)`）
5. 如果执行本地事务过程中，producer挂掉，或者超时（迟迟未收到commit/rollback），MQ 将会不停的询问其同组的其他 producer 来获取状态。
6. Consumer 端的消费成功机制由 MQ 保证。

**事务消息存储：** 

RocketMQ 通过使用 Half Topic 以及 Operation Topic 两个内部队列来存储事务消息推进状态 。Half Topic 对应队列中存放着 prepare 消息，Operation Topic 对应队列则存放了 prepare message 对应的 commit/rollback 消息，消息体中则是 prepare message 对应的 offset，broker通过比对两个队列的差值来找到尚未提交的超时事务，进行回查。 （broker.transaction.queue.TransactionalMessageBridge类）



