# Netty 核心概念与架构

## 1. Netty 概述

```
Netty 是什么？

┌─────────────────────────────────────────────────────────────┐
│                      Netty                                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  定义: 异步事件驱动的网络应用框架                           │
│                                                             │
│  核心特点:                                                  │
│  ├── 高性能: NIO 非阻塞 I/O                                │
│  ├── 易用性: 简化网络编程复杂度                             │
│  ├── 可扩展: 灵活的组件和扩展点                             │
│  ├── 成熟稳定: 经过大规模生产验证                           │
│  └── 跨平台: 支持 Linux/Windows/macOS                      │
│                                                             │
│  应用场景:                                                  │
│  ├── RPC 框架: Dubbo、gRPC 底层通信                        │
│  ├── 消息中间件: RocketMQ、RabbitMQ                        │
│  ├── 即时通讯: 聊天服务器、推送服务                         │
│  ├── 游戏服务器: 高并发游戏逻辑                             │
│  └── Web 服务器: HTTP/HTTPS 服务                           │
│                                                             │
│  使用 Netty 的知名项目:                                    │
│  ├── Elasticsearch    ├── Cassandra                       │
│  ├── Spark            ├── Flink                           │
│  ├── Dubbo            ├── RocketMQ                        │
│  └── Spring WebFlux                                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 2. 线程模型

```
传统 BIO 模型:

┌─────────────────────────────────────────────────────────────┐
│  一个连接一个线程                                           │
│                                                             │
│  Thread-1 ←→ Client 1                                      │
│  Thread-2 ←→ Client 2                                      │
│  Thread-3 ←→ Client 3                                      │
│  ...                                                        │
│  Thread-N ←→ Client N                                      │
│                                                             │
│  问题:                                                      │
│  - 线程资源浪费（大部分时间在等待 I/O）                    │
│  - 线程切换开销大                                          │
│  - 无法支持高并发                                          │
└─────────────────────────────────────────────────────────────┘

Netty Reactor 模型:

┌─────────────────────────────────────────────────────────────┐
│  主从多线程模型                                             │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Boss Group (1-2个线程)                 │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐             │   │
│  │  │ EventLoop│  │ EventLoop│  │ EventLoop│             │   │
│  │  └─────────┘  └─────────┘  └─────────┘             │   │
│  │  职责: 接收连接，分发给 Worker                       │   │
│  └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│                           ▼                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Worker Group (N个线程)                  │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐             │   │
│  │  │ EventLoop│  │ EventLoop│  │ EventLoop│             │   │
│  │  └─────────┘  └─────────┘  └─────────┘             │   │
│  │  职责: 处理 I/O 读写、业务逻辑                       │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  EventLoop: 一个线程绑定一个 Selector，处理多个 Channel     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 3. 核心组件

```
Netty 核心组件:

┌─────────────────────────────────────────────────────────────┐
│                    Netty 组件层次                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Bootstrap/ServerBootstrap                                  │
│  ├── 启动引导类，配置线程组、Channel、Handler              │
│                                                             │
│  EventLoopGroup                                            │
│  ├── 线程组，管理多个 EventLoop                            │
│  ├── BossGroup: 接收连接                                   │
│  └── WorkerGroup: 处理 I/O                                 │
│                                                             │
│  EventLoop                                                 │
│  ├── 事件循环，一个线程绑定一个 Selector                   │
│  └── 处理 Channel 的 I/O 事件                              │
│                                                             │
│  Channel                                                    │
│  ├── 网络连接抽象，一个连接对应一个 Channel                │
│  └── 提供 I/O 操作（read/write/connect/bind）             │
│                                                             │
│  ChannelPipeline                                           │
│  ├── 处理器链，包含多个 ChannelHandler                     │
│  └── 责任链模式处理入站/出站事件                           │
│                                                             │
│  ChannelHandler                                            │
│  ├── 处理器，处理 I/O 事件                                 │
│  ├── ChannelInboundHandler: 入站处理器                     │
│  └── ChannelOutboundHandler: 出站处理器                    │
│                                                             │
│  ChannelFuture                                              │
│  ├── 异步操作结果                                          │
│  └── 支持监听器回调                                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 4. 第一个 Netty 程序

### 4.1 Maven 依赖

```xml
<dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-all</artifactId>
    <version>4.1.106.Final</version>
</dependency>
```

### 4.2 服务端

```java
public class NettyServer {
    
    private final int port;
    
    public NettyServer(int port) {
        this.port = port;
    }
    
    public void start() throws InterruptedException {
        // 1. 创建线程组
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        
        try {
            // 2. 创建启动引导类
            ServerBootstrap bootstrap = new ServerBootstrap();
            bootstrap.group(bossGroup, workerGroup)
                .channel(NioServerSocketChannel.class)
                .option(ChannelOption.SO_BACKLOG, 128)
                .childOption(ChannelOption.SO_KEEPALIVE, true)
                .childHandler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel ch) {
                        ChannelPipeline pipeline = ch.pipeline();
                        // 添加处理器
                        pipeline.addLast(new StringDecoder());
                        pipeline.addLast(new StringEncoder());
                        pipeline.addLast(new ServerHandler());
                    }
                });
            
            // 3. 绑定端口，启动服务器
            ChannelFuture future = bootstrap.bind(port).sync();
            System.out.println("Server started on port " + port);
            
            // 4. 等待服务器关闭
            future.channel().closeFuture().sync();
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
    
    public static void main(String[] args) throws InterruptedException {
        new NettyServer(8080).start();
    }
}

public class ServerHandler extends SimpleChannelInboundHandler<String> {
    
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, String msg) {
        System.out.println("Received: " + msg);
        ctx.writeAndFlush("Echo: " + msg);
    }
    
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        cause.printStackTrace();
        ctx.close();
    }
}
```

### 4.3 客户端

```java
public class NettyClient {
    
    private final String host;
    private final int port;
    
    public NettyClient(String host, int port) {
        this.host = host;
        this.port = port;
    }
    
    public void connect() throws InterruptedException {
        EventLoopGroup group = new NioEventLoopGroup();
        
        try {
            Bootstrap bootstrap = new Bootstrap();
            bootstrap.group(group)
                .channel(NioSocketChannel.class)
                .option(ChannelOption.SO_KEEPALIVE, true)
                .handler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel ch) {
                        ChannelPipeline pipeline = ch.pipeline();
                        pipeline.addLast(new StringDecoder());
                        pipeline.addLast(new StringEncoder());
                        pipeline.addLast(new ClientHandler());
                    }
                });
            
            ChannelFuture future = bootstrap.connect(host, port).sync();
            System.out.println("Connected to server");
            
            // 发送消息
            future.channel().writeAndFlush("Hello Netty!");
            
            future.channel().closeFuture().sync();
        } finally {
            group.shutdownGracefully();
        }
    }
}

public class ClientHandler extends SimpleChannelInboundHandler<String> {
    
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, String msg) {
        System.out.println("Server response: " + msg);
    }
}
```

## 5. 编解码器

```
Netty 编解码器:

┌─────────────────────────────────────────────────────────────┐
│                    编解码器类型                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  内置编解码器:                                              │
│  ├── StringDecoder/Encoder: 字符串编解码                    │
│  ├── ObjectDecoder/Encoder: Java 对象编解码                 │
│  ├── ByteToMessageDecoder: 字节转消息                       │
│  └── MessageToByteEncoder: 消息转字节                       │
│                                                             │
│  自定义协议编解码器:                                        │
│  ├── LengthFieldBasedFrameDecoder: 基于长度字段             │
│  ├── DelimiterBasedFrameDecoder: 基于分隔符                │
│  └── FixedLengthFrameDecoder: 固定长度                     │
│                                                             │
│  序列化框架:                                                │  ├── Protobuf: Google 高性能序列化
│  ├── Kryo: Java 高性能序列化                               │
│  ├── Hessian: 跨语言序列化                                 │
│  └── JSON: 可读性好                                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

```java
// 自定义协议编解码器
public class MessageDecoder extends ByteToMessageDecoder {
    
    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
        // 检查是否有足够的字节
        if (in.readableBytes() < 8) {
            return;
        }
        
        // 标记当前读位置
        in.markReaderIndex();
        
        // 读取长度
        int length = in.readInt();
        
        // 检查是否有足够的数据
        if (in.readableBytes() < length) {
            in.resetReaderIndex();
            return;
        }
        
        // 读取数据
        byte[] data = new byte[length];
        in.readBytes(data);
        
        // 反序列化
        String message = new String(data, StandardCharsets.UTF_8);
        out.add(message);
    }
}

public class MessageEncoder extends MessageToByteEncoder<String> {
    
    @Override
    protected void encode(ChannelHandlerContext ctx, String msg, ByteBuf out) {
        byte[] data = msg.getBytes(StandardCharsets.UTF_8);
        out.writeInt(data.length);
        out.writeBytes(data);
    }
}
```

## 6. 心跳机制

```java
// 服务端心跳检测
public class HeartbeatHandler extends ChannelInboundHandlerAdapter {
    
    private static final int READER_IDLE_TIME = 60;  // 60秒没有读事件
    private static final int WRITER_IDLE_TIME = 30;  // 30秒没有写事件
    private static final int ALL_IDLE_TIME = 90;     // 90秒没有读写事件
    
    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) {
        if (evt instanceof IdleStateEvent) {
            IdleStateEvent event = (IdleStateEvent) evt;
            
            if (event.state() == IdleState.READER_IDLE) {
                System.out.println("读空闲，关闭连接");
                ctx.close();
            } else if (event.state() == IdleState.WRITER_IDLE) {
                System.out.println("写空闲，发送心跳");
                ctx.writeAndFlush("heartbeat");
            } else if (event.state() == IdleState.ALL_IDLE) {
                System.out.println("读写空闲");
            }
        }
    }
}

// 配置心跳
bootstrap.childHandler(new ChannelInitializer<SocketChannel>() {
    @Override
    protected void initChannel(SocketChannel ch) {
        ch.pipeline().addLast(new IdleStateHandler(
            READER_IDLE_TIME, WRITER_IDLE_TIME, ALL_IDLE_TIME,
            TimeUnit.SECONDS));
        ch.pipeline().addLast(new HeartbeatHandler());
    }
});
```

## 7. 粘包拆包处理

```
粘包拆包问题:

┌─────────────────────────────────────────────────────────────┐
│  粘包: 多个数据包粘在一起                                   │
│  发送: [包1][包2][包3]                                      │
│  接收: [包1包2包3]                                          │
│                                                             │
│  拆包: 一个数据包被拆分                                     │
│  发送: [大包]                                               │
│  接收: [包的一部分][包的另一部分]                           │
│                                                             │
│  解决方案:                                                  │
│  ├── 固定长度: FixedLengthFrameDecoder                     │
│  ├── 分隔符: DelimiterBasedFrameDecoder                    │
│  ├── 长度字段: LengthFieldBasedFrameDecoder                 │
│  └── 自定义协议                                           │
└─────────────────────────────────────────────────────────────┘
```

```java
// 方案1: 固定长度
pipeline.addLast(new FixedLengthFrameDecoder(100));

// 方案2: 分隔符
ByteBuf delimiter = Unpooled.copiedBuffer("\n".getBytes());
pipeline.addLast(new DelimiterBasedFrameDecoder(1024, delimiter));

// 方案3: 长度字段（推荐）
// 协议: [长度(4字节)][数据(N字节)]
pipeline.addLast(new LengthFieldBasedFrameDecoder(
    1024,    // 最大帧长度
    0,       // 长度字段偏移
    4,       // 长度字段大小
    0,       // 长度调整
    4        // 跳过的字节数
));
```

## 8. 最佳实践

```
Netty 使用 Checklist:

□ 线程组配置:
  - BossGroup 线程数 1-2 个即可
  - WorkerGroup 线程数默认 CPU 核心数 * 2
  - 业务逻辑复杂时使用单独的业务线程池

□ Channel 配置:
  - SO_BACKLOG: TCP 连接队列大小
  - SO_KEEPALIVE: TCP 保持连接
  - TCP_NODELAY: 禁用 Nagle 算法

□ 内存管理:
  - 使用 ByteBuf 池化分配
  - 及时释放 ByteBuf（release）
  - 避免内存泄漏（ResourceLeakDetector）

□ 编解码:
  - 使用自定义协议处理粘包拆包
  - 选择高效的序列化方式
  - 考虑数据压缩（大数据量）

□ 异常处理:
  - 实现 exceptionCaught 方法
  - 记录异常日志
  - 优雅关闭连接

□ 性能优化:
  - 使用 CompositeByteBuf 减少内存拷贝
  - 使用 FileRegion 零拷贝传输文件
  - 合理设置水位线（WriteBufferWaterMark）
```
