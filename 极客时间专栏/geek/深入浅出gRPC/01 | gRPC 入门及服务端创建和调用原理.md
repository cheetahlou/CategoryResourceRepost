
# 1. RPC 入门

## 1.1 RPC框架原理

RPC框架的目标就是让远程服务调用更加简单、透明，RPC框架负责屏蔽底层的传输方式（TCP或者UDP）、序列化方式（XML/Json/二进制）和通信细节。服务调用者可以像调用本地接口一样调用远程的服务提供者，而不需要关心底层通信细节和调用过程。

RPC框架的调用原理图如下所示：

<img src="https://static001.geekbang.org/resource/image/b2/fb/b265dc0bd6eae1b88b236f517609c9fb.png" alt="" />

## 1.2 业界主流的RPC框架

业界主流的RPC框架整体上分为三类：

1. 支持多语言的RPC框架，比较成熟的有Google的gRPC、Apache（Facebook）的Thrift；
1. 只支持特定语言的RPC框架，例如新浪微博的Motan；
1. 支持服务治理等服务化特性的分布式服务框架，其底层内核仍然是RPC框架,例如阿里的Dubbo。

随着微服务的发展，基于语言中立性原则构建微服务，逐渐成为一种主流模式，例如对于后端并发处理要求高的微服务，比较适合采用Go语言构建，而对于前端的Web界面，则更适合Java和JavaScript。

因此，基于多语言的RPC框架来构建微服务，是一种比较好的技术选择。例如Netflix，API服务编排层和后端的微服务之间就采用gRPC进行通信。

## 1.3 gRPC简介

gRPC是一个高性能、开源和通用的RPC框架，面向服务端和移动端，基于HTTP/2设计。

### 1.3.1 gRPC概览

gRPC是由Google开发并开源的一种语言中立的RPC框架，当前支持C、Java和Go语言，其中C版本支持C、C++、Node.js、C#等。当前Java版本最新Release版为1.5.0，Git地址如下：

[https://github.com/grpc/grpc-java](https://github.com/grpc/grpc-java)

gRPC的调用示例如下所示：

<img src="https://static001.geekbang.org/resource/image/6d/d9/6d9a335ad96491e4d610a31b5089a2d9.png" alt="" />

### 1.3.2 gRPC特点

1. 语言中立，支持多种语言；
1. 基于IDL文件定义服务，通过proto3工具生成指定语言的数据结构、服务端接口以及客户端Stub；
1. 通信协议基于标准的HTTP/2设计，支持双向流、消息头压缩、单TCP的多路复用、服务端推送等特性，这些特性使得gRPC在移动端设备上更加省电和节省网络流量；
1. 序列化支持PB（Protocol Buffer）和JSON，PB是一种语言无关的高性能序列化框架，基于HTTP/2 + PB,保障了RPC调用的高性能。

# 2. gRPC服务端创建

以官方的helloworld为例，介绍gRPC服务端创建以及service调用流程（采用简单RPC模式）。

## 2.1 服务端创建业务代码

服务定义如下（helloworld.proto）：

```
service Greeter {
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}
message HelloRequest {
  string name = 1;
}
message HelloReply {
  string message = 1;
}


```

服务端创建代码如下（HelloWorldServer类）：

```
private void start() throws IOException {
    /* The port on which the server should run */
    int port = 50051;
    server = ServerBuilder.forPort(port)
        .addService(new GreeterImpl())
        .build()
        .start();
...

```

其中，服务端接口实现类（GreeterImpl）如下所示：

```
static class GreeterImpl extends GreeterGrpc.GreeterImplBase {
    @Override
    public void sayHello(HelloRequest req, StreamObserver&lt;HelloReply&gt; responseObserver) {
      HelloReply reply = HelloReply.newBuilder().setMessage(&quot;Hello &quot; + req.getName()).build();
      responseObserver.onNext(reply);
      responseObserver.onCompleted();
    }
  }

```

## 2.2 服务端创建流程

gRPC服务端创建采用Build模式，对底层服务绑定、transportServer和NettyServer的创建和实例化做了封装和屏蔽，让服务调用者不用关心RPC调用细节，整体上分为三个过程：

1. 创建Netty HTTP/2服务端；
1. 将需要调用的服务端接口实现类注册到内部的Registry中，RPC调用时，可以根据RPC请求消息中的服务定义信息查询到服务接口实现类；
1. 创建gRPC Server，它是gRPC服务端的抽象，聚合了各种Listener，用于RPC消息的统一调度和处理。

下面我们看下gRPC服务端创建流程：

<img src="https://static001.geekbang.org/resource/image/c6/37/c64c0e8e97711dc62e866861cd5c2e37.png" alt="" />

gRPC服务端创建关键流程分析：

<li>
**NettyServer实例创建：**gRPC服务端创建，首先需要初始化NettyServer，它是gRPC基于Netty 4.1 HTTP/2协议栈之上封装的HTTP/2服务端。NettyServer实例由NettyServerBuilder的buildTransportServer方法构建，NettyServer构建完成之后，监听指定的Socket地址，即可实现基于HTTP/2协议的请求消息接入。
</li>
<li>
**绑定IDL定义的服务接口实现类：**gRPC与其它一些RPC框架的差异点是服务接口实现类的调用并不是通过动态代理和反射机制，而是通过proto工具生成代码，在服务端启动时，将服务接口实现类实例注册到gRPC内部的服务注册中心上。请求消息接入之后，可以根据服务名和方法名，直接调用启动时注册的服务实例，而不需要通过反射的方式进行调用，性能更优。
</li>
<li>
**gRPC服务实例（ServerImpl）构建：**ServerImpl负责整个gRPC服务端消息的调度和处理，创建ServerImpl实例过程中，会对服务端依赖的对象进行初始化，例如Netty的线程池资源、gRPC的线程池、内部的服务注册类（InternalHandlerRegistry）等，ServerImpl初始化完成之后，就可以调用NettyServer的start方法启动HTTP/2服务端，接收gRPC客户端的服务调用请求。
</li>

## 2.3 服务端service调用流程

gRPC的客户端请求消息由Netty Http2ConnectionHandler接入，由gRPC负责将PB消息（或者JSON）反序列化为POJO对象，然后通过服务定义查询到该消息对应的接口实例，发起本地Java接口调用，调用完成之后，将响应消息序列化为PB（或者JSON），通过HTTP2 Frame发送给客户端。<br />
流程并不复杂，但是细节却比较多，整个service调用可以划分为如下四个过程：

1. gRPC请求消息接入；
1. gRPC消息头和消息体处理；
1. 内部的服务路由和调用；
1. 响应消息发送。

### 2.3.1 gRPC请求消息接入

gRPC的请求消息由Netty HTTP/2协议栈接入，通过gRPC注册的Http2FrameListener，将解码成功之后的HTTP Header和HTTP Body发送到gRPC的NettyServerHandler中，实现基于HTTP/2的RPC请求消息接入。

gRPC请求消息接入流程如下：

<img src="https://static001.geekbang.org/resource/image/b2/cf/b269a81ef5012a8ed5409e97c071eecf.png" alt="" />

关键流程解读如下：

<li>
Netty 4.1提供了HTTP/2底层协议栈，通过Http2ConnectionHandler及其依赖的其它类库，实现了HTTP/2消息的统一接入和处理。
通过注册Http2FrameListener监听器，可以回调接收HTTP2协议的消息头、消息体、优先级、Ping、SETTINGS等。
gRPC通过FrameListener重载Http2FrameListener的onDataRead、onHeadersRead等方法，将Netty的HTTP/2消息转发到gRPC的NettyServerHandler中；
</li>
<li>
Netty的HTTP/2协议接入仍然是通过ChannelHandler的CodeC机制实现，它并不影响NIO线程模型。
因此，理论上各种协议、以及同一个协议的多个服务端实例可以共用同一个NIO线程池（NioEventLoopGroup）,也可以独占。
在实践中独占模式普遍会存在线程资源占用过载问题，很容易出现句柄等资源泄漏。
在gRPC中，为了避免该问题，默认采用共享池模式创建NioEventLoopGroup，所有的gRPC服务端实例，都统一从SharedResourceHolder分配NioEventLoopGroup资源，实现NioEventLoopGroup的共享。
</li>

### 2.3.2 gRPC消息头和消息体处理

gRPC消息头的处理入口是NettyServerHandler的onHeadersRead()，处理流程如下所示：

<img src="https://static001.geekbang.org/resource/image/12/99/125c42c9d4f333e23b9896f478517099.png" alt="" />

处理流程如下：

<li>
对HTTP Header的Content-Type校验，此处必须是&quot;application/grpc&quot;；
</li>
<li>
从HTTP Header的URL中提取接口和方法名，以HelloWorldServer为例，它的method为：“helloworld.Greeter/SayHello”；
</li>
<li>
<p>将Netty的HTTP Header转换成gRPC内部的Metadata，Metadata内部维护了一个键值对的二维数组namesAndValues，以及一系列的类型转换方法（点击放大图片）：<br />
<img src="https://static001.geekbang.org/resource/image/7c/b9/7c833bc328f64ef864b430c45d68fab9.png" alt="" /></p>
</li>
<li>
创建NettyServerStream对象，它持有了Sink和TransportState类，负责将消息封装成GrpcFrameCommand，与底层Netty进行交互，实现协议消息的处理；
</li>
<li>
创建NettyServerStream之后，会触发ServerTransportListener的streamCreated方法，在该方法中，主要完成了消息上下文和gRPC业务监听器的创建；
</li>
<li>
gRPC上下文创建：CancellableContext创建之后，支持超时取消，如果gRPC客户端请求消息在Http Header中携带了“grpc-timeout”，系统在创建CancellableContext的同时会启动一个延时定时任务，延时周期为超时时间，一旦该定时器成功执行，就会调用CancellableContext.CancellationListener的cancel方法，发送CancelServerStreamCommand指令；
</li>
<li>
JumpToApplicationThreadServerStreamListener的创建：它是ServerImpl的内部类，从命名上基本可以看出它的用途，即从ServerStream跳转到应用线程中进行服务调用，gRPC服务端的接口调用主要通过JumpToApplicationThreadServerStreamListener的messageRead和halfClosed方法完成；
</li>
<li>
将NettyServerStream的TransportState缓存到Netty的Http2Stream中，当处理请求消息体时，可以根据streamId获取到Http2Stream，进而根据“streamKey”还原NettyServerStream的TransportState，进行后续处理。
</li>

gRPC消息体的处理入口是NettyServerHandler的onDataRead()，处理流程如下所示：

<img src="https://static001.geekbang.org/resource/image/20/9a/20c7a0f2ca1c9b801a406777283cb99a.png" alt="" />

消息体处理比较简单，下面就关键技术点进行讲解：

<li>
因为Netty HTTP/2协议Http2FrameListener分别提供了onDataRead和onHeadersRead回调方法，所以gRPC NettyServerHandler在处理完消息头之后需要缓存上下文，以便后续处理消息体时使用；
</li>
<li>
onDataRead和onHeadersRead方法都是由Netty的NIO线程负责调度，但是在执行onDataRead的过程中发生了线程切换，如下所示（ServerTransportListenerImpl类）：
</li>

```
wrappedExecutor.execute(new ContextRunnable(context) {
         @Override
         public void runInContext() {
           ServerStreamListener listener = NOOP_LISTENER;
           try {
             ServerMethodDefinition&lt;?, ?&gt; method = registry.lookupMethod(methodName);
             if (method == null) {
               method = fallbackRegistry.lookupMethod(methodName, stream.getAuthority());
             }

```

因此，实际上它们是并行+交叉串行实行的，后续章节介绍线程模型时会介绍切换原则。

### 2.3.3 内部的服务路由和调用

内部的服务路由和调用，主要包括如下几个步骤：

1. 将请求消息体反序列为Java的POJO对象，即IDL中定义的请求参数对象；
1. 根据请求消息头中的方法名到注册中心查询到对应的服务定义信息；
1. 通过Java本地接口调用方式，调用服务端启动时注册的IDL接口实现类。

具体流程如下所示：

<img src="https://static001.geekbang.org/resource/image/80/4d/808ed4d507d2f72624f80457b2d3ca4d.png" alt="" />

中间的交互流程比较复杂，涉及的类较多，但是关键步骤主要有三个：

<li>
**解码：**对HTTP/2 Body进行应用层解码，转换成服务端接口的请求参数，解码的关键就是调用requestMarshaller.parse(input)，将PB码流转换成Java对象；
</li>
<li>
**路由：**根据URL中的方法名从内部服务注册中心查询到对应的服务实例，路由的关键是调用registry.lookupMethod(methodName)获取到ServerMethodDefinition对象；
</li>
<li>
**调用：**调用服务端接口实现类的指定方法，实现RPC调用，与一些RPC框架不同的是，此处调用是Java本地接口调用，非反射调用，性能更优，它的实现关键是UnaryRequestMethod.invoke(request, responseObserver)方法。
</li>

### 2.3.4 响应消息发送

响应消息的发送由StreamObserver的onNext触发，流程如下所示：

<img src="https://static001.geekbang.org/resource/image/77/a6/77aedcd98c5910cad5f8d3e50cddc2a6.png" alt="" />

响应消息的发送原理如下：

<li>
分别发送gRPC HTTP/2响应消息头和消息体，由NettyServerStream的Sink将响应消息封装成SendResponseHeadersCommand和SendGrpcFrameCommand，加入到WriteQueue中；
</li>
<li>
<p>WriteQueue通过Netty的NioEventLoop线程进行消息处理，NioEventLoop将SendResponseHeadersCommand和SendGrpcFrameCommand写入到Netty的<br />
Channel中，进而触发DefaultChannelPipeline的<br />
write(Object msg,    ChannelPromise promise)操作；</p>
</li>
<li>
响应消息通过ChannelPipeline职责链进行调度，触发NettyServerHandler的sendResponseHeaders和sendGrpcFrame方法，调用Http2ConnectionEncoder的writeHeaders和writeData方法，将响应消息通过Netty的HTTP/2协议栈发送给客户端。
</li>

需要指出的是，请求消息的接收、服务调用以及响应消息发送，多次发生NIO线程和应用线程之间的互相切换，以及并行处理。因此上述流程中的一些步骤，并不是严格按照图示中的顺序执行的，后续线程模型章节，会做分析和介绍。

# 3. 源码分析

## 3.1 主要类和功能交互流程

### 3.1.1 gRPC请求消息头处理

<img src="https://static001.geekbang.org/resource/image/32/57/325ca29ddb2cf12eb96fa10af2cfe757.png" alt="" />

gRPC请求消息头处理涉及的主要类库如下：

1. **NettyServerHandler：**gRPC Netty Server的ChannelHandler实现，负责HTTP/2请求消息和响应消息的处理；
1. **SerializingExecutor：**应用调用线程池，负责RPC请求消息的解码、响应消息编码以及服务接口的调用等；
1. **MessageDeframer：**负责请求Framer的解析，主要用于处理HTTP/2 Header和Body的读取。
1. **ServerCallHandler：**真正的服务接口处理类，提供onMessage(ReqT request)和onHalfClose()方法，用于服务接口的调用。

### 3.1.2 gRPC请求消息体处理和服务调用

<img src="https://static001.geekbang.org/resource/image/37/8d/37a206a67f0f75990a01d64859df398d.png" alt="" />

### 3.1.3 gRPC响应消息处理

<img src="https://static001.geekbang.org/resource/image/df/d6/df4800c136c5097f6ffe68fb3306f3d6.png" alt="" />

需要说明的是，响应消息的发送由调用服务端接口的应用线程执行，在本示例中，由SerializingExecutor进行调用。

当请求消息头被封装成SendResponseHeadersCommand并被插入到WriteQueue之后，后续操作由Netty的NIO线程NioEventLoop负责处理。

应用线程继续发送响应消息体，将其封装成SendGrpcFrameCommand并插入到WriteQueue队列中，由Netty的NIO线程NioEventLoop处理。响应消息的发送严格按照顺序：即先消息头，后消息体。

## 3.2 源码分析

了解gRPC服务端消息接入和service调用流程之后，针对主要的流程和类库，进行源码分析，以加深对gRPC服务端工作原理的了解。

### 3.2.1 Netty服务端创建

基于Netty的HTTP/2协议栈，构建gRPC服务端，Netty HTTP/2协议栈初始化代码如下所示（创建NettyServerHandler，NettyServerHandler类）：

```
 frameWriter = new WriteMonitoringFrameWriter(frameWriter, keepAliveEnforcer);
    Http2ConnectionEncoder encoder = new DefaultHttp2ConnectionEncoder(connection, frameWriter);
    Http2ConnectionDecoder decoder = new FixedHttp2ConnectionDecoder(connection, encoder,
        frameReader);
    Http2Settings settings = new Http2Settings();
    settings.initialWindowSize(flowControlWindow);
    settings.maxConcurrentStreams(maxStreams);
    settings.maxHeaderListSize(maxHeaderListSize);
    return new NettyServerHandler(
        transportListener, streamTracerFactories, decoder, encoder, settings, maxMessageSize,
        keepAliveTimeInNanos, keepAliveTimeoutInNanos,
        maxConnectionAgeInNanos, maxConnectionAgeGraceInNanos,
        keepAliveEnforcer);

```

创建gRPC FrameListener，作为Http2FrameListener，监听HTTP/2消息的读取，回调到NettyServerHandler中（NettyServerHandler类）：

```
decoder().frameListener(new FrameListener());

```

将NettyServerHandler添加到Netty的ChannelPipeline中，接收和发送HTTP/2消息（NettyServerTransport类）：

```
ChannelHandler negotiationHandler = protocolNegotiator.newHandler(grpcHandler);
    channel.pipeline().addLast(negotiationHandler);

```

gRPC服务端请求和响应消息统一由NettyServerHandler拦截处理，相关方法如下：

<img src="https://static001.geekbang.org/resource/image/5b/ee/5b1c230a68d37379c544cbe7b59270ee.png" alt="" />

NettyServerHandler是gRPC应用侧和底层协议栈的桥接类，负责将原生的HTTP/2消息调度到gRPC应用侧，同时将应用侧的消息发送到协议栈。

### 3.2.2 服务实例创建和绑定

gRPC服务端启动时，需要将调用的接口实现类实例注册到内部的服务注册中心，用于后续的接口调用，关键代码如下（InternalHandlerRegistry类）：

```
Builder addService(ServerServiceDefinition service) {
      services.put(service.getServiceDescriptor().getName(), service);
      return this;
    }

```

服务接口绑定时，由Proto3工具生成代码，重载bindService()方法（GreeterImplBase类）：

```
@java.lang.Override public final io.grpc.ServerServiceDefinition bindService() {
      return io.grpc.ServerServiceDefinition.builder(getServiceDescriptor())
          .addMethod(
            METHOD_SAY_HELLO,
            asyncUnaryCall(
              new MethodHandlers&lt;
                io.grpc.examples.helloworld.HelloRequest,
                io.grpc.examples.helloworld.HelloReply&gt;(
                  this, METHODID_SAY_HELLO)))
          .build();
    }

```

### 3.2.3 service调用

<li>
<p>**gRPC消息的接收：**<br />
gRPC消息的接入由Netty HTTP/2协议栈回调gRPC的FrameListener，进而调用NettyServerHandler的onHeadersRead(ChannelHandlerContext ctx, int streamId, Http2Headers headers)和onDataRead(int streamId, ByteBuf data, int padding, boolean endOfStream)，代码如下所示：<br />
<img src="https://static001.geekbang.org/resource/image/90/8c/90c06120c24587569abd596cb889ac8c.png" alt="" /></p>
消息头和消息体的处理，主要由MessageDeframer的deliver方法完成，相关代码如下（MessageDeframer类）：
</li>

```
if (inDelivery) {
     return;
   }
   inDelivery = true;
   try {
          while (pendingDeliveries &gt; 0 &amp;&amp; readRequiredBytes()) {
       switch (state) {
         case HEADER:
           processHeader();
           break;
         case BODY:
           processBody();
           pendingDeliveries--;
           break;
         default:
           throw new AssertionError(&quot;Invalid state: &quot; + state);

```

gRPC请求消息（PB）的解码由PrototypeMarshaller负责，代码如下(ProtoLiteUtils类)：

```
public T parse(InputStream stream) {
       if (stream instanceof ProtoInputStream) {
         ProtoInputStream protoStream = (ProtoInputStream) stream;
         if (protoStream.parser() == parser) {
           try {
             T message = (T) ((ProtoInputStream) stream).message();
...

```

<li>**gRPC响应消息发送：**<br />
响应消息分为两部分发送：响应消息头和消息体，分别被封装成不同的WriteQueue.AbstractQueuedCommand，插入到WriteQueue中。<br />
消息头封装代码（NettyServerStream类）：</li>

```
public void writeHeaders(Metadata headers) {
     writeQueue.enqueue(new SendResponseHeadersCommand(transportState(),
         Utils.convertServerHeaders(headers), false),
         true);
   }

```

消息体封装代码（NettyServerStream类）：

```
ByteBuf bytebuf = ((NettyWritableBuffer) frame).bytebuf();
     final int numBytes = bytebuf.readableBytes();
     onSendingBytes(numBytes);
     writeQueue.enqueue(
         new SendGrpcFrameCommand(transportState(), bytebuf, false),
         channel.newPromise().addListener(new ChannelFutureListener() {
           @Override
           public void operationComplete(ChannelFuture future) throws Exception {
             transportState().onSentBytes(numBytes);
           }
         }), flush);

```

Netty的NioEventLoop将响应消息发送到ChannelPipeline，最终被NettyServerHandler拦截并处理。<br />
响应消息头处理代码如下（NettyServerHandler类）：

```
private void sendResponseHeaders(ChannelHandlerContext ctx, SendResponseHeadersCommand cmd,
     ChannelPromise promise) throws Http2Exception {
   int streamId = cmd.stream().id();
   Http2Stream stream = connection().stream(streamId);
   if (stream == null) {
     resetStream(ctx, streamId, Http2Error.CANCEL.code(), promise);
     return;
   }
   if (cmd.endOfStream()) {
     closeStreamWhenDone(promise, streamId);
   }
   encoder().writeHeaders(ctx, streamId, cmd.headers(), 0, cmd.endOfStream(), promise);
 }

```

响应消息体处理代码如下（NettyServerHandler类）：

```
private void sendGrpcFrame(ChannelHandlerContext ctx, SendGrpcFrameCommand cmd,
     ChannelPromise promise) throws Http2Exception {
   if (cmd.endStream()) {
     closeStreamWhenDone(promise, cmd.streamId());
   }
   encoder().writeData(ctx, cmd.streamId(), cmd.content(), 0, cmd.endStream(), promise);
 }

```

<li>服务接口实例调用：<br />
经过一系列预处理，最终由ServerCalls的ServerCallHandler调用服务接口实例，代码如下（ServerCalls类）：</li>

```
return new EmptyServerCallListener&lt;ReqT&gt;() {
         ReqT request;
         @Override
         public void onMessage(ReqT request) {
           this.request = request;
         }
         @Override
         public void onHalfClose() {
           if (request != null) {
             method.invoke(request, responseObserver);
             responseObserver.freeze();
             if (call.isReady()) {
               onReady();
             }

```

最终的服务实现类调用如下（GreeterGrpc类）：

```
public void invoke(Req request, io.grpc.stub.StreamObserver&lt;Resp&gt; responseObserver) {
     switch (methodId) {
       case METHODID_SAY_HELLO:
         serviceImpl.sayHello((io.grpc.examples.helloworld.HelloRequest) request,
 (io.grpc.stub.StreamObserver&lt;io.grpc.examples.helloworld.HelloReply&gt;) responseObserver);
         break;
       default:
         throw new AssertionError();
     }

```

## 3.3 服务端线程模型

gRPC的线程由Netty线程 + gRPC应用线程组成，它们之间的交互和切换比较复杂，下面做下详细介绍。

### 3.3.1 Netty Server线程模型

<img src="https://static001.geekbang.org/resource/image/7c/3c/7c205269a713a3b98033ac8760b3633c.png" alt="" />

它的工作流程总结如下：

<li>
从主线程池（bossGroup）中随机选择一个Reactor线程作为Acceptor线程，用于绑定监听端口，接收客户端连接；
</li>
<li>
Acceptor线程接收客户端连接请求之后创建新的SocketChannel，将其注册到主线程池（bossGroup）的其它Reactor线程上，由其负责接入认证、握手等操作；
</li>
<li>
步骤2完成之后，应用层的链路正式建立，将SocketChannel从主线程池的Reactor线程的多路复用器上摘除，重新注册到Sub线程池（workerGroup）的线程上，用于处理I/O的读写操作。
</li>

Netty Server使用的NIO线程实现是NioEventLoop，它的职责如下：

<li>
作为服务端Acceptor线程，负责处理客户端的请求接入；
</li>
<li>
作为客户端Connecor线程，负责注册监听连接操作位，用于判断异步连接结果；
</li>
<li>
作为I/O线程，监听网络读操作位，负责从SocketChannel中读取报文；
</li>
<li>
作为I/O线程，负责向SocketChannel写入报文发送给对方，如果发生写半包，会自动注册监听写事件，用于后续继续发送半包数据，直到数据全部发送完成；
</li>
<li>
作为定时任务线程，可以执行定时任务，例如链路空闲检测和发送心跳消息等；
</li>
<li>
作为线程执行器可以执行普通的任务Task（Runnable）。
</li>

### 3.3.2 gRPC service 线程模型

gRPC服务端调度线程为SerializingExecutor，它实现了Executor和Runnable接口，通过外部传入的Executor对象，调度和处理Runnable，同时内部又维护了一个任务队列ConcurrentLinkedQueue，通过run方法循环处理队列中存放的Runnable对象，代码示例如下：

<img src="https://static001.geekbang.org/resource/image/44/be/44d1dde76116b42ff1f110697b2e39be.png" alt="" />

### 3.3.3 线程调度和切换策略

Netty Server I/O线程的职责：

1. gRPC请求消息的读取、响应消息的发送
1. HTTP/2协议消息的编码和解码
1. NettyServerHandler的调度

gRPC service线程的职责：

1. 将gRPC请求消息（PB码流）反序列化为接口的请求参数对象
1. 将接口响应对象序列化为PB码流
1. gRPC服务端接口实现类调用

gRPC的线程模型遵循Netty的线程分工原则，即：协议层消息的接收和编解码由Netty的I/O(NioEventLoop)线程负责；后续应用层的处理由应用线程负责，防止由于应用处理耗时而阻塞Netty的I/O线程。

基于上述分工原则，在gRPC请求消息的接入和响应发送过程中，系统不断的在Netty I/O线程和gRPC应用线程之间进行切换。明白了分工原则，也就能够理解为什么要做频繁的线程切换。

gRPC线程模型存在的一个缺点，就是在一次RPC调用过程中，做了多次I/O线程到应用线程之间的切换，频繁切换会导致性能下降，这也是为什么gRPC性能比一些基于私有协议构建的RPC框架性能低的一个原因。尽管gRPC的性能已经比较优异，但是仍有一定的优化空间。

源代码下载地址：

链接:[https://github.com/geektime-geekbang/gRPC_LLF/tree/master](https://github.com/geektime-geekbang/gRPC_LLF/tree/master)
