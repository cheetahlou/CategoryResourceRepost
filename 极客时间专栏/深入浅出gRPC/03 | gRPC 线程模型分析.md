
# RPC线程模型

## 1.1 BIO线程模型

在JDK 1.4推出Java NIO之前，基于Java的所有Socket通信都采用了同步阻塞模式（BIO），这种一请求一应答的通信模型简化了上层的应用开发，但是在性能和可靠性方面却存在着巨大的瓶颈。

因此，在很长一段时间里，大型的应用服务器都采用C或者C++语言开发，因为它们可以直接使用操作系统提供的异步I/O或者AIO能力。

当并发访问量增大、响应时间延迟增大之后，采用Java BIO开发的服务端软件只有通过硬件的不断扩容来满足高并发和低时延。

它极大地增加了企业的成本，并且随着集群规模的不断膨胀，系统的可维护性也面临巨大的挑战，只能通过采购性能更高的硬件服务器来解决问题，这会导致恶性循环。

传统采用BIO的Java Web服务器如下所示（典型的如Tomcat的BIO模式）：

<img src="https://static001.geekbang.org/resource/image/62/42/62e08cf3d345b1ccb91429f28d454542.png" alt="" />

采用该线程模型的服务器调度特点如下：

- 服务端监听线程Acceptor负责客户端连接的接入，每当有新的客户端接入，就会创建一个新的I/O线程负责处理Socket；
- 客户端请求消息的读取和应答的发送，都有I/O线程负责；
- 除了I/O读写操作，默认情况下业务的逻辑处理，例如DB操作等，也都在I/O线程处理；
- I/O操作采用同步阻塞操作，读写没有完成，I/O线程会同步阻塞。

BIO线程模型主要存在如下三个问题：

1. **性能问题：**一连接一线程模型导致服务端的并发接入数和系统吞吐量受到极大限制；
1. **可靠性问题：**由于I/O操作采用同步阻塞模式，当网络拥塞或者通信对端处理缓慢会导致I/O线程被挂住，阻塞时间无法预测；
1. **可维护性问题：**I/O线程数无法有效控制、资源无法有效共享（多线程并发问题），系统可维护性差。

为了解决同步阻塞I/O面临的一个链路需要一个线程处理的问题，通常会对它的线程模型进行优化，后端通过一个线程池来处理多个客户端的请求接入，形成客户端个数&quot;M&quot;与线程池最大线程数&quot;N&quot;的比例关系，其中M可以远远大于N，通过线程池可以灵活的调配线程资源，设置线程的最大值，防止由于海量并发接入导致线程耗尽，它的工作原理如下所示：

<img src="https://static001.geekbang.org/resource/image/6d/ca/6da1ab2c9d34a3660374a8a997ac03ca.png" alt="" />

优化之后的BIO模型采用了线程池实现，因此避免了为每个请求都创建一个独立线程造成的线程资源耗尽问题。但是由于它底层的通信依然采用同步阻塞模型，阻塞的时间取决于对方I/O线程的处理速度和网络I/O的传输速度。

本质上来讲，无法保证生产环境的网络状况和对端的应用程序能足够快，如果应用程序依赖对方的处理速度，它的可靠性就非常差，优化之后的BIO线程模型仍然无法从根本上解决性能线性扩展问题。

## 1.2 异步非阻塞线程模型

从JDK1.0到JDK1.3，Java的I/O类库都非常原始，很多UNIX网络编程中的概念或者接口在I/O类库中都没有体现，例如Pipe、Channel、Buffer和Selector等。2002年发布JDK1.4时，NIO以JSR-51的身份正式随JDK发布。它新增了个java.nio包，提供了很多进行异步I/O开发的API和类库，主要的类和接口如下：

- 进行异步I/O操作的缓冲区ByteBuffer等；
- 进行异步I/O操作的管道Pipe；
- 进行各种I/O操作（异步或者同步）的Channel，包括ServerSocketChannel和SocketChannel；
- 多种字符集的编码能力和解码能力；
- 实现非阻塞I/O操作的多路复用器selector；
- 基于流行的Perl实现的正则表达式类库；
- 文件通道FileChannel。

新的NIO类库的提供，极大地促进了基于Java的异步非阻塞编程的发展和应用,也诞生了很多优秀的Java NIO框架，例如Apache的Mina、以及当前非常流行的Netty。

Java NIO类库的工作原理如下所示：

<img src="https://static001.geekbang.org/resource/image/53/38/53437505107a710a603c6a8593f00638.png" alt="" />

在Java NIO类库中，最重要的就是多路复用器Selector，它是Java NIO编程的基础，熟练地掌握Selector对于掌握NIO编程至关重要。多路复用器提供选择已经就绪的任务的能力。

简单来讲，Selector会不断地轮询注册在其上的Channel，如果某个Channel上面有新的TCP连接接入、读和写事件，这个Channel就处于就绪状态，会被Selector轮询出来，然后通过SelectionKey可以获取就绪Channel的集合，进行后续的I/O操作。

通常一个I/O线程会聚合一个Selector，一个Selector可以同时注册N个Channel,这样单个I/O线程就可以同时并发处理多个客户端连接。另外，由于I/O操作是非阻塞的，因此也不会受限于网络速度和对方端点的处理时延，可靠性和效率都得到了很大提升。

典型的NIO线程模型（Reactor模式）如下所示：

<img src="https://static001.geekbang.org/resource/image/b5/5f/b5233ba74ab40a0ad9f4c7822164795f.png" alt="" />

## 1.3 RPC性能三原则

影响RPC框架性能的三个核心要素如下：

1. **I/O模型：**用什么样的通道将数据发送给对方，BIO、NIO或者AIO，IO模型在很大程度上决定了框架的性能；
1. **协议：**采用什么样的通信协议，Rest+ JSON或者基于TCP的私有二进制协议，协议的选择不同，性能模型也不同，相比于公有协议，内部私有二进制协议的性能通常可以被设计的更优；
1. **线程：**数据报如何读取？读取之后的编解码在哪个线程进行，编解码后的消息如何派发，通信线程模型的不同，对性能的影响也非常大。

在以上三个要素中，线程模型对性能的影响非常大。随着硬件性能的提升，CPU的核数越来越越多，很多服务器标配已经达到32或64核。

通过多线程并发编程，可以充分利用多核CPU的处理能力，提升系统的处理效率和并发性能。但是如果线程创建或者管理不当，频繁发生线程上下文切换或者锁竞争，反而会影响系统的性能。

线程模型的优劣直接影响了RPC框架的性能和并发能力，它也是大家选型时比较关心的技术细节之一。下面我们一起来分析和学习下gRPC的线程模型。

# 2. gRPC线程模型分析

gRPC的线程模型主要包括服务端线程模型和客户端线程模型，其中服务端线程模型主要包括：

- 服务端监听和客户端接入线程（HTTP /2 Acceptor）
- 网络I/O读写线程
- 服务接口调用线程

客户端线程模型主要包括：

- 客户端连接线程（HTTP/2 Connector）
- 网络I/O读写线程
- 接口调用线程
- 响应回调通知线程

## 2.1 服务端线程模型

gRPC服务端线程模型整体上可以分为两大类：

- 网络通信相关的线程模型，基于Netty4.1的线程模型实现
- 服务接口调用线程模型，基于JDK线程池实现

### 2.1.1 服务端线程模型概述

gRPC服务端线程模型和交互图如下所示：

<img src="https://static001.geekbang.org/resource/image/7b/b3/7b75fb1c58e0bee27cddc8b3f3e843b3.png" alt="" />

其中，HTTP/2服务端创建、HTTP/2请求消息的接入和响应发送都由Netty负责，gRPC消息的序列化和反序列化、以及应用服务接口的调用由gRPC的SerializingExecutor线程池负责。

### 2.1.2 I/O通信线程模型

gRPC的做法是服务端监听线程和I/O线程分离的Reactor多线程模型，它的代码如下所示（NettyServer类）：

```
public void start(ServerListener serverListener) throws IOException {
    listener = checkNotNull(serverListener, &quot;serverListener&quot;);
    allocateSharedGroups();
    ServerBootstrap b = new ServerBootstrap();
    b.group(bossGroup, workerGroup);
    b.channel(channelType);
    if (NioServerSocketChannel.class.isAssignableFrom(channelType)) {
      b.option(SO_BACKLOG, 128);
      b.childOption(SO_KEEPALIVE, true);

```

它的工作原理如下：

<img src="https://static001.geekbang.org/resource/image/4a/1e/4a59ff78a4b550df92138c593e71771e.png" alt="" />

流程如下：

**步骤1：**业务线程发起创建服务端操作，在创建服务端的时候实例化了2个EventLoopGroup，1个EventLoopGroup实际就是一个EventLoop线程组，负责管理EventLoop的申请和释放。

EventLoopGroup管理的线程数可以通过构造函数设置，如果没有设置，默认取-Dio.netty.eventLoopThreads，如果该系统参数也没有指定，则为“可用的CPU内核 * 2”。

bossGroup线程组实际就是Acceptor线程池，负责处理客户端的TCP连接请求，如果系统只有一个服务端端口需要监听，则建议bossGroup线程组线程数设置为1。workerGroup是真正负责I/O读写操作的线程组，通过ServerBootstrap的group方法进行设置，用于后续的Channel绑定。

**步骤2：**服务端Selector轮询，监听客户端连接，代码示例如下（NioEventLoop类）：

```
int selectedKeys = selector.select(timeoutMillis);
 selectCnt ++;

```

**步骤3：**如果监听到客户端连接，则创建客户端SocketChannel连接，从workerGroup中随机选择一个NioEventLoop线程，将SocketChannel注册到该线程持有的Selector，代码示例如下（NioServerSocketChannel类）：

```
protected int doReadMessages(List&lt;Object&gt; buf) throws Exception {
        SocketChannel ch = SocketUtils.accept(javaChannel());
        try {
            if (ch != null) {
                buf.add(new NioSocketChannel(this, ch));
                return 1;
            }

```

**步骤4：**通过调用EventLoopGroup的next()获取一个EventLoop（NioEventLoop），用于处理网络I/O事件。

Netty线程模型的核心是NioEventLoop，它的职责如下：

1. 作为服务端Acceptor线程，负责处理客户端的请求接入
1. 作为I/O线程，监听网络读操作位，负责从SocketChannel中读取报文
1. 作为I/O线程，负责向SocketChannel写入报文发送给对方，如果发生写半包，会自动注册监听写事件，用于后续继续发送半包数据，直到数据全部发送完成
1. 作为定时任务线程，可以执行定时任务，例如链路空闲检测和发送心跳消息等
1. 作为线程执行器可以执行普通的任务线程（Runnable）NioEventLoop处理网络I/O操作的相关代码如下：

```
try {
            int readyOps = k.readyOps();
            if ((readyOps &amp; SelectionKey.OP_CONNECT) != 0) {
                int ops = k.interestOps();
                ops &amp;= ~SelectionKey.OP_CONNECT;
                k.interestOps(ops);

                unsafe.finishConnect();
            }
            if ((readyOps &amp; SelectionKey.OP_WRITE) != 0) {
                              ch.unsafe().forceFlush();
            }
            if ((readyOps &amp; (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
                unsafe.read();
            }

```

除了处理I/O操作，NioEventLoop也可以执行Runnable和定时任务。NioEventLoop继承SingleThreadEventExecutor，这就意味着它实际上是一个线程个数为1的线程池，类继承关系如下所示：

<img src="https://static001.geekbang.org/resource/image/f3/14/f3f92ac29c50a6e31743919450057914.png" alt="" />

SingleThreadEventExecutor聚合了JDK的java.util.concurrent.Executor和消息队列Queue，自定义提供线程池功能，相关代码如下（SingleThreadEventExecutor类）：

```
private final Queue&lt;Runnable&gt; taskQueue;
    private volatile Thread thread;
    @SuppressWarnings(&quot;unused&quot;)
    private volatile ThreadProperties threadProperties;
    private final Executor executor;
    private volatile boolean interrupted;


```

直接调用NioEventLoop的execute(Runnable task)方法即可执行自定义的Task，代码示例如下（SingleThreadEventExecutor类）：

```
public void execute(Runnable task) {
        if (task == null) {
            throw new NullPointerException(&quot;task&quot;);
        }
        boolean inEventLoop = inEventLoop();
        if (inEventLoop) {
            addTask(task);
        } else {
            startThread();
            addTask(task);
            if (isShutdown() &amp;&amp; removeTask(task)) {
                reject();
            }

```

除了SingleThreadEventExecutor，NioEventLoop同时实现了ScheduledExecutorService接口，这意味着它也可以执行定时任务，相关接口定义如下：

<img src="https://static001.geekbang.org/resource/image/07/17/0725a5e639ba33e0bd153fac3ed34a17.png" alt="" />

为了防止大量Runnable和定时任务执行影响网络I/O的处理效率，Netty提供了一个配置项：ioRatio，用于设置I/O操作和其它任务执行的时间比例，默认为50%，相关代码示例如下（NioEventLoop类）：

```
final long ioTime = System.nanoTime() - ioStartTime;
runAllTasks(ioTime * (100 - ioRatio) / ioRatio);

```

NioEventLoop同时支持I/O操作和Runnable执行的原因如下：避免锁竞争，例如心跳检测，往往需要周期性的执行，如果NioEventLoop不支持定时任务执行，则用户需要自己创建一个类似ScheduledExecutorService的定时任务线程池或者定时任务线程，周期性的发送心跳，发送心跳需要网络操作，就要跟I/O线程所持有的资源进行交互，例如Handler、ByteBuf、NioSocketChannel等，这样就会产生锁竞争，需要考虑并发安全问题。原理如下：

<img src="https://static001.geekbang.org/resource/image/eb/13/eb0b9ec4947a7a499f3a8f783c755413.png" alt="" />

### 2.1.3. 服务调度线程模型

gRPC服务调度线程主要职责如下：

- 请求消息的反序列化，主要包括：HTTP/2 Header的反序列化，以及将PB(Body)反序列化为请求对象；
- 服务接口的调用，method.invoke(非反射机制)；
<li>将响应消息封装成WriteQueue.QueuedCommand，写入到Netty Channel中，同时，对响应Header和Body对象做序列化<br />
服务端调度的核心是SerializingExecutor，它同时实现了JDK的Executor和Runnable接口，既是一个线程池，同时也是一个Task。</li>

SerializingExecutor聚合了JDK的Executor，由Executor负责Runnable的执行，代码示例如下（SerializingExecutor类）：

```
public final class SerializingExecutor implements Executor, Runnable {
  private static final Logger log =
      Logger.getLogger(SerializingExecutor.class.getName());
private final Executor executor;
  private final Queue&lt;Runnable&gt; runQueue = new ConcurrentLinkedQueue&lt;Runnable&gt;();

```

其中，Executor默认使用的是JDK的CachedThreadPool，在构建ServerImpl的时候进行初始化，代码如下：

<img src="https://static001.geekbang.org/resource/image/ab/62/ab64e112210d75e7b9201a06f9dec662.png" alt="" />

当服务端接收到客户端HTTP/2请求消息时，由Netty的NioEventLoop线程切换到gRPC的SerializingExecutor，进行消息的反序列化、以及服务接口的调用，代码示例如下（ServerTransportListenerImpl类）：

```
final Context.CancellableContext context = createContext(stream, headers, statsTraceCtx);
      final Executor wrappedExecutor;
      if (executor == directExecutor()) {
        wrappedExecutor = new SerializeReentrantCallsDirectExecutor();
      } else {
        wrappedExecutor = new SerializingExecutor(executor);
      }
      final JumpToApplicationThreadServerStreamListener jumpListener
          = new JumpToApplicationThreadServerStreamListener(wrappedExecutor, stream, context);
      stream.setListener(jumpListener);
      wrappedExecutor.execute(new ContextRunnable(context) {
          @Override
          public void runInContext() {
            ServerStreamListener listener = NOOP_LISTENER;
            try {
              ServerMethodDefinition&lt;?, ?&gt; method = registry.lookupMethod(methodName);
...

```

相关的调用堆栈，示例如下：

<img src="https://static001.geekbang.org/resource/image/3d/d3/3d3f45e12b87885967901f823e1eddd3.png" alt="" />

响应消息的发送，由SerializingExecutor发起，将响应消息头和消息体序列化，然后分别封装成SendResponseHeadersCommand和SendGrpcFrameCommand，调用Netty NioSocketChannle的write方法，发送到Netty的ChannelPipeline中，由gRPC的NettyServerHandler拦截之后，真正写入到SocketChannel中，代码如下所示（NettyServerHandler类）：

```
public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise)
      throws Exception {
    if (msg instanceof SendGrpcFrameCommand) {
      sendGrpcFrame(ctx, (SendGrpcFrameCommand) msg, promise);
    } else if (msg instanceof SendResponseHeadersCommand) {
      sendResponseHeaders(ctx, (SendResponseHeadersCommand) msg, promise);
    } else if (msg instanceof CancelServerStreamCommand) {
      cancelStream(ctx, (CancelServerStreamCommand) msg, promise);
    } else if (msg instanceof ForcefulCloseCommand) {
      forcefulClose(ctx, (ForcefulCloseCommand) msg, promise);
    } else {
      AssertionError e =
          new AssertionError(&quot;Write called for unexpected type: &quot; + msg.getClass().getName());
      ReferenceCountUtil.release(msg);

```

响应消息体的发送堆栈如下所示：

<img src="https://static001.geekbang.org/resource/image/33/2e/336739fac897e18e932aa98ff874982e.png" alt="" />

Netty I/O线程和服务调度线程的运行分工界面以及切换点如下所示：

<img src="https://static001.geekbang.org/resource/image/b4/e9/b4ba273d77641bc440466ad7d37d70e9.png" alt="" />

事实上，在实际服务接口调用过程中，NIO线程和服务调用线程切换次数远远超过4次，频繁的线程切换对gRPC的性能带来了一定的损耗。

## 2.2 客户端线程模型

gRPC客户端的线程主要分为三类：

1. 业务调用线程
1. 客户端连接和I/O读写线程
1. 请求消息业务处理和响应回调线程

### 2.2.1 客户端线程模型概述

gRPC客户端线程模型工作原理如下图所示（同步阻塞调用为例）：<br />
<img src="https://static001.geekbang.org/resource/image/77/90/77ffa44324ea0e318caa3693021ae490.png" alt="" />

客户端调用主要涉及的线程包括：

- 应用线程，负责调用gRPC服务端并获取响应，其中请求消息的序列化由该线程负责；
- 客户端负载均衡以及Netty Client创建，由grpc-default-executor线程池负责；
- HTTP/2客户端链路创建、网络I/O数据的读写，由Netty NioEventLoop线程负责；
- 响应消息的反序列化由SerializingExecutor负责，与服务端不同的是，客户端使用的是ThreadlessExecutor，并非JDK线程池；
- SerializingExecutor通过调用responseFuture的set(value)，唤醒阻塞的应用线程，完成一次RPC调用。

### 2.2.2 I/O通信线程模型

相比于服务端，客户端的线程模型简单一些，它的工作原理如下：<br />
<img src="https://static001.geekbang.org/resource/image/f0/fa/f0b5c823ca2ee2a7ccc285ca8d081ffa.png" alt="" />

第1步，由grpc-default-executor发起客户端连接，示例代码如下（NettyClientTransport类）：

```
Bootstrap b = new Bootstrap();
    b.group(eventLoop);
    b.channel(channelType);
    if (NioSocketChannel.class.isAssignableFrom(channelType)) {
      b.option(SO_KEEPALIVE, true);
    }
    for (Map.Entry&lt;ChannelOption&lt;?&gt;, ?&gt; entry : channelOptions.entrySet()) {
      b.option((ChannelOption&lt;Object&gt;) entry.getKey(), entry.getValue());
    }


```

相比于服务端，客户端只需要创建一个NioEventLoop，因为它不需要独立的线程去监听客户端连接，也没必要通过一个单独的客户端线程去连接服务端。

Netty是异步事件驱动的NIO框架，它的连接和所有I/O操作都是非阻塞的，因此不需要创建单独的连接线程。

另外，客户端使用的work线程组并非通常意义的EventLoopGroup，而是一个EventLoop：即HTTP/2客户端使用的work线程并非一组线程（默认线程数为CPU内核 * 2），而是一个EventLoop线程。

这个其实也很容易理解，一个NioEventLoop线程可以同时处理多个HTTP/2客户端连接，它是多路复用的，对于单个HTTP/2客户端，如果默认独占一个work线程组，将造成极大的资源浪费，同时也可能会导致句柄溢出（并发启动大量HTTP/2客户端）。

第2步，发起连接操作，判断连接结果，判断连接结果，如果没有连接成功，则监听连接网络操作位SelectionKey.OP_CONNECT。如果连接成功，则调用pipeline().fireChannelActive()将监听位修改为READ。代码如下（NioSocketChannel类）：

```
protected boolean doConnect(SocketAddress remoteAddress, SocketAddress localAddress) throws Exception {
        if (localAddress != null) {
            doBind0(localAddress);
        }
        boolean success = false;
        try {
            boolean connected = SocketUtils.connect(javaChannel(), remoteAddress);
            if (!connected) {
                selectionKey().interestOps(SelectionKey.OP_CONNECT);
            }
            success = true;
            return connected;

```

第3步，由NioEventLoop的多路复用器轮询连接操作结果，判断连接结果，如果或连接成功，重新设置监听位为READ（AbstractNioChannel类）：

```
protected void doBeginRead() throws Exception {
        final SelectionKey selectionKey = this.selectionKey;
        if (!selectionKey.isValid()) {
            return;
        }
        readPending = true;
        final int interestOps = selectionKey.interestOps();
        if ((interestOps &amp; readInterestOp) == 0) {
            selectionKey.interestOps(interestOps | readInterestOp);
        }

```

第4步，由NioEventLoop线程负责I/O读写，同服务端。

### 2.2.3 客户端调用线程模型

客户端调用线程交互流程如下所示：

<img src="https://static001.geekbang.org/resource/image/2b/d1/2b4ae73b676745847f2799720fc90bd1.png" alt="" />

请求消息的发送由用户线程发起，相关代码示例如下（GreeterBlockingStub类）：

```
public io.grpc.examples.helloworld.HelloReply sayHello(io.grpc.examples.helloworld.HelloRequest request) {
      return blockingUnaryCall(
          getChannel(), METHOD_SAY_HELLO, getCallOptions(), request);
    }

```

HTTP/2 Header的创建、以及请求参数反序列化为Protobuf，均由用户线程负责完成，相关代码示例如下（ClientCallImpl类）：

```
public void sendMessage(ReqT message) {
    Preconditions.checkState(stream != null, &quot;Not started&quot;);
    Preconditions.checkState(!cancelCalled, &quot;call was cancelled&quot;);
    Preconditions.checkState(!halfCloseCalled, &quot;call was half-closed&quot;);
    try {
      InputStream messageIs = method.streamRequest(message);
      stream.writeMessage(messageIs);
...

```

用户线程将请求消息封装成CreateStreamCommand和SendGrpcFrameCommand，发送到Netty的ChannelPipeline中，然后返回，完成线程切换。后续操作由Netty NIO线程负责，相关代码示例如下（NettyClientHandler类）：

```
public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise)
          throws Exception {
    if (msg instanceof CreateStreamCommand) {
      createStream((CreateStreamCommand) msg, promise);
    } else if (msg instanceof SendGrpcFrameCommand) {
      sendGrpcFrame(ctx, (SendGrpcFrameCommand) msg, promise);
    } else if (msg instanceof CancelClientStreamCommand) {
      cancelStream(ctx, (CancelClientStreamCommand) msg, promise);
    } else if (msg instanceof SendPingCommand) {
      sendPingFrame(ctx, (SendPingCommand) msg, promise);
    } else if (msg instanceof GracefulCloseCommand) {
      gracefulClose(ctx, (GracefulCloseCommand) msg, promise);
    } else if (msg instanceof ForcefulCloseCommand) {
      forcefulClose(ctx, (ForcefulCloseCommand) msg, promise);
    } else if (msg == NOOP_MESSAGE) {
      ctx.write(Unpooled.EMPTY_BUFFER, promise);
...

```

客户端响应消息的接收，由gRPC的NettyClientHandler负责，相关代码如下所示：

<img src="https://static001.geekbang.org/resource/image/69/79/6941dbae4f7b8470f3c5fd653dc33179.png" alt="" />

接收到HTTP/2响应之后，Netty将消息投递到SerializingExecutor，由SerializingExecutor的ThreadlessExecutor负责响应的反序列化，以及responseFuture的设值，相关代码示例如下（UnaryStreamToFuture类）：

```
public void onClose(Status status, Metadata trailers) {
      if (status.isOk()) {
        if (value == null) {
          // No value received so mark the future as an error
          responseFuture.setException(
              Status.INTERNAL.withDescription(&quot;No value received for unary call&quot;)
                  .asRuntimeException(trailers));
        }
        responseFuture.set(value);
      } else {
        responseFuture.setException(status.asRuntimeException(trailers));
      }

```

# 线程模型总结

## 3.1 优点

### 3.1.1 Netty线程模型

Netty4之后，对线程模型进行了优化，通过串行化的设计避免线程竞争：当系统在运行过程中，如果频繁的进行线程上下文切换，会带来额外的性能损耗。

多线程并发执行某个业务流程，业务开发者还需要时刻对线程安全保持警惕，哪些数据可能会被并发修改，如何保护？这不仅降低了开发效率，也会带来额外的性能损耗。

为了解决上述问题，Netty采用了串行化设计理念，从消息的读取、编码以及后续Handler的执行，始终都由I/O线程NioEventLoop负责，这就意外着整个流程不会进行线程上下文的切换，数据也不会面临被并发修改的风险，对于用户而言，甚至不需要了解Netty的线程细节，这确实是个非常好的设计理念，它的工作原理图如下：

<img src="https://static001.geekbang.org/resource/image/71/4a/71399ff626d246d67a4fc21b2835fe4a.png" alt="" />

一个NioEventLoop聚合了一个多路复用器Selector，因此可以处理成百上千的客户端连接，Netty的处理策略是每当有一个新的客户端接入，则从NioEventLoop线程组中顺序获取一个可用的NioEventLoop，当到达数组上限之后，重新返回到0，通过这种方式，可以基本保证各个NioEventLoop的负载均衡。一个客户端连接只注册到一个NioEventLoop上，这样就避免了多个I/O线程去并发操作它。

Netty通过串行化设计理念降低了用户的开发难度，提升了处理性能。利用线程组实现了多个串行化线程水平并行执行，线程之间并没有交集，这样既可以充分利用多核提升并行处理能力，同时避免了线程上下文的切换和并发保护带来的额外性能损耗。

Netty 3的I/O事件处理流程如下：

<img src="https://static001.geekbang.org/resource/image/52/3e/52a661e44e12274c41484e98f20c153e.png" alt="" />

Netty 4的I/O消息处理流程如下所示：

<img src="https://static001.geekbang.org/resource/image/be/86/be77fe6bbcfaecd4ca96d16b05aa3786.png" alt="" />

Netty 4修改了Netty 3的线程模型：在Netty 3的时候，upstream是在I/O线程里执行的，而downstream是在业务线程里执行。

当Netty从网络读取一个数据报投递给业务handler的时候，handler是在I/O线程里执行，而当我们在业务线程中调用write和writeAndFlush向网络发送消息的时候，handler是在业务线程里执行，直到最后一个Header handler将消息写入到发送队列中，业务线程才返回。

Netty4修改了这一模型，在Netty 4里inbound(对应Netty 3的upstream)和outbound(对应Netty 3的downstream)都是在NioEventLoop(I/O线程)中执行。

当我们在业务线程里通过ChannelHandlerContext.write发送消息的时候，Netty 4在将消息发送事件调度到ChannelPipeline的时候，首先将待发送的消息封装成一个Task，然后放到NioEventLoop的任务队列中，由NioEventLoop线程异步执行。

后续所有handler的调度和执行，包括消息的发送、I/O事件的通知，都由NioEventLoop线程负责处理。

### 3.1.2 gRPC线程模型

消息的序列化和反序列化均由gRPC线程负责，而没有在Netty的Handler中做CodeC，原因如下：Netty4优化了线程模型，所有业务Handler都由Netty的I/O线程负责，通过串行化的方式消除锁竞争，原理如下所示：

<img src="https://static001.geekbang.org/resource/image/90/ab/9011eef3e09d565dd844358134a571ab.png" alt="" />

如果大量的Handler都在Netty I/O线程中执行，一旦某些Handler执行比较耗时，则可能会反向影响I/O操作的执行，像序列化和反序列化操作，都是CPU密集型操作，更适合在业务应用线程池中执行，提升并发处理能力。因此，gRPC并没有在I/O线程中做消息的序列化和反序列化。

## 3.2 改进点思考

### 3.2.1 时间可控的接口调用直接在I/O线程上处理

gRPC采用的是网络I/O线程和业务调用线程分离的策略，大部分场景下该策略是最优的。但是，对于那些接口逻辑非常简单，执行时间很短，不需要与外部网元交互、访问数据库和磁盘，也不需要等待其它资源的，则建议接口调用直接在Netty /O线程中执行，不需要再投递到后端的服务线程池。避免线程上下文切换，同时也消除了线程并发问题。

例如提供配置项或者接口，系统默认将消息投递到后端服务调度线程，但是也支持短路策略，直接在Netty的NioEventLoop中执行消息的序列化和反序列化、以及服务接口调用。

### 3.2.2 减少锁竞争

当前gRPC的线程切换策略如下：

<img src="https://static001.geekbang.org/resource/image/bc/28/bc60a016b98fd462e8b969312a4a5428.png" alt="" />

优化之后的gRPC线程切换策略：

<img src="https://static001.geekbang.org/resource/image/82/c9/8205614f248c915610d102b52e95bbc9.png" alt="" />

通过线程绑定技术（例如采用一致性hash做映射）,将Netty的I/O线程与后端的服务调度线程做绑定，1个I/O线程绑定一个或者多个服务调用线程，降低锁竞争，提升性能。

### 3.2.3 关键点总结

RPC调用涉及的主要队列如下：

- Netty的消息发送队列（客户端和服务端都涉及）；
- gRPC SerializingExecutor的消息队列（JDK的BlockingQueue）；
- gRPC消息发送任务队列WriteQueue。

源代码下载地址：

链接: [https://github.com/geektime-geekbang/gRPC_LLF/tree/master](https://github.com/geektime-geekbang/gRPC_LLF/tree/master)
