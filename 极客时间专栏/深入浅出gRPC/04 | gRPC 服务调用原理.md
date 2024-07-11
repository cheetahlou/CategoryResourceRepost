
# 1. 常用的服务调用方式

无论是RPC框架，还是当前比较流行的微服务框架，通常都会提供多种服务调用方式，以满足不同业务场景的需求，比较常用的服务调用方式如下：

1. **同步服务调用：**最常用的服务调用方式，开发比较简单，比较符合编程人员的习惯，代码相对容易维护些；
1. **并行服务调用：**对于无上下文依赖的多个服务，可以一次并行发起多个调用，这样可以有效降低服务调用的时延；
1. **异步服务调用：**客户端发起服务调用之后，不同步等待响应，而是注册监听器或者回调函数，待接收到响应之后发起异步回调，驱动业务流程继续执行，比较常用的有 Reactive响应式编程和JDK的Future-Listener回调机制。

下面我们分别对上述几种服务调用方式进行讲解。

## 1.1 同步服务调用

同步服务调用是最常用的一种服务调用方式，它的工作原理和使用都非常简单，RPC/微服务框架默认都支持该调用形式。

同步服务调用的工作原理如下：客户端发起RPC调用，将请求消息路由到I/O线程，无论I/O线程是同步还是异步发送消息，发起调用的业务线程都会同步阻塞，等待服务端的应答，由I/O线程唤醒同步等待的业务线程，获取应答，然后业务流程继续执行。它的工作原理图如下所示：

<img src="https://static001.geekbang.org/resource/image/47/21/47c58251d2f8bd98bfbcd72bcd14e421.png" alt="" />

第1步，消费者调用服务端发布的接口，接口调用由服务框架包装成动态代理，发起远程服务调用。

第2步，通信框架的I/O线程通过网络将请求消息发送给服务端。

第3步，消费者业务线程调用通信框架的消息发送接口之后，直接或者间接调用wait()方法，同步阻塞等待应答。（备注：与步骤2无严格的顺序先后关系，不同框架实现策略不同）

第4步，服务端返回应答消息给消费者，由通信框架负责应答消息的反序列化。

第5步，I/O线程获取到应答消息之后，根据消息上下文找到之前同步阻塞的业务线程，notify()阻塞的业务线程，返回应答给消费者，完成服务调用。

同步服务调用会阻塞调用方的业务线程，为了防止服务端长时间不返回应答消息导致客户端用户线程被挂死，业务线程等待的时候需要设置超时时间，超时时间的取值需要综合考虑业务端到端的时延控制目标，以及自身的可靠性，超时时间不宜设置过大或者过小,通常在几百毫秒到几秒之间。

## 1.2 并行服务调用

在大多数业务应用中，服务总是被串行的调用和执行，例如A业务调用B服务，B服务又调用C服务，最后形成一个串行的服务调用链：A业务 — B服务 — C服务…

串行服务调用代码比较简单，便于开发和维护，但在一些时延敏感型的业务场景中，需要采用并行服务调用来降低时延，比较典型的场景如下：

1. 多个服务之间逻辑上不存在上下文依赖关系，执行先后顺序没有严格的要求，逻辑上可以被并行执行；
1. 长流程业务，调用多个服务，对时延比较敏感，其中有部分服务逻辑上无上下文关联，可以被并行调用。

并行服务调用的主要目标有两个：

1. 降低业务E2E时延。
1. 提升整个系统的吞吐量。

以游戏业务中购买道具流程为例，对并行服务调用的价值进行说明：

<img src="https://static001.geekbang.org/resource/image/97/7e/97618a016f16a63217b53f9d3ff3c07e.png" alt="" />

在购买道具时，三个鉴权流程实际可以并行执行，最终执行结果做个Join即可。如果采用传统的串行服务调用，耗时将是三个鉴权服务时延之和，显然是没有必要的。

计费之后的通知类服务亦如此（注意：通知服务也可以使用MQ做订阅/发布），单个服务的串行调用会导致购买道具时延比较长，影响游戏玩家的体验。

要解决串行调用效率低的问题，有两个解决对策：

1. 并行服务调用，一次I/O操作，可以发起批量调用，然后同步等待响应；
1. 异步服务调用，在同一个业务线程中异步执行多个服务调用，不阻塞业务线程。

采用并行服务调用的伪代码示例：

```
ParallelFuture future = ParallelService.invoke(serviceName [], methodName[], args []);
List&lt;Object&gt; results = future.get(timeout);//同步阻塞式获取批量服务调用的响应列表

```

采用并行服务调用之后，它的总耗时为并行服务调用中耗时最大的服务的执行时间，即 T = Max(T(服务A)，T(服务B)，T(服务C))，如果采用同步串行服务调用，则总耗时为并行调用的各个服务耗时之和，即：T = T(服务A) + T(服务B) + T(服务C）。服务调用越多，并行服务调用的优势越明显。

并行服务调用的一种实现策略如下所示：

<img src="https://static001.geekbang.org/resource/image/c1/bb/c1585171726fbb13dfbf488b52ba34bb.png" alt="" />

第1步，服务框架提供批量服务调用接口供消费者使用，它的定义样例如下：ParallelService.invoke(serviceName [], methodName[], args [])；

第2步，平台的并行服务调用器创建并行Future，缓存批量服务调用上下文信息；

第3步，并行服务调用器循环调用普通的Invoker，通过循环的方式执行单个服务调用，获取到单个服务的Future之后设置到Parallel Future中；

第4步，返回Parallel Future给消费者；

第5步，普通Invoker调用通信框架的消息发送接口，发起远程服务调用；

第6步，服务端返回应答，通信框架对报文做反序列化，转换成业务对象更新Parallel Future的结果列表；

第7步，消费者调用Parallel Future的get(timeout)方法, 同步阻塞，等待所有结果全部返回；

第8步，Parallel Future通过对结果集进行判断，看所有服务调用是否都已经完成（包括成功、失败和异常）；

第9步，所有批量服务调用结果都已经返回，notify消费者线程，消费者获取到结果列表，完成批量服务调用，流程继续执行。

通过批量服务调用+ Future机制，可以实现并行服务调用，由于在调用过程中没有创建新的线程，用户就不需要担心依赖线程上下文的功能发生异常。

## 1.3 异步服务调用

JDK原生的Future主要用于异步操作，它代表了异步操作的执行结果，用户可以通过调用它的get方法获取结果。如果当前操作没有执行完，get操作将阻塞调用线程:

<img src="https://static001.geekbang.org/resource/image/ce/76/cefed1d339b00c88b8f9548b45bb5d76.png" alt="" />

在实际项目中，往往会扩展JDK的Future，提供Future-Listener机制，它支持主动获取和被动异步回调通知两种模式，适用于不同的业务场景。

异步服务调用的工作原理如下：

<img src="https://static001.geekbang.org/resource/image/fc/69/fc8debf2fe872fe0398a9fa2f50bcd69.png" alt="" />

异步服务调用的工作流程如下：

第1步，消费者调用服务端发布的接口，接口调用由服务框架包装成动态代理，发起远程服务调用；

第2步，通信框架异步发送请求消息，如果没有发生I/O异常，返回；

第3步，请求消息发送成功后，I/O线程构造Future对象，设置到RPC上下文中；

第4步，业务线程通过RPC上下文获取Future对象；

第5步，构造Listener对象，将其添加到Future中，用于服务端应答异步回调通知；

第6步，业务线程返回，不阻塞等待应答；

第7步，服务端返回应答消息，通信框架负责反序列化等；

第8步，I/O线程将应答设置到Future对象的操作结果中；

第9步，Future对象扫描注册的监听器列表，循环调用监听器的operationComplete方法，将结果通知给监听器，监听器获取到结果之后，继续后续业务逻辑的执行，异步服务调用结束。

异步服务调用相比于同步服务调用有两个优点：

1. 化串行为并行，提升服务调用效率，减少业务线程阻塞时间。
1. 化同步为异步，避免业务线程阻塞。

基于Future-Listener的纯异步服务调用代码示例如下：

```
xxxService1.xxxMethod(Req);
Future f1 = RpcContext.getContext().getFuture();
Listener l = new xxxListener();
f1.addListener(l);
class xxxListener{
public void operationComplete(F future)
{ //判断是否执行成功，执行后续业务流程}
     }

```

# 服务调用的一些误区和典型问题

对于服务调用方式的理解，容易出现各种误区，例如把I/O异步与服务调用的异步混淆起来，认为异步服务调用一定性能更高等。

另外，由于Restful风格API的盛行，很多RPC/微服务框架开始支持Restful API，而且通常都是基于HTTP/1.0/1.1协议实现的。对于内部的RPC调用，使用HTTP/1.0/1.1协议代替TCP私有协议，可能会带来一些潜在的性能风险，需要在开放性、标准性以及性能成本上综合考虑，谨慎选择。

## 2.1 理解误区

### 2.1.1 I/O异步服务就是异步

实际上，通信框架基于NIO实现，并不意味着服务框架就支持异步服务调用了，两者本质上不是同一个层面的事情。在RPC/微服务框架中，引入NIO带来的好处是显而易见的：

- 所有的I/O操作都是非阻塞的，避免有限的I/O线程因为网络、对方处理慢等原因被阻塞；
- 多路复用的Reactor线程模型：基于Linux的epoll和Selector，一个I/O线程可以并行处理成百上千条链路，解决了传统同步I/O通信线程膨胀的问题。

NIO只解决了通信层面的异步问题，跟服务调用的异步没有必然关系，也就是说，即便采用传统的BIO通信，依然可以实现异步服务调用，只不过通信效率和可靠性比较差而已。

对异步服务调用和通信框架的关系进行说明：<br />
<img src="https://static001.geekbang.org/resource/image/a6/b8/a69619ff7aad67e7a714925c69642eb8.png" alt="" />

用户发起远程服务调用之后，经历层层业务逻辑处理、消息编码，最终序列化后的消息会被放入到通信框架的消息队列中。业务线程可以选择同步等待、也可以选择直接返回，通过消息队列的方式实现业务层和通信层的分离是比较成熟、典型的做法，目前主流的RPC框架或者Web服务器很少直接使用业务线程进行网络读写。

通过上图可以看出，采用NIO还是BIO对上层的业务是不可见的，双方的汇聚点就是消息队列，在Java实现中它通常就是个Queue。业务线程将消息放入到发送队列中，可以选择主动等待或者立即返回，跟通信框架是否是NIO没有任何关系。因此不能认为I/O异步就代表服务调用也是异步的。

### 2.1.2 服务调用天生就是同步的

RPC/微服务框架的一个目标就是让用户像调用本地方法一样调用远程服务，而不需要关心服务提供者部署在哪里，以及部署形态（透明化调用）。由于本地方法通常都是同步调用，所以服务调用也应该是同步的。

从服务调用形式上看，主要包含3种：

1. **one way方式：**只有请求，没有应答。例如通知消息。
1. **请求-响应方式：**一请求一应答，这种方式比较常用。
1. **请求-响应-异步通知方式：**客户端发送请求之后，服务端接收到就立即返回应答，类似TCP的ACK。业务接口层面的响应通过异步通知的方式告知请求方。例如电商类的支付接口、充值缴费接口等。

OneWay方式的调用示意图如下：

请求-应答模式最常用，例如HTTP协议，就是典型的请求-应答方式：

请求-响应-异步通知方式流程：通过流程设计，将执行时间可能较长的服务接口从流程上设计成异步。通常在服务调用时请求方携带回调的通知地址，服务端接收到请求之后立即返回应答，表示该请求已经被接收处理。当服务调用真正完成之后，再通过回调地址反向调用服务消费端，将响应通知异步返回。通过接口层的异步，来实现整个服务调用流程的异步，降低了异步调用的开发难度。

One way方式的服务调用由于不需要返回应答，因此也很容易被设计成异步的：消费者发起远程服务调用之后，立即返回，不需要同步阻塞等待应答。

对于请求-响应方式，一般的观点都认为消费者必需要等待服务端响应，拿到结果之后才能返回，否则结果从哪里取？即便业务线程不阻塞，没有获取到结果流程还是无法继续执行下去。

从逻辑上看，上述观点没有问题。但实际上，同步阻塞等待应答并非是唯一的技术选择，我们也可以利用Java的Future-Listener机制来实现异步服务调用。从业务角度看，它的效果与同步等待等价，但是从技术层面看，却是个很大的进步，它可以保证业务线程在不同步阻塞的情况下实现同步等待的效果，服务执行效率更高。

即接口层面请求-响应式定义与具体的技术实现无关，选择同步还是异步服务调用，取决于技术实现。当然，异步通知类接口，从技术实现上做异步更容易些。

### 2.1.3 异步服务调用性能更高

对于I/O密集型，资源不是瓶颈，大部分时间都在同步等应答的场景，异步服务调用会带来巨大的吞吐量提升，资源使用率也可以提高，更加充分的利用硬件资源提升性能。

另外，对于时延不稳定的接口，例如依赖第三方服务的响应速度、数据库操作类等，通常异步服务调用也会带来性能提升。

但是，如果接口调用时延本身都非常小（例如毫秒级），内存计算型，不依赖第三方服务，内部也没有I/O操作，则异步服务调用并不会提升性能。能否提升性能，主要取决于业务的应用场景。

## 2.2 Restful API的潜在性能风险

使用Restful API可以带来很多收益：

- API接口更加规范和标准，可以通过Swagger API规范来描述服务接口，并生成客户端和服务端代码；
- Restful API可读性更好，也更容易维护；
- 服务提供者和消费者基于API契约，双方可以解耦，不需要在客户端引入SDK和类库的直接依赖，未来的独立升级也更方便；
- 内外可以使用同一套API，非常容易开放给外部或者合作伙伴使用，而不是对内和对外维护两套不同协议的API。

通常，对外开放的API使用Restful是通用的做法，但是在系统内部，例如商品中心和订单中心，RPC调用使用 Restful风格的API作为微服务的API，却可能存在性能风险。

### 2.2.1 HTTP1.X的性能问题

如果HTTP服务器采用同步阻塞I/O，例如Tomcat5.5之前的BIO模型，如下图所示：

<img src="https://static001.geekbang.org/resource/image/08/b0/08deb9535575670243d3793f3dc2f1b0.png" alt="" />

就会存在如下几个问题：

1. **性能问题：**一连接一线程模型导致服务端的并发接入数和系统吞吐量受到极大限制；
1. **可靠性问题：**由于I/O操作采用同步阻塞模式，当网络拥塞或者通信对端处理缓慢会导致I/O线程被挂住，阻塞时间无法预测；
1. **可维护性问题：**I/O线程数无法有效控制、资源无法有效共享（多线程并发问题），系统可维护性差。

显然，如果采用的Restful API 底层使用的HTTP协议栈是同步阻塞I/O，则服务端的处理性能将大打折扣。

### 2.2.2 异步非阻塞I/O的HTTP协议栈

如果HTTP协议栈采用了异步非阻塞I/O模型（例如Netty、Servlet3.X版本），则可以解决同步阻塞I/O的问题，带来如下收益：

- 同一个I/O线程可以并行处理多个客户端链接，有效降低了I/O线程数量，提升了资源调度利用率；
- 读写操作都是非阻塞的，不会因为对端处理慢、网络时延大等导致的I/O线程被阻塞。

相比于TCP类协议，例如Thrift, 采用了非阻塞I/O的HTTP/1.X协议仍然存在性能问题，原因如下所示：

<img src="https://static001.geekbang.org/resource/image/df/71/df88e6d9be74aff50bc9461e5bb69971.png" alt="" />

由于HTTP协议是无状态的，客户端发送请求之后，必须等待接收到服务端响应之后，才能继续发送请求（非websocket、pipeline等模式）。

在某一个时刻，链路上只存在单向的消息流，实际上把TCP的双工变成了单工模式。如果服务端响应耗时较大，则单个HTTP链路的通信性能严重下降，只能通过不断的新建连接来提升I/O性能。

但这也会带来很多副作用，例如句柄数的增加、I/O线程的负载加重等。显而易见，修8条单向车道的成本远远高于修一条双向8车道的成本。

除了无状态导致的链路传输性能差之外，HTTP/1.X还存在如下几个影响性能的问题：

<li>
HTTP客户端超时之后，由于协议是无状态的，客户端无法对请求和响应进行关联，只能关闭链路重连，反复的建链会增加成本开销和时延（如果客户端选择不关闭链路，继续发送新的请求，服务端可能会把上一条客户端认为超时的响应返回回去，也可能按照HTTP协议规范直接关闭链路，无路哪种处理，都会导致链路被关闭）。如果采用传统的RPC私有协议，请求和响应可以通过消息ID或者会话ID做关联，某条消息的超时并不需要关闭链路，只需要丢弃该消息重发即可。
</li>
<li>
HTTP本身包含文本类型的协议消息头，占用一些字节，另外，采用JSON类文本的序列化方式，报文相比于传统的私有RPC协议也大很多，降低了传输性能。
</li>
<li>
服务端无法主动推送响应。
</li>

如果业务对性能和资源成本要求非常苛刻，在选择使用基于HTTP/1.X的Restful API 代替私有RPC API（通常是基于TCP的二进制私有协议）时就要三思。反之，如果业务对性能要求较低，或者在硬件成本和开放性、规范性上更看重后者，则使用Restful API也无妨。

### 2.2.3 推荐解决方案

如果选择Restful API作为内部RPC或者微服务的接口协议，则建议使用HTTP/2.0协议来承载，它的优点如下：支持双向流、消息头压缩、单TCP的多路复用、服务端推送等特性。可以有效解决传统HTTP/1.X协议遇到的问题，效果与RPC的TCP私有协议接近。

# 3. gRPC服务调用

gRPC的通信协议基于标准的HTTP/2设计，主要提供了两种RPC调用方式：

1. 普通RPC调用方式，即请求-响应模式。
1. 基于HTTP/2.0的streaming调用方式。

## 3.1 普通RPC调用

普通的RPC调用提供了三种实现方式：

1. 同步阻塞式服务调用，通常实现类是xxxBlockingStub（基于proto定义生成）。
1. 异步非阻塞调用，基于Future-Listener机制，通常实现类是xxxFutureStub。
1. 异步非阻塞调用，基于Reactive的响应式编程模式，通常实现类是xxxStub。

### 3.1.1 同步阻塞式RPC调用

同步阻塞式服务调用，代码示例如下（HelloWorldClient类）：

```
 blockingStub = GreeterGrpc.newBlockingStub(channel);
 ...
 HelloRequest request 
= HelloRequest.newBuilder().setName(name).build();
    HelloReply response;
    try {
      response = blockingStub.sayHello(request);
...

```

创建GreeterBlockingStub，然后调用它的sayHello，此时会阻塞调用方线程（例如main函数），直到收到服务端响应之后，业务代码继续执行，打印响应日志。

实际上，同步服务调用是由gRPC框架的ClientCalls在框架层做了封装，异步发起服务调用之后，同步阻塞调用方线程，直到收到响应再唤醒被阻塞的业务线程，源码如下（ClientCalls类）：

```
try {
      ListenableFuture&lt;RespT&gt; responseFuture = futureUnaryCall(call, param);
      while (!responseFuture.isDone()) {
        try {
          executor.waitAndDrain();
        } catch (InterruptedException e) {
          Thread.currentThread().interrupt();
          throw Status.CANCELLED.withCause(e).asRuntimeException();
        }

```

判断当前操作是否完成，如果没完成，则在ThreadlessExecutor中阻塞（阻塞调用方线程，ThreadlessExecutor不是真正的线程池），代码如下（ThreadlessExecutor类）：

```
Runnable runnable = queue.take();
      while (runnable != null) {
        try {
          runnable.run();
        } catch (Throwable t) {
          log.log(Level.WARNING, &quot;Runnable threw exception&quot;, t);
        }
        runnable = queue.poll();
      }

```

调用queue的take方法会阻塞，直到队列中有消息(响应)，才会继续执行（BlockingQueue类）：

```
/**
     * Retrieves and removes the head of this queue, waiting if necessary
     * until an element becomes available.
     *
     * @return the head of this queue
     * @throws InterruptedException if interrupted while waiting
     */
    E take() throws InterruptedException;

```

### 3.1.2 基于Future的异步RPC调用

业务调用代码示例如下（HelloWorldClient类）：

```
HelloRequest request 
= HelloRequest.newBuilder().setName(name).build();
    try {
   com.google.common.util.concurrent.ListenableFuture&lt;io.grpc.examples.helloworld.HelloReply&gt;
              listenableFuture = futureStub.sayHello(request);
      Futures.addCallback(listenableFuture, new FutureCallback&lt;HelloReply&gt;() {
        @Override
        public void onSuccess(@Nullable HelloReply result) {
          logger.info(&quot;Greeting: &quot; + result.getMessage());
        }

```

调用GreeterFutureStub的sayHello方法返回的不是应答，而是ListenableFuture，它继承自JDK的Future，接口定义如下：

```
@GwtCompatible
public interface ListenableFuture&lt;V&gt; extends Future&lt;V&gt; 

```

将ListenableFuture加入到gRPC的Future列表中，创建一个新的FutureCallback对象，当ListenableFuture获取到响应之后，gRPC的DirectExecutor线程池会调用新创建的FutureCallback，执行onSuccess或者onFailure，实现异步回调通知。

接着我们分析下ListenableFuture的实现原理，ListenableFuture的具体实现类是GrpcFuture，代码如下（ClientCalls类）：

```
public static &lt;ReqT, RespT&gt; ListenableFuture&lt;RespT&gt; futureUnaryCall(
      ClientCall&lt;ReqT, RespT&gt; call,
      ReqT param) {
    GrpcFuture&lt;RespT&gt; responseFuture = new GrpcFuture&lt;RespT&gt;(call);
    asyncUnaryRequestCall(call, param, new UnaryStreamToFuture&lt;RespT&gt;(responseFuture), false);
    return responseFuture;
  }

```

获取到响应之后，调用complete方法（AbstractFuture类）：

```
private void complete() {
    for (Waiter currentWaiter = clearWaiters();
        currentWaiter != null;
        currentWaiter = currentWaiter.next) {
      currentWaiter.unpark();
    }

```

将ListenableFuture加入到Future列表中之后，同步获取响应（在gRPC线程池中阻塞，非业务调用方线程）（Futures类）：

```
public static &lt;V&gt; V getUninterruptibly(Future&lt;V&gt; future)
      throws ExecutionException {
    boolean interrupted = false;
    try {
      while (true) {
        try {
          return future.get();
        } catch (InterruptedException e) {
          interrupted = true;
        }
      }

```

获取到响应之后，回调callback的onSuccess，代码如下（Futures类）：

```
 value = getUninterruptibly(future);
        } catch (ExecutionException e) {
          callback.onFailure(e.getCause());
          return;
        } catch (RuntimeException e) {
          callback.onFailure(e);
          return;
        } catch (Error e) {
          callback.onFailure(e);
          return;
        }
        callback.onSuccess(value);

```

除了将ListenableFuture加入到Futures中由gRPC的线程池执行异步回调，也可以自定义线程池执行异步回调，代码示例如下（HelloWorldClient类）：

```
listenableFuture.addListener(new Runnable() {
        @Override
        public void run() {
          try {
            HelloReply response = listenableFuture.get();
            logger.info(&quot;Greeting: &quot; + response.getMessage());
          }
          catch(Exception e)
          {
            e.printStackTrace();
          }
        }
      }, Executors.newFixedThreadPool(1));


```

### 3.1.3 Reactive风格异步RPC调用

业务调用代码示例如下（HelloWorldClient类）：

```
HelloRequest request = HelloRequest.newBuilder().setName(name).build();
    io.grpc.stub.StreamObserver&lt;io.grpc.examples.helloworld.HelloReply&gt; responseObserver =
            new io.grpc.stub.StreamObserver&lt;io.grpc.examples.helloworld.HelloReply&gt;()
            {
                public  void onNext(HelloReply value)
               {
                   logger.info(&quot;Greeting: &quot; + value.getMessage());
               }
                public void onError(Throwable t){
                    logger.warning(t.getMessage());
                }
                public void onCompleted(){}
            };
           stub.sayHello(request,responseObserver);

```

构造响应StreamObserver，通过响应式编程，处理正常和异常回调，接口定义如下：

<img src="https://static001.geekbang.org/resource/image/45/1d/45b9e8b4dea5ff3498533097808f2e1d.png" alt="" />

将响应StreamObserver作为入参传递到异步服务调用中，该方法返回空，程序继续向下执行，不阻塞当前业务线程，代码如下所示（GreeterGrpc.GreeterStub）：

```
public void sayHello(io.grpc.examples.helloworld.HelloRequest request,
        io.grpc.stub.StreamObserver&lt;io.grpc.examples.helloworld.HelloReply&gt; responseObserver) {
      asyncUnaryCall(
          getChannel().newCall(METHOD_SAY_HELLO, getCallOptions()), request, responseObserver);
    }

```

下面分析下基于Reactive方式异步调用的代码实现，把响应StreamObserver对象作为入参传递到异步调用中，代码如下(ClientCalls类)：

```
 private static &lt;ReqT, RespT&gt; void asyncUnaryRequestCall(
      ClientCall&lt;ReqT, RespT&gt; call, ReqT param, StreamObserver&lt;RespT&gt; responseObserver,
      boolean streamingResponse) {
    asyncUnaryRequestCall(call, param,
        new StreamObserverToCallListenerAdapter&lt;ReqT, RespT&gt;(call, responseObserver,
            new CallToStreamObserverAdapter&lt;ReqT&gt;(call),
            streamingResponse),
        streamingResponse);
  }

```

当收到响应消息时，调用StreamObserver的onNext方法，代码如下（StreamObserverToCallListenerAdapter类）：

```
public void onMessage(RespT message) {
      if (firstResponseReceived &amp;&amp; !streamingResponse) {
        throw Status.INTERNAL
            .withDescription(&quot;More than one responses received for unary or client-streaming call&quot;)
            .asRuntimeException();
      }
      firstResponseReceived = true;
      observer.onNext(message);

```

当Streaming关闭时，调用onCompleted方法，如下所示（StreamObserverToCallListenerAdapter类）：

```
 public void onClose(Status status, Metadata trailers) {
      if (status.isOk()) {
        observer.onCompleted();
      } else {
        observer.onError(status.asRuntimeException(trailers));
      }
    }

```

通过源码分析可以发现，Reactive风格的异步调用，相比于Future模式，没有任何同步阻塞点，无论是业务线程还是gRPC框架的线程都不会同步等待，相比于Future异步模式，Reactive风格的调用异步化更彻底一些。

## 3.2 Streaming模式服务调用

基于HTTP/2.0,gRPC提供了三种streaming模式：

1. 服务端streaming
1. 客户端streaming
1. 服务端和客户端双向streaming

### 3.2.1 服务端streaming

服务端streaming模式指客户端1个请求，服务端返回N个响应，每个响应可以单独的返回，它的原理如下所示：

<img src="https://static001.geekbang.org/resource/image/f4/26/f4989ade79af24dcaa46d05ac62cae26.png" alt="" />

适用的场景主要是客户端发送单个请求，但是服务端可能返回的是一个响应列表，服务端不想等到所有的响应列表都组装完成才返回应答给客户端，而是处理完成一个就返回一个响应，直到服务端关闭stream，通知客户端响应全部发送完成。

在实际业务中，应用场景还是比较多的，最典型的如SP短信群发功能，如果不使用streaming模式，则原群发流程如下所示：

<img src="https://static001.geekbang.org/resource/image/27/e6/2742f5dc95f8032a082bd483504594e6.png" alt="" />

采用gRPC服务端streaming模式之后，流程优化如下：

<img src="https://static001.geekbang.org/resource/image/46/c2/46efe67ea4301bd4cfce2e824e86ccc2.png" alt="" />

实际上，不同用户之间的短信下发和通知是独立的，不需要互相等待，采用streaming模式之后，单个用户的体验会更好。

服务端streaming模式的本质就是如果响应是个列表，列表中的单个响应比较独立，有些耗时长，有些耗时短，为了防止快的等慢的，可以处理完一个就返回一个，不需要等所有的都处理完才统一返回响应。可以有效避免客户端要么在等待，要么需要批量处理响应，资源使用不均的问题，也可以压缩单个响应的时延，端到端提升用户的体验（时延敏感型业务）。

像请求-响应-异步通知类业务，也比较适合使用服务端streaming模式。它的proto文件定义如下所示：

```
 rpc ListFeatures(Rectangle) returns (stream Feature) {}

```

下面一起看下业务示例代码：

<img src="https://static001.geekbang.org/resource/image/7f/0f/7faa1b53676125cfaa32b1dae7c9ec0f.png" alt="" />

服务端Sreaming模式也支持同步阻塞和Reactive异步两种调用方式，以Reactive异步为例，它的代码实现如下（RouteGuideImplBase类）：

```
public void listFeatures(io.grpc.examples.routeguide.Rectangle request,
        io.grpc.stub.StreamObserver&lt;io.grpc.examples.routeguide.Feature&gt; responseObserver) {
      asyncUnimplementedUnaryCall(METHOD_LIST_FEATURES, responseObserver);
    }

```

构造io.grpc.stub.StreamObserver&lt;io.grpc.examples.routeguide.Feature&gt; responseObserver，实现它的三个回调接口，注意由于是服务端streaming模式，所以它的onNext(Feature value)将会被回调多次，每次都代表一个响应，如果所有的响应都返回，则会调用onCompleted()方法。

### 3.2.2 客户端streaming

与服务端streaming类似，客户端发送多个请求，服务端返回一个响应，多用于汇聚和汇总计算场景，proto文件定义如下：

```
rpc RecordRoute(stream Point) returns (RouteSummary) {}

```

业务调用代码示例如下（RouteGuideClient类）：

```
StreamObserver&lt;Point&gt; requestObserver = asyncStub.recordRoute(responseObserver);
    try {
      // Send numPoints points randomly selected from the features list.
      for (int i = 0; i &lt; numPoints; ++i) {
        int index = random.nextInt(features.size());
        Point point = features.get(index).getLocation();
        info(&quot;Visiting point {0}, {1}&quot;, RouteGuideUtil.getLatitude(point),
            RouteGuideUtil.getLongitude(point));
        requestObserver.onNext(point);
...

```

异步服务调用获取请求StreamObserver对象，循环调用requestObserver.onNext(point)，异步发送请求消息到服务端，发送完成之后，调用requestObserver.onCompleted()，通知服务端所有请求已经发送完成，可以接收服务端的响应了。

响应接收的代码如下所示：由于响应只有一个，所以onNext只会被调用一次（RouteGuideClient类）：

```
StreamObserver&lt;RouteSummary&gt; responseObserver = new StreamObserver&lt;RouteSummary&gt;() {
      @Override
      public void onNext(RouteSummary summary) {
        info(&quot;Finished trip with {0} points. Passed {1} features. &quot;
            + &quot;Travelled {2} meters. It took {3} seconds.&quot;, summary.getPointCount(),
            summary.getFeatureCount(), summary.getDistance(), summary.getElapsedTime());
        if (testHelper != null) {
          testHelper.onMessage(summary);
        }

```

异步服务调用时，将响应StreamObserver实例作为参数传入，代码如下：

```
StreamObserver&lt;Point&gt; requestObserver = asyncStub.recordRoute(responseObserver);

```

### 3.2.3 双向streaming

客户端发送N个请求，服务端返回N个或者M个响应，利用该特性，可以充分利用HTTP/2.0的多路复用功能，在某个时刻，HTTP/2.0链路上可以既有请求也有响应，实现了全双工通信（对比单行道和双向车道），示例如下：

proto文件定义如下：

```
rpc RouteChat(stream RouteNote) returns (stream RouteNote) {}

```

业务代码示例如下（RouteGuideClient类）：

```
StreamObserver&lt;RouteNote&gt; requestObserver =
        asyncStub.routeChat(new StreamObserver&lt;RouteNote&gt;() {
          @Override
          public void onNext(RouteNote note) {
            info(&quot;Got message \&quot;{0}\&quot; at {1}, {2}&quot;, note.getMessage(), note.getLocation()
                .getLatitude(), note.getLocation().getLongitude());
            if (testHelper != null) {
              testHelper.onMessage(note);
            }
          }

```

构造Streaming响应对象StreamObserver<RouteNote>并实现onNext等接口，由于服务端也是Streaming模式，因此响应是多个的，也就是说onNext会被调用多次。<br />
通过在循环中调用requestObserver的onNext方法，发送请求消息，代码如下所示（RouteGuideClient类）：

```
for (RouteNote request : requests) {
        info(&quot;Sending message \&quot;{0}\&quot; at {1}, {2}&quot;, request.getMessage(), request.getLocation()
            .getLatitude(), request.getLocation().getLongitude());
        requestObserver.onNext(request);
      }
    } catch (RuntimeException e) {
      // Cancel RPC
      requestObserver.onError(e);
      throw e;
    }
    // Mark the end of requests
    requestObserver.onCompleted();

```

requestObserver的onNext方法实际调用了ClientCall的消息发送方法，代码如下（CallToStreamObserverAdapter类）：

```
private static class CallToStreamObserverAdapter&lt;T&gt; extends ClientCallStreamObserver&lt;T&gt; {
    private boolean frozen;
    private final ClientCall&lt;T, ?&gt; call;
    private Runnable onReadyHandler;
    private boolean autoFlowControlEnabled = true;
    public CallToStreamObserverAdapter(ClientCall&lt;T, ?&gt; call) {
      this.call = call;
    }
    private void freeze() {
      this.frozen = true;
    }
    @Override
    public void onNext(T value) {
      call.sendMessage(value);
    }

```

对于双向Streaming模式，只支持异步调用方式。

## 3.3 总结

gRPC服务调用支持同步和异步方式，同时也支持普通的RPC和streaming模式，可以最大程度满足业务的需求。<br />
对于streaming模式，可以充分利用HTTP/2.0协议的多路复用功能，实现在一条HTTP链路上并行双向传输数据，有效的解决了HTTP/1.X的数据单向传输问题，在大幅减少HTTP连接的情况下，充分利用单条链路的性能，可以媲美传统的RPC私有长连接协议：更少的链路、更高的性能：

<img src="https://static001.geekbang.org/resource/image/af/b2/af32d0e627414f83e63137bfedfdc7b2.png" alt="" />

gRPC的网络I/O通信基于Netty构建，服务调用底层统一使用异步方式，同步调用是在异步的基础上做了上层封装。因此，gRPC的异步化是比较彻底的，对于提升I/O密集型业务的吞吐量和可靠性有很大的帮助。

源代码下载地址：

链接: [https://github.com/geektime-geekbang/gRPC_LLF/tree/master](https://github.com/geektime-geekbang/gRPC_LLF/tree/master)
