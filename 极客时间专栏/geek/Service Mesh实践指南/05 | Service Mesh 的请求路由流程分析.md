
## 内容概要

本文主要分析 WeiboMesh 在运行阶段请求路由的实现，对比现有的通用 Service Mesh （比如 Istio ）在这方面的不同。

前面我们对 WeiboMesh 的演化、实现以及 Service Mesh 规范等做了简要分析和解读，希望能帮助你基于 WeiboMesh 更好地理解 Service Mesh 的架构理念。

这里我们从实践的角度来体会下当前最受关注的 Service Mesh 实现 Istio 和 WeiboMesh 在服务调用和请求路由方面的一些异同，希望对你入坑 Service Mesh 有所帮助。

WeiboMesh 源自于 Weibo 内部对异构体系服务化的强烈需求以及对历史沉淀的取舍权衡，它没有把历史作为包袱而是巧妙地结合自身实际情况完成了对 Service Mesh 规范的实现，与同时期的其他 Service Mesh 实现相较而言，WeiboMesh 以最接地气的 Service Mesh 实现著称并不无道理。

如果你的团队拖着繁重的技术债务，又想尝试 Service Mesh 理念带来的诸多好处，那 WeiboMesh 应该能为你带来些许灵感。下面就让我们一起来体验一下。

## WeiboMesh 的服务注册与发现

由微博开源 Motan RPC 框架演化而来的 WeiboMesh 服务注册与发现功能扩展自 Motan 相关功能，机制和用法基本不变，流程同样是服务启动的时候将自己的分组、服务名称、版本等等注册到注册中心。

稍微有点区别的可能在于扩展了具体执行注册和发现操作的角色，WeiboMesh 的服务由 Server Agent 完成注册，而通过 Client Agent 完成对依赖服务的订阅与发现。Mesh 层接管了服务进出的流量，那请求又是如何在 WeiboMesh 中流转的呢？

## 请求在 WeiboMesh 中的流转

我们以 WeiboMesh 的示例项目（[github.com/motan-ecosystem/motan-examples）中各语言的](http://github.com/motan-ecosystem/motan-examples%EF%BC%89%E4%B8%AD%E5%90%84%E8%AF%AD%E8%A8%80%E7%9A%84) HelloWorldService 作为服务网格中的各种相互依赖的异构服务，基于 zookeeper 注册中心，看下一个请求是如何在 WeiboMesh 中流转的。假定我们有个 Test 业务（motan-examples/php/motan.php），它依赖由 OpenResty 提供的一个 HTTP 服务 （[http://10.211.55.3:9900/http_server）。](http://10.211.55.3:9900/http_server%EF%BC%89%E3%80%82)

<img src="https://static001.geekbang.org/resource/image/fb/3f/fb057dae851374fe4eb2ac04cc38423f.png" alt="" />

作为服务提供方，HTTP 服务一侧的 Server Agent 将 HTTP 接口导出为 WeiboMesh 集群中的 HelloWorldService，并将其注册到 zookeeper，而依赖此服务的 Test 业务一侧的 Client Agent 订阅了这个 HelloWorldService 服务作为自己的一个 referer，并将服务的节点实时发现回来，Test 业务只需要对本地的 Client Agent 发起调用即可完成对 HelloWorldService 的依赖，而由 Client Agent 与对端的 Server Agent 发起点对点通信，下面我们通过抓包来演示请求的流转。

<img src="https://static001.geekbang.org/resource/image/2d/56/2dc5607bc836a9a907bc2b389c790856.png" alt="" />

图中 z2.shared 为 Test 业务部署的节点，而所依赖的 HTTP 服务 [http://10.211.55.3:9900/http_server](http://10.211.55.3:9900/http_server) 部署于 z.shared 节点，可以看出 Test 直接请求本机的 9981 Mesh 端口（Client Agent），Client Agent 又对远端的 Mesh 端口（Server Agent：9990）发起直连调用。

下面是将 HTTP 服务导出为Motan RPC 服务，并暴露到 WeiboMesh 集群 的相关 Server Agent 配置，可以看到其将 HTTP 接口导出为 motan2 协议 的 Motan RPC 服务，并在 9990 端口提供服务。

```
  http-mesh-example-helloworld:
	path: com.weibo.motan.HelloWorldService
	export: &quot;motan2:9990&quot;
	provider: http
	registry: &quot;zk-registry&quot; # registry id
	HTTP_URL: http://10.211.55.3:9900/http_server
	basicRefer: mesh-server-basicService

```

WeiboMesh 又是如何实现控制面板来对请求进行精细控制的呢？

目前主要由指令管理系统（motan-manager，提供包括 RPC 服务查询、流量切换、Motan 指令设置等功能）、决策系统（提供遥测数据搜集、自动化扩缩容并配合指令系统实现流量弹性调度等功能。

这部分为微博内部功能，目前暂无开源实现）还有各种过滤器（请求验证等功能推荐通过 Motan 过滤器机制来扩展实现）等模块来完成各种策略执行、验证、流量管理等功能。以流量切换为例，我们只需要执行如下指令就可以完成对流量的比例切换，由此可见 WeiboMesh 的使用成本是极低的。

```
{
    &quot;clientCommandList&quot; : [
        {
            &quot;index&quot;: 1,
            &quot;version&quot;: &quot;1.0&quot;,
            &quot;dc&quot;: &quot;yf&quot;,
            &quot;commandType&quot;: 0,
            &quot;pattern&quot;: &quot;\*&quot;,
            &quot;mergeGroups&quot;: [
                &quot;openapi-tc-test-rpc:1&quot;,
                &quot;openapi-yf-test-rpc:1&quot;
            ],
            &quot;routeRules&quot;: [],
            &quot;remark&quot;: &quot;切换50%流量到另外一个机房&quot;
        }
    ]
}

```

我们再来对比看看其他有代表性的 Service Mesh 实现是如何复杂的，比如 Istio。

众所周知 Istio 在 Service Mesh 领域可谓是一枝独秀，出身名门的光环使大家对它期待有加，虽然从 2017 年 5 月 24 日Google、IBM 联合 Lyft 公司共同宣布 Istio 项目以来，Istio 尚未给出一个可以生产验证的版本，但是这并不影响社区对它的强烈兴趣。

不过 Istio、Conduit 等项目的出现是对 Service Mesh 理念的认可，也进一步促成了数据面板和控制面板作为 Service Mesh 主体功能这一事实规范的形成。

Istio 借助一个 SideCar 模式的 proxy 作为数据面板接管了容器中业务进出流量，而又实现了控制面板来对 proxy 进行易于扩展的细粒度控制，完成应用流量的切分、访问授权、遥测、策略配置等一系列服务治理功能。

下面我们就根据 Istio 的示例项目 bookinfo 来分析一个请求在 Istio 内是如何流转的（Kubernetes 平台作为 Istio 默认使用的平台也是支持最完整的平台，所以本文下面的分析都基于 Kubernetes 平台进行，且 Kubernetes 为基于 kubeadm 在裸机部署）。

## Istio 服务的注册与发现

我们知道 Istio 假定存在服务注册表，以跟踪应用程序中服务的 Pod/VM，它还假设服务的新实例自动注册到服务注册表，并且不健康的实例将被自动删除。

诸如 Kubernetes，Mesos 等平台已经为基于容器的应用程序提供了这样的功能。也就是说，Istio 本身并不直接提供对服务的注册和发现功能，它依赖于部署的平台，那它又是如何依赖平台来实现的呢？

我们先来看看原生的 Kubernetes 平台是如何实现服务注册和发现的。

当我们执行 kubectl 命令部署服务时，Kubernetes 会根据 Pod 调度机制在相应的节点上部署相关的 Pod，并根据 Service 的部署规范描述部署对应的服务，同时新增部署的服务状态会通过 Kubernetes 的 kube-api-server 注册到中心的 Etcd 集群，又由 Kubernetes 的 List-Watch 机制将相关变更同步到各模块，实现机制是典型的发布订阅模型。

其默认的服务发现组件 kube-dns 中的 kube2sky 模块则通过 watch kube-api-server 来对 Kubernetes 集群中的 Service 和 Endpoint 数据进行更新和维护。请求调用时，对应的 DNS 请求被 kube-dns 中的 dnsmasq 接收，走 DNS 解析机制完成服务发现逻辑。

再看 Istio 的情况，Istio 服务部署前需要执行 Istioctl kube-inject 在 Pod 中注入 Istio 相关支持，这里一个关键的目的就在于以 SideCar 模式注入 Envoy 代理，而这其中有一步基础的初始化工作就是在 SideCar 容器 Istio-proxy 中添加相关的 iptables 规则，将所有服务相关的进出流量都拦截到 Envoy 的处理端口，这是 Istio 服务注册和发现的关键一步。

Istio 服务部署过程中除了依赖 Kubernetes 平台将服务注册到注册中心，并将状态同步给相关组件，还将 Istio 集群所有的流量都归一到 Envoy 代理处理，由 Envoy 去请求所依赖的服务，代理过程中 Envoy 通过 Pilot 组件的各种服务发现接口来完成服务发现的逻辑。

而 Pilot 正是 Istio 对各平台的服务注册和发现功能做平台无关抽象的核心组件。服务网格中的 Envoy 实例依赖 Pilot 执行服务发现，并相应地动态更新其负载均衡池。

Istio 依赖平台完成服务注册及状态变更，请求过程中服务的发现依赖于 Pilot，这是 Pilot 主要职能之一，这部分的实现是为了平台透明化而做的解耦，我的理解是一方面出于开源通用性方面的考虑，一方面也来自于大厂对云原生市场前景的看好与抢占吧。

而 WeiboMesh 作为微博生产的实现，目前并没有对那么多平台适配的需求，我们只需直接扩展对各个注册中心的支持即可。好处是更简洁，直接。而且现实场景下，除非是新启动的项目，否则很难完全做到云原生，不过大家都在往这个方向走是确定的，所以我们也计划后期对各种平台做支持，希望大家共同期待、共同参与进来。

## 请求在 Istio 集群中的流转

仍然以裸机 Kubernetes 平台部署 Istio 为例，我们在此前基础上部署 Istio 测试服务 bookinfo。下面是相关的部署情况（节点概况、Pod 分布以及服务部署）：

<img src="https://static001.geekbang.org/resource/image/e4/d1/e4e7a25cf6f837cc0cfe8244c1bdecd1.png" alt="" />

因为我们是裸机部署，没有 LoadBalance 的支持，所以我们根据 Istio-ingress 的设置，可以使用 NodeIP + NodePort 的方式（Pod Istio-ingress-6ff959f7ff-8cz8x 所在节点为 kube-node2，ip 为 10.211.55.76，而 Service Istio-ingress 80 端口服务对应的 NodePort 为 32370）来访问测试服务 productpage [http://10.211.55.76:32370/productpage](http://10.211.55.76:32370/productpage) 下面我们看看这个请求在 Istio 集群中是如何流转的。

通过 Kubernetes ingress 机制，我们得以从集群外访问 Istio 集群的服务。在任意一个节点我们可以查看到 32370 端口相关的 iptables 规则，可知访问这个端口的流量最终会指向 10.244.1.158:80， 而 10.244.1.158 是 Istio-ingress 所在的 Pod 节点，具体的实现细节属于 kube-proxy 模块的内容，感兴趣的同学可以详细了解一下。

<img src="https://static001.geekbang.org/resource/image/f9/43/f9526edd56af81efacee59c279c08e43.png" alt="" />

而请求到了 Istio-ingress Pod （10.244.1.158 对应的 Pod 是 Istio-ingress-6ff959f7ff-8cz8x）后，我们进入这个 Pod  的容器可以查看到里面运行的 Pilot 和 Envoy 代理。

ingress  Pod 中的 Envoy 代理与 Service 中的 Envoy 代理角色不同，这里 Envoy 的角色是 ingress-controller，所以可以看到 Envoy 监听了 80 端口来直接接受请求（80 端口接受外部请求转发到集群内，而 15000 端口与其他 Service SideCar 场景一样，作为管理端口）。

<img src="https://static001.geekbang.org/resource/image/41/5b/41173645767808153e62ad95e4c05b5b.png" alt="" />

Envoy 通过与 Pilot 的服务发现服务持续通信维持了一个实时可用的 Cluster，如下图：

<img src="https://static001.geekbang.org/resource/image/60/aa/60d45296c714c34c9e44cf8503a514aa.png" alt="" />

Envoy 通过与 Pilot 提供的各种发现服务通信，比如通过 RDS（rds/v1/routes/80） 获取 80 端口正真提供服务的集群，通过 CDS（cds/v1/clusters/istio-proxy/ingress~~istio-ingress-6ff959f7ff-8cz8x.istio-system~istio-system.svc.cluster.local） 获取该集群提供服务的服务名，通过 SDS（sds/v1/registration/productpage.default.svc.cluster.local）获取服务对应的 Endpoint，完成服务定向（10.244.1.162:9080，我们发现 10.244.1.162 正好是 productpage 服务所在的 Pod productpage-v1-d759956f4-xmhm5 节点 IP，而 9080 端口正好是 productpage 服务提供的端口）。

当然在请求真正从 Istio-ingress 节点转发到 productpage Pod 的时候，并没有直接提供服务，入口流量打到 productpage 的 Envoy 代理，Envoy 将请求数据首先发往 Mixer 组件中的 Check &amp; report 服务进行相关的验证和数据上报，通过后请求才会真正的被处理。

<img src="https://static001.geekbang.org/resource/image/37/d1/37ebdd873fcadac9ff06da9e76af3ad1.png" alt="" />

图中 Istio-ingress（10.244.1.158）到 productpage（10.244.1.162）的请求，先是发往 Mixer 的 check &amp; report 服务（10.105.94.107:9091），才由本 Pod 的9080（真正的 productpage 服务）处理。

<img src="https://static001.geekbang.org/resource/image/59/cb/59f491da1e77c360b5f2055d91ea6bcb.png" alt="" />

现在请求已经到了基于 Python 开发的 productpage 服务，而这个服务依赖于 reviews、details 等服务，这部分请求又是如何流转的呢？我们这里以 productpage 对 details 服务的依赖为例进行说明。

productpage 和 detail 都是 Istio 网格中的服务，他们各自的 Pod 都以 SideCar 模式部署了 Envoy，所有的进出流量都经过 Envoy  代理，productpage 通过服务名称 [http://details:9080/details/0](http://details:9080/details/0) 访问 detail 服务，detail 服务的发现是基于 kube-DNS 解析完成的。我们可以通过 nslookup 命令来复原这一点。

<img src="https://static001.geekbang.org/resource/image/2e/d1/2e0b35ebb5d4a031892a2a4504fea8d1.png" alt="" />

而 productpage 对 detail 服务的调用地址明确指出依赖 detail 服务的 9080 端口。

<img src="https://static001.geekbang.org/resource/image/74/56/7466a326fdb9802d2d249f317da9e156.png" alt="" />

所以 productpage 最终依赖的 detail 服务地址为 10.103.96.223:9080，而往出的流量被 envoy 接管，所以对 detail 服务的调用由 envoy 代理发出并完成。从这些请求的分析大家不难窥探出 Istio 集群中服务间的依赖体系以及数据面板控制面板在这个过程中起到的关键作用。

我们这里从请求流转的角度分析了一个请求在 Istio 和 WeiboMesh 中是如何处理的，不难发现 WeiboMesh 在严格遵循 Service Mesh 的规范和理念的同时，是一个入门更方便而且使用更简单的 ServiceMesh 实现，它来自微博架构的实战和迭代演进。

而 Istio 我个人感觉相对较为复杂些，因为面对繁荣的原生云市场，Istio 为了支持各种平台、环境的部署，需要抽象和拟合的元素实在太多，这极大增加了整体复杂度，也对性能造成不少损失。

而且 Istio 对请求的控制依赖于每次与 Mixer 的交互，其网络消耗对系统整体性能影响极大，部署一组 Istio 集群就要部署一堆 Pod，这些都是看得见的复杂度，想满足大众的口味，就难免众口难调，这点大锅饭肯定比不上私房菜好吃。

但是 Istio 也在不断地更新，我们期待它在不远的将来能带来更精简的部署和更高的性能，不过这同样也是 WeiboMesh 所追求的，大家互通有无，一起成长。

通过上面的分析，你应该对这两款 Service Mesh 实现有了更深入的认识，不过你可能会有些疑问，为什么 WeiboMesh 的组件中，SideCar 代理有两个，确切的是说是一个代理的双重角色（Server Agent 和 Client Agent），还有 WeiboMesh 集群中，Mesh 层是通过 Motan 协议来交互的，这样的设计的出发点是什么呢？下一篇中我们将围绕这些问题进行细节阐述。

阅读过程中你有哪些问题和感悟？欢迎留言与大家交流，我们也会请作者解答你的疑问。
