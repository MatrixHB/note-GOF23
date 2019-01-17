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

