
# 1. RPC调用安全策略

## 1.1 严峻的安全形势

近年来，个人信息泄漏和各种信息安全事件层出不穷，个人信息安全以及隐私数据保护面临严峻的挑战。

很多国家已经通过立法的方式保护个人信息和数据安全，例如我国2016年11月7日出台、2017年6月1日正式实施的《网络安全法》，以及2016年4月14日欧盟通过的《一般数据保护法案》（GDP R），该法案将于2018年5月25日正式生效。

GDPR的通过意味着欧盟对个人信息保护及其监管达到了前所未有的高度，堪称史上最严格的数据保护法案。

作为企业内部各系统、模块之间调用的通信框架，即便是内网通信，RPC调用也需要考虑安全性，RPC调用安全主要涉及如下三点：

1. **个人/企业敏感数据加密：**例如针对个人的账号、密码、手机号等敏感信息进行加密传输，打印接口日志时需要做数据模糊化处理等，不能明文打印；
1. **对调用方的身份认证：**调用来源是否合法，是否有访问某个资源的权限，防止越权访问；
1. **数据防篡改和完整性：**通过对请求参数、消息头和消息体做签名，防止请求消息在传输过程中被非法篡改。

## 1.2 敏感数据加密传输

### 1.2.1 基于SSL/TLS的通道加密

当存在跨网络边界的RPC调用时，往往需要通过TLS/SSL对传输通道进行加密，以防止请求和响应消息中的敏感数据泄漏。跨网络边界调用场景主要有三种：

1. 后端微服务直接开放给端侧，例如手机App、TV、多屏等，没有统一的API Gateway/SLB做安全接入和认证；
1. 后端微服务直接开放给DMZ部署的管理或者运维类Portal；
1. 后端微服务直接开放给第三方合作伙伴/渠道。

除了跨网络之外，对于一些安全等级要求比较高的业务场景，即便是内网通信，只要跨主机/VM/容器通信，都强制要求对传输通道进行加密。在该场景下，即便只存在内网各模块的RPC调用，仍然需要做SSL/TLS。

使用SSL/TLS的典型场景如下所示：

<img src="https://static001.geekbang.org/resource/image/1a/48/1a69fa3b2939b510dc9af185e66e3d48.png" alt="" />

目前使用最广的SSL/TLS工具/类库就是OpenSSL，它是为网络通信提供安全及数据完整性的一种安全协议，囊括了主要的密码算法、常用的密钥和证书封装管理功能以及SSL协议。

多数SSL加密网站是用名为OpenSSL的开源软件包，由于这也是互联网应用最广泛的安全传输方法，被网银、在线支付、电商网站、门户网站、电子邮件等重要网站广泛使用。

### 1.2.2 针对敏感数据的单独加密

有些RPC调用并不涉及敏感数据的传输，或者敏感字段占比较低，为了最大程度的提升吞吐量，降低调用时延，通常会采用HTTP/TCP + 敏感字段单独加密的方式，既保障了敏感信息的传输安全，同时也降低了采用SSL/TLS加密通道带来的性能损耗，对于JDK原生的SSL类库，这种性能提升尤其明显。

它的工作原理如下所示：

<img src="https://static001.geekbang.org/resource/image/65/dd/6507b91b6cae0bbcaea4ec535d5944dd.png" alt="" />

通常使用Handler拦截机制，对请求和响应消息进行统一拦截，根据注解或者加解密标识对敏感字段进行加解密，这样可以避免侵入业务。

采用该方案的缺点主要有两个：

1. 对敏感信息的识别可能存在偏差，容易遗漏或者过度保护，需要解读数据和隐私保护方面的法律法规，而且不同国家对敏感数据的定义也不同，这会为识别带来很多困难；
1. 接口升级时容易遗漏，例如开发新增字段，忘记识别是否为敏感数据。

## 1.3 认证和鉴权

RPC的认证和鉴权机制主要包含两点：

1. **认证：**对调用方身份进行识别，防止非法调用；
1. **鉴权：**对调用方的权限进行校验，防止越权调用。

事实上，并非所有的RPC调用都必须要做认证和鉴权，例如通过API Gateway网关接入的流量，已经在网关侧做了鉴权和身份认证，对来自网关的流量RPC服务端就不需要重复鉴权。

另外，一些对安全性要求不太高的场景，可以只做认证而不做细粒度的鉴权。

### 1.3.1 身份认证

内部RPC调用的身份认证场景，主要有如下两大类：

1. 防止对方知道服务提供者的地址之后，绕过注册中心/服务路由策略直接访问RPC服务提供端；
1. RPC服务只想供内部模块调用，不想开放给其它业务系统使用（双方网络是互通的）。

身份认证的方式比较多，例如HTTP Basic Authentication、OAuth2等，比较简单使用的是令牌认证（Token）机制，它的工作原理如下所示：

<img src="https://static001.geekbang.org/resource/image/e0/b6/e0a4b9ecb2788ca842f2d41029a6adb6.png" alt="" />

工作原理如下：

1. RPC客户端和服务端通过HTTPS与注册中心连接，做双向认证，以保证客户端和服务端与注册中心之间的安全；
1. 服务端生成Token并注册到注册中心，由注册中心下发给订阅者。通过订阅/发布机制，向RPC客户端做Token授权；
1. 服务端开启身份认证，对RPC调用进行Token校验，认证通过之后才允许调用后端服务接口。

### 1.3.2 权限管控

身份认证可以防止非法调用，如果需要对调用方进行更细粒度的权限管控，则需要做对RPC调用做鉴权。例如管理员可以查看、修改和删除某个后台资源，而普通用户只能查看资源，不能对资源做管理操作。

在RPC调用领域比较流行的是基于OAuth2.0的权限认证机制，它的工作原理如下：

<img src="https://static001.geekbang.org/resource/image/eb/af/ebb67f6d52854e4e6771ebac4a9041af.png" alt="" />

OAuth2.0的认证流程如下：

1. 客户端向资源拥有者申请授权（例如携带用户名+密码等证明身份信息的凭证）；
1. 资源拥有者对客户端身份进行校验，通过之后同意授权；
1. 客户端使用步骤2的授权凭证，向认证服务器申请资源访问令牌（access token）；
1. 认证服务器对授权凭证进行合法性校验，通过之后，颁发access token；
1. 客户端携带access token（通常在HTTP Header中）访问后端资源，例如发起RPC调用；
1. 服务端对access token合法性进行校验（是否合法、是否过期等），同时对token进行解析，获取客户端的身份信息以及对应的资源访问权限列表，实现对资源访问权限的细粒度管控；
1. access token校验通过，返回资源信息给客户端。

步骤2的用户授权，有四种方式：

1. 授权码模式（authorization code）
1. 简化模式（implicit）
1. 密码模式（resource owner password credentials）
1. 客户端模式（client credentials）

需要指出的是，OAuth 2.0是一个规范，不同厂商即便遵循该规范，实现也可能会存在细微的差异。大部分厂商在采用OAuth 2.0的基础之上，往往会衍生出自己特有的OAuth 2.0实现。

对于access token，为了提升性能，RPC服务端往往会缓存，不需要每次调用都与AS服务器做交互。同时，access token是有过期时间的，根据业务的差异，过期时间也会不同。客户端在token过期之前，需要刷新Token，或者申请一个新的Token。

考虑到access token的安全，通常选择SSL/TLS加密传输，或者对access token单独做加密，防止access token泄漏。

## 1.4 数据完整性和一致性

RPC调用，除了数据的机密性和有效性之外，还有数据的完整性和一致性需要保证，即如何保证接收方收到的数据与发送方发出的数据是完全相同的。

利用消息摘要可以保障数据的完整性和一致性，它的特点如下：

- 单向Hash算法，从明文到密文的不可逆过程，即只能加密而不能解密；
- 无论消息大小，经过消息摘要算法加密之后得到的密文长度都是固定的；
- 输入相同，则输出一定相同。

目前常用的消息摘要算法是SHA-1、MD5和MAC，MD5可产生一个128位的散列值。 SHA-1则是以MD5为原型设计的安全散列算法，可产生一个160位的散列值，安全性更高一些。MAC除了能够保证消息的完整性，还能够保证来源的真实性。

由于MD5已被发现有许多漏洞，在实际应用中更多使用SHA和MAC，而且往往会把数字签名和消息摘要混合起来使用。

# gRPC安全机制

谷歌提供了可扩展的安全认证机制，以满足不同业务场景需求，它提供的授权机制主要有四类：

1. **通道凭证：**默认提供了基于HTTP/2的TLS，对客户端和服务端交换的所有数据进行加密传输；
1. **调用凭证：**被附加在每次RPC调用上，通过Credentials将认证信息附加到消息头中，由服务端做授权认证；
1. **组合凭证：**将一个频道凭证和一个调用凭证关联起来创建一个新的频道凭证，在这个频道上的每次调用会发送组合的调用凭证来作为授权数据，最典型的场景就是使用HTTP S来传输Access Token；
1. **Google的OAuth 2.0：**gRPC内置的谷歌的OAuth 2.0认证机制，通过gRPC访问Google API 时，使用Service Accounts密钥作为凭证获取授权令牌。

## 2.1 SSL/TLS认证

gRPC基于HTTP/2协议，默认会开启SSL/TLS，考虑到兼容性和适用范围，gRPC提供了三种协商机制：

- **PlaintextNegotiator：**非SSL/TLS加密传输的HTTP/2通道，不支持客户端通过HTTP/1.1的Upgrade升级到HTTP/2,代码示例如下（PlaintextNegotiator类）：

```
static final class PlaintextNegotiator implements ProtocolNegotiator {
   @Override
   public Handler newHandler(GrpcHttp2ConnectionHandler handler) {
     return new BufferUntilChannelActiveHandler(handler);
   }
 }

```

- **PlaintextUpgradeNegotiator：**非SSL/TLS加密传输的HTTP/2通道，支持客户端通过HTTP/1.1的Upgrade升级到HTTP/2，代码示例如下（PlaintextUpgradeNegotiator类）：

```
static final class PlaintextUpgradeNegotiator implements ProtocolNegotiator {
   @Override
   public Handler newHandler(GrpcHttp2ConnectionHandler handler) {
          Http2ClientUpgradeCodec upgradeCodec = new Http2ClientUpgradeCodec(handler);
     HttpClientCodec httpClientCodec = new HttpClientCodec();
     final HttpClientUpgradeHandler upgrader =
         new HttpClientUpgradeHandler(httpClientCodec, upgradeCodec, 1000);
     return new BufferingHttp2UpgradeHandler(upgrader);
   }
 }

```

- **TlsNegotiator：**基于SSL/TLS加密传输的HTTP/2通道，代码示例如下（TlsNegotiator类）：

```
static final class TlsNegotiator implements ProtocolNegotiator {
   private final SslContext sslContext;
   private final String host;
   private final int port;
   TlsNegotiator(SslContext sslContext, String host, int port) {
     this.sslContext = checkNotNull(sslContext, &quot;sslContext&quot;);
     this.host = checkNotNull(host, &quot;host&quot;);
     this.port = port;
   }

```

下面对gRPC的SSL/TLS工作原理进行详解。

### 2.1.1 SSL/TLS工作原理

SSL/TLS分为单向认证和双向认证，在实际业务中，单向认证使用较多，即客户端认证服务端，服务端不认证客户端。

SSL单向认证的过程原理如下：

1. SL客户端向服务端传送客户端SSL协议的版本号、支持的加密算法种类、产生的随机数，以及其它可选信息；
1. 服务端返回握手应答，向客户端传送确认SSL协议的版本号、加密算法的种类、随机数以及其它相关信息；
1. 服务端向客户端发送自己的公钥；
1. 客户端对服务端的证书进行认证，服务端的合法性校验包括：证书是否过期、发行服务器证书的CA是否可靠、发行者证书的公钥能否正确解开服务器证书的“发行者的数字签名”、服务器证书上的域名是否和服务器的实际域名相匹配等；
1. 客户端随机产生一个用于后面通讯的“对称密码”，然后用服务端的公钥对其加密，将加密后的“预主密码”传给服务端；
1. 服务端将用自己的私钥解开加密的“预主密码”，然后执行一系列步骤来产生主密码；
1. 客户端向服务端发出信息，指明后面的数据通讯将使用主密码为对称密钥，同时通知服务器客户端的握手过程结束；
1. 服务端向客户端发出信息，指明后面的数据通讯将使用主密码为对称密钥，同时通知客户端服务器端的握手过程结束；
1. SSL的握手部分结束，SSL安全通道建立，客户端和服务端开始使用相同的对称密钥对数据进行加密，然后通过Socket进行传输。

SSL单向认证的流程图如下所示：

<img src="https://static001.geekbang.org/resource/image/67/ee/672ce5cc60be10d880553ff883c953ee.png" alt="" />

SSL双向认证相比单向认证，多了一步服务端发送认证请求消息给客户端，客户端发送自签名证书给服务端进行安全认证的过程。

客户端接收到服务端要求客户端认证的请求消息之后，发送自己的证书信息给服务端，信息如下：

<img src="https://static001.geekbang.org/resource/image/b2/bf/b20f662117c0211a7b963fb80f9ac9bf.png" alt="" />

服务端对客户端的自签名证书进行认证，信息如下：

<img src="https://static001.geekbang.org/resource/image/8f/c1/8f00556a8c2309add279f53a60207dc1.png" alt="" />

### 2.1.2 HTTP/2的ALPN

对于一些新的web协议，例如HTTP/2，客户端和浏览器需要知道服务端是否支持HTTP/2,对于HTTP/2 Over HTTP可以使用HTTP/1.1的Upgrade机制进行协商，对于HTTP/2 Over TLS，则需要使用到NPN或ALPN扩展来完成协商。

ALPN作为HTTP/2 Over TLS的协商机制，已经被定义到 RFC7301中，从2016年开始它已经取代NPN成为HTTP/2Over TLS的标准协商机制。目前所有支持HTTP/2的浏览器都已经支持ALPN。

Jetty为 OpenJDK 7和OpenJDK 8提供了扩展的ALPN实现（JDK默认不支持），ALPN类库与Jetty容器本身并不强绑定，无论是否使用Jetty作为Web容器，都可以集成Jetty提供的ALPN类库，以实现基于TLS的HTTP/2协议。

如果要开启ALPN，需要增加如下JVM启动参数：

```
java -Xbootclasspath/p:&lt;path_to_alpn_boot_jar&gt; ...

```

客户端代码示例如下：

```
SSLContext sslContext = ...;
final SSLSocket sslSocket = (SSLSocket)context.getSocketFactory().createSocket(&quot;localhost&quot;, server.getLocalPort());
ALPN.put(sslSocket, new ALPN.ClientProvider()
{
    public boolean supports()
    {
        return true;
    }
    public List&lt;String&gt; protocols()
    {
        return Arrays.asList(&quot;h2&quot;, &quot;http/1.1&quot;);
    }
    public void unsupported()
    {
        ALPN.remove(sslSocket);
    }
    public void selected(String protocol)
    {
        ALPN.remove(sslSocket);
          }
});

```

服务端代码示例如下：

```
final SSLSocket sslSocket = ...;
ALPN.put(sslSocket, new ALPN.ServerProvider()
{
    public void unsupported()
    {
        ALPN.remove(sslSocket);
    }
    public String select(List&lt;String&gt; protocols);
    {
        ALPN.remove(sslSocket);
        return protocols.get(0);
    }
});

```

以上代码示例来源：[http://www.eclipse.org/jetty/documentation/9.3.x/alpn-chapter.html](http://www.eclipse.org/jetty/documentation/9.3.x/alpn-chapter.html)

需要指出的是，Jetty ALPN类库版本与JDK版本是配套使用的，配套关系如下所示：

可以通过如下网站查询双方的配套关系：[http://www.eclipse.org/jetty/documentation/9.3.x/alpn-chapter.html](http://www.eclipse.org/jetty/documentation/9.3.x/alpn-chapter.html)

如果大家需要了解更多的Jetty ALPN相关信息，可以下载jetty的ALPN源码和文档学习。

### 2.1.3 gRPC 的TLS策略

gRPC的TLS实现有两种策略：

1. 基于OpenSSL的TLS
1. 基于Jetty ALPN/NPN的TLS

对于非安卓的后端Java应用，gRPC强烈推荐使用OpenSSL，原因如下：

1. 性能更高：基于OpenSSL的gRPC调用比使用JDK GCM的性能高10倍以上；
1. 密码算法更丰富：OpenSSL支持的密码算法比JDK SSL提供的更丰富，特别是HTTP/2协议使用的加密算法；
1. OpenSSL支持ALPN回退到NPN；
1. 不需要根据JDK的版本升级配套升级ALPN类库（Jetty的ALPN版本与JDK特定版本配套使用）。

gRPC的HTTP/2和TLS基于Netty框架实现，如果使用OpenSSL，则需要依赖Netty的netty-tcnative。

Netty的OpenSSL有两种实现机制：Dynamic linked和Statically Linked。在开发和测试环境中，建议使用Statically Linked的方式（netty-tcnative-boringssl-static），它提供了对ALPN的支持以及HTTP/2需要的密码算法，不需要额外再集成Jetty的ALPN类库。从1.1.33.Fork16版本开始支持所有的操作系统，可以实现跨平台运行。

对于生产环境，则建议使用Dynamic linked的方式，原因如下：

1. 很多场景下需要升级OpenSSL的版本或者打安全补丁，如果使用动态链接方式（例如apt-ge），则应用软件不需要级联升级；
1. 对于一些紧急的OpenSSL安全补丁，如果采用Statically Linked的方式，需要等待Netty社区提供新的静态编译补丁版本，可能会存在一定的滞后性。

netty-tcnative-boringssl-static的Maven配置如下：

```
&lt;project&gt;
  &lt;dependencies&gt;
    &lt;dependency&gt;
      &lt;groupId&gt;io.netty&lt;/groupId&gt;
      &lt;artifactId&gt;netty-tcnative-boringssl-static&lt;/artifactId&gt;
      &lt;version&gt;2.0.6.Final&lt;/version&gt;
    &lt;/dependency&gt;
  &lt;/dependencies&gt;
&lt;/project&gt;

```

使用Dynamically Linked (netty-tcnative)的相关约束如下：

1. 

```
OpenSSL version &gt;= 1.0.2 for ALPN

```

或者

```
version &gt;= 1.0.1 for NPN

```

1. 类路径中包含

```
netty-tcnative version &gt;= 1.1.33.Fork7

```

尽管gRPC强烈不建议使用基于JDK的TLS，但是它还是提供了对Jetty ALPN/NPN的支持。

通过Xbootclasspath参数开启ALPN，示例如下：

```
java -Xbootclasspath/p:/path/to/jetty/alpn/extension.jar

```

由于ALPN类库与JDK版本号有强对应关系，如果匹配错误，则会导致SSL握手失败，因此可以通过 Jetty-ALPN-Agent来自动为JDK版本选择合适的ALPN版本，启动参数如下所示：

```
java -javaagent:/path/to/jetty-alpn-agent.jar

```

### 2.1.4 基于TLS的gRPC代码示例

以基于JDK（Jetty-ALPN）的TLS为例，给出gRPC SSL安全认证的代码示例。

TLS服务端创建：

```
int port = 18443;
    SelfSignedCertificate ssc = new SelfSignedCertificate();
    server = ServerBuilder.forPort(port).useTransportSecurity(ssc.certificate(),
            ssc.privateKey())
        .addService(new GreeterImpl())
        .build()
        .start();

```

其中SelfSignedCertificate是Netty提供的用于测试的临时自签名证书类，在实际项目中，需要加载生成环境的CA和密钥。<br />
在启动参数中增加SSL握手日志打印以及Jetty的ALPN Agent类库，示例如下：

<img src="https://static001.geekbang.org/resource/image/0c/91/0cd945f90415049a211ddde507a54191.png" alt="" />

启动服务端，显示SSL证书已经成功加载：

<img src="https://static001.geekbang.org/resource/image/4e/20/4ea36b4f882197e496a2ffea7ecddf20.png" alt="" />

TLS客户端代码创建：

```
this(NettyChannelBuilder.forAddress(host, port).sslContext(
            GrpcSslContexts.forClient().
            ciphers(Http2SecurityUtil.CIPHERS,
                    SupportedCipherSuiteFilter.INSTANCE).
            trustManager(InsecureTrustManagerFactory.INSTANCE).build()));

```

NettyChannel创建时，使用gRPC的GrpcSslContexts指定客户端模式，设置HTTP/2的密钥，同时加载CA证书工厂，完成TLS客户端的初始化。

与服务端类似，需要通过-javaagent指定ALPN Agent类库路径，同时开启SSL握手调试日志打印，启动客户端，运行结果如下所示：

<img src="https://static001.geekbang.org/resource/image/77/7e/7789c382a2e32b77c9a8f201f0d0057e.png" alt="" />

### 2.1.5 gRPC TLS源码分析

gRPC在Netty SSL类库基础上做了二次封装，以简化业务的使用，以服务端代码为例进行说明，服务端开启TLS，代码如下（NettyServerBuilder类）：

```
public NettyServerBuilder useTransportSecurity(File certChain, File privateKey) {
    try {
      sslContext = GrpcSslContexts.forServer(certChain, privateKey).build();

```

实际调用GrpcSslContexts创建了Netty SslContext，下面一起分析下GrpcSslContexts的实现，它调用了Netty SslContextBuilder，加载X.509 certificate chain file和PKCS#8 private key file（PEM格式），代码如下（SslContextBuilder类）：

```
public static SslContextBuilder forServer(File keyCertChainFile, File keyFile) {
        return new SslContextBuilder(true).keyManager(keyCertChainFile, keyFile);
    }

```

Netty的SslContext加载keyCertChainFile和private key file（SslContextBuilder类）：

```
X509Certificate[] keyCertChain;
        PrivateKey key;
        try {
            keyCertChain = SslContext.toX509Certificates(keyCertChainFile);
        } catch (Exception e) {
            throw new IllegalArgumentException(&quot;File does not contain valid certificates: &quot; + keyCertChainFile, e);
        }
        try {
            key = SslContext.toPrivateKey(keyFile, keyPassword);

```

加载完成之后，通过SslContextBuilder创建SslContext，完成SSL上下文的创建。

服务端开启SSL之后，gRPC会根据初始化完成的SslContext创建SSLEngine，然后实例化Netty的SslHandler，将其加入到ChannelPipeline中，代码示例如下（ServerTlsHandler类）：

```
public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
      super.handlerAdded(ctx);
      SSLEngine sslEngine = sslContext.newEngine(ctx.alloc());
      ctx.pipeline().addFirst(new SslHandler(sslEngine, false));
    }

```

下面一起分析下Netty SSL服务端的源码，SSL服务端接收客户端握手请求消息的入口方法是decode方法，首先获取接收缓冲区的读写索引，并对读取的偏移量指针进行备份（SslHandler类）：

```
protected void decode(ChannelHandlerContext ctx, ByteBuf in, List&lt;Object&gt; out) throws SSLException {
        final int startOffset = in.readerIndex();
        final int endOffset = in.writerIndex();
        int offset = startOffset;
        int totalLength = 0;
...

```

对半包标识进行判断，如果上一个消息是半包消息，则判断当前可读的字节数是否小于整包消息的长度，如果小于整包长度，则说明本次读取操作仍然没有把SSL整包消息读取完整，需要返回I/O线程继续读取，代码如下：

```
if (packetLength &gt; 0) {
            if (endOffset - startOffset &lt; packetLength) {
                return;
...

```

如果消息读取完整，则修改偏移量：同时置位半包长度标识：

```
} else {
                offset += packetLength;
                totalLength = packetLength;
                packetLength = 0;
            }

```

下面在for循环中读取SSL消息，一个ByteBuf可能包含多条完整的SSL消息。首先判断可读的字节数是否小于协议消息头长度，如果是则退出循环继续由I/O线程接收后续的报文：

```
if (readableBytes &lt; SslUtils.SSL_RECORD_HEADER_LENGTH) {
                break;
            }

```

获取SSL消息包的报文长度，具体算法不再介绍，可以参考SSL的规范文档进行解读，代码如下（SslUtils类）：

```
 if (tls) {
            // SSLv3 or TLS - Check ProtocolVersion
            int majorVersion = buffer.getUnsignedByte(offset + 1);
            if (majorVersion == 3) {
                // SSLv3 or TLS
                packetLength = buffer.getUnsignedShort(offset + 3) + SSL_RECORD_HEADER_LENGTH;
...

```

对长度进行判断，如果SSL报文长度大于可读的字节数，说明是个半包消息，将半包标识长度置位，返回I/O线程继续读取后续的数据报，代码如下（SslHandler类）：

```
 if (packetLength &gt; readableBytes) {
                // wait until the whole packet can be read
                this.packetLength = packetLength;
                break;
            }

```

对消息进行解码，将SSL加密的消息解码为加密前的原始数据，unwrap方法如下：

```
private boolean unwrap(
            ChannelHandlerContext ctx, ByteBuf packet, int offset, int length) throws SSLException {

        boolean decoded = false;
        boolean wrapLater = false;
        boolean notifyClosure = false;
        ByteBuf decodeOut = allocate(ctx, length);
        try {
            while (!ctx.isRemoved()) {
                final SSLEngineResult result = engineType.unwrap(this, packet, offset, length, decodeOut);
                final Status status = result.getStatus();
...

```

调用SSLEngine的unwrap方法对SSL原始消息进行解码，对解码结果进行判断，如果越界，说明out缓冲区不够，需要进行动态扩展。如果是首次越界，为了尽量节约内存，使用SSL最大缓冲区长度和SSL原始缓冲区可读的字节数中较小的。如果再次发生缓冲区越界，说明扩张后的缓冲区仍然不够用，直接使用SSL缓冲区的最大长度，保证下次解码成功。

解码成功之后，对SSL引擎的操作结果进行判断：如果需要继续接收数据，则继续执行解码操作；如果需要发送握手消息，则调用wrapNonAppData发送握手消息；如果需要异步执行SSL代理任务，则调用立即执行线程池执行代理任务；如果是握手成功，则设置SSL操作结果，发送SSL握手成功事件；如果是应用层的业务数据，则继续执行解码操作，其它操作结果，抛出操作类型异常（SslHandler类）：

```
switch (handshakeStatus) {
                    case NEED_UNWRAP:
                        break;
                    case NEED_WRAP:
                        wrapNonAppData(ctx, true);
                        break;
                    case NEED_TASK:
                        runDelegatedTasks();
                        break;
                    case FINISHED:
                        setHandshakeSuccess();
                        wrapLater = true;
...

```

需要指出的是，SSL客户端和服务端接收对方SSL握手消息的代码是相同的，那为什么SSL服务端和客户端发送的握手消息不同呢？这些是SSL引擎负责区分和处理的，我们在创建SSL引擎的时候设置了客户端模式，SSL引擎就是根据这个来进行区分的。

SSL的消息读取实际就是ByteToMessageDecoder将接收到的SSL加密后的报文解码为原始报文，然后将整包消息投递给后续的消息解码器，对消息做二次解码。基于SSL的消息解码模型如下：

<img src="https://static001.geekbang.org/resource/image/c9/4e/c9267e5c82a9e08f7df3cb39e286844e.png" alt="" />

SSL消息读取的入口都是decode，因为是非握手消息，它的处理非常简单，就是循环调用引擎的unwrap方法，将SSL报文解码为原始的报文，代码如下（SslHandler类）：

```
switch (status) {
                case BUFFER_OVERFLOW:
                    int readableBytes = decodeOut.readableBytes();
                    int bufferSize = engine.getSession().getApplicationBufferSize() - readableBytes;
                    if (readableBytes &gt; 0) {
                        decoded = true;
                        ctx.fireChannelRead(decodeOut);
...

```

握手成功之后的所有消息都是应用数据，因此它的操作结果为NOT_HANDSHAKING，遇到此标识之后继续读取消息，直到没有可读的字节，退出循环。

如果读取到了可用的字节，则将读取到的缓冲区加到输出结果列表中，有后续的Handler进行处理，例如对HTTPS的请求报文做反序列化。

SSL消息发送时，由SslHandler对消息进行编码，编码后的消息实际就是SSL加密后的消息。从待加密的消息队列中弹出消息，调用SSL引擎的wrap方法进行编码，代码如下（SslHandler类）：

```
 while (!ctx.isRemoved()) {
                Object msg = pendingUnencryptedWrites.current();
                if (msg == null) {
                    break;
                }
                ByteBuf buf = (ByteBuf) msg;
                if (out == null) {
                    out = allocateOutNetBuf(ctx, buf.readableBytes());
                }
                SSLEngineResult result = wrap(alloc, engine, buf, out);

```

wrap方法很简单，就是调用SSL引擎的编码方法，然后对写索引进行修改，如果缓冲区越界，则动态扩展缓冲区：

```
for (;;) {
                ByteBuffer out0 = out.nioBuffer(out.writerIndex(), out.writableBytes());
                SSLEngineResult result = engine.wrap(in0, out0);
                in.skipBytes(result.bytesConsumed());
                out.writerIndex(out.writerIndex() + result.bytesProduced());
...

```

对SSL操作结果进行判断，因为已经握手成功，因此返回的结果是NOT_HANDSHAKING，执行finishWrap方法，调用ChannelHandlerContext的write方法，将消息写入发送缓冲区中，如果待发送的消息为空，则构造空的ByteBuf写入（SslHandler类）：

```
private void finishWrap(ChannelHandlerContext ctx, ByteBuf out, ChannelPromise promise, boolean inUnwrap,
            boolean needUnwrap) {
        if (out == null) {
            out = Unpooled.EMPTY_BUFFER;
        } else if (!out.isReadable()) {
            out.release();
            out = Unpooled.EMPTY_BUFFER;
        }
        if (promise != null) {
            ctx.write(out, promise);
        } else {
            ctx.write(out);
        }

```

编码后，调用ChannelHandlerContext的flush方法消息发送给对方，完成消息的加密发送。

## 2.2 Google OAuth 2.0

### 2.2.1 工作原理

gRPC默认提供了多种OAuth 2.0认证机制，假如gRPC应用运行在GCE里，可以通过服务账号的密钥生成Token用于RPC调用的鉴权，密钥可以从环境变量 GOOGLE_APPLICATION_CREDENTIALS 对应的文件里加载。如果使用GCE，可以在虚拟机设置的时候为其配置一个默认的服务账号，运行是可以与认证系统交互并为Channel生成RPC调用时的access Token。

### 2.2.2. 代码示例

以OAuth2认证为例，客户端代码如下所示，创建OAuth2Credentials，并实现Token刷新接口：

<img src="https://static001.geekbang.org/resource/image/d7/a3/d7b020fecb7c9a28bb4f3696984753a3.png" alt="" />

创建Stub时，指定CallCredentials，代码示例如下（基于gRPC1.3版本，不同版本接口可能发生变化）：

```
GoogleAuthLibraryCallCredentials callCredentials =
            new GoogleAuthLibraryCallCredentials(credentials);
blockingStub = GreeterGrpc.newBlockingStub(channel)
.withCallCredentials(callCredentials);

```

下面的代码示例，用于在GCE环境中使用Google的OAuth2：

```
ManagedChannel channel = ManagedChannelBuilder.forTarget(&quot;pubsub.googleapis.com&quot;)
.build();
GoogleCredentials creds = GoogleCredentials.getApplicationDefault();
creds = creds.createScoped(Arrays.asList(&quot;https://www.googleapis.com/auth/pubsub&quot;));
CallCredentials callCreds = MoreCallCredentials.from(creds);
PublisherGrpc.PublisherBlockingStub publisherStub =
    PublisherGrpc.newBlockingStub(channel).withCallCredentials(callCreds);
publisherStub.publish(someMessage);

```

2.3. 自定义安全认证策略

参考Google内置的Credentials实现类，实现自定义的Credentials，可以扩展gRPC的鉴权策略，Credentials的实现类如下所示：

<img src="https://static001.geekbang.org/resource/image/bf/3c/bf248fabe09b968f020a7502748c093c.png" alt="" />

以OAuth2Credentials为例，实现getRequestMetadata(URI uri)方法，获取access token，将其放入Metadata中，通过CallCredentials将其添加到请求Header中发送到服务端，代码示例如下（GoogleAuthLibraryCallCredentials类）：

```
Map&lt;String, List&lt;String&gt;&gt; metadata = creds.getRequestMetadata(uri);
            Metadata headers;
            synchronized (GoogleAuthLibraryCallCredentials.this) {
              if (lastMetadata == null || lastMetadata != metadata) {
                lastMetadata = metadata;
                lastHeaders = toHeaders(metadata);
              }
              headers = lastHeaders;
            }
            applier.apply(headers);

```

对于扩展方需要自定义Credentials，实现getRequestMetadata(URI uri)方法，由gRPC的CallCredentials将鉴权信息加入到HTTP Header中发送到服务端。

源代码下载地址：

链接: [https://github.com/geektime-geekbang/gRPC_LLF/tree/master](https://github.com/geektime-geekbang/gRPC_LLF/tree/master)
