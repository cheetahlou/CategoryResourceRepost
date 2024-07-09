
## 内容概要

本文主要分析 WeiboMesh 在 agent 方面的实现，如何结合各种注册中心来实现 WeiboMesh 的 agent。

前面我们谈到 WeiboMesh 源自我们对跨语言服务化的探索，最终我们通过在传输层和应用层之间增加对业务透明的 Mesh 层来解决大规模异构系统服务通信和治理的问题。

本文我将对这层抽象的核心基础设施层进行简要分析，希望能帮助你基于 WeiboMesh 对 Service Mesh 理念有一个更全面的理解。

## 水到渠成的 Service Mesh

在本系列文章的写作过程中，我详细调研并试用了业界几个重要的 Service Mesh 实现，发现大家基本都是在 2016 年开始做这个事情，由于都是在摸索中前进，所以实现的方式可能不尽相同，但思路和方向却出奇一致。

要知道，从传统应用演化到微服务不容易，从服务化演化出 Service Mesh 方案更不是一蹴而就的事情。

我有幸在微博亲历了这个过程，微博在微服务和服务治理方面已经有多年的技术和经验积累，即便如此，在 WeiboMesh 的演化过程中，依然经过了一个漫长曲折的过程。大家最终都相会于 Service Mesh，我认为这绝非巧合。

<img src="https://static001.geekbang.org/resource/image/53/99/533a4079197e2dc00c8d00a8a5906599.png" alt="" />

随着近年来容器、服务编排技术的不断成熟以及云计算技术的迅速普及，越来越多的团队和组织开始基于微服务化和云化的方式构建自己的应用，云原生应用发展方兴未艾，微服务架构将复杂的系统切分为数十个甚至上百个易于被小型团队所理解和维护的小服务，极大简化了开发流程，使得服务的变更、升级成本大大降低，机动性得到极大提高。

但是微服务及云化技术并不是解决分布式系统各种问题的银弹，它并不能消除分布式系统的复杂性，伴随这种架构方式改变而来的是大量微服务间的连接管理、服务治理和监控变得极其复杂等问题。

这也是为什么 2013 年以来微博完成了基于 Motan RPC 的微服务化改造后，在服务治理方面下了如此大的功夫，最终完成了 Motan 这个服务开发、治理平台的原因。

我们实现了各种负载均衡、服务发现、高可用、动态路由、熔断等功能，花很大力气解决了微服务治理的问题，最后发现在跨语言服务化中同样需要解决这些问题，于是演化出 WeiboMesh 这一层作为统一解决这些问题的最终方案。

结合微博的经历和社区其他 Mesh案例，不难得出微服务的下一步就是 Service Mesh 这样的结论，同时 Linkerd 和 Envoy 等 Service Mesh 产品加入 CNCF （Cloud Native Computing Foundation 云原生计算基金会）的事件，也从另一个角度证明了社区对 Service Mesh 理念的认可。

<img src="https://static001.geekbang.org/resource/image/47/b2/47aead7f5b5c1bdcfb251ce0e2784ab2.jpg" alt="" />

**图片来自 Service Mesh 社区敖小剑老师**

现在社区有观点把 Envoy、Linkerd 这种 SideCar 模式的代理实现看作第一代 Service Mesh，把 Istio、Conduit 看作第二代 Service Mesh，第二代 Service Mesh 除了使用 SideCar 代理作为数据面板来保障数据可靠性传输外，还通过独立的控制面板做到对服务的遥测、策略执行等精细控制。

这里我们先不关心目前是第几代，因为从生产效果来看，虽然 ServiceMesh 给我们带来诸多好处，但目前各家仍处于一个比较初级的阶段。我们更应该关注 Service Mesh 实现功能的边界，也就是Service Mesh 到底要解决哪些问题？

Service Mesh 通过在传输层和应用层间抽象出与业务无关、甚至是对其透明、公共基础的一层解决了微服务间连接、流量控制、服务治理和监控、遥测等问题。

社区对这个边界的认知是基本一致的。不管是 Istio、Conduit，还是华为的 ServiceComb、微博的 WeiboMesh，实现方案中都有数据面板和控制面板这样的概念，这已经形成了 Service Mesh 的事实规范。

<img src="https://static001.geekbang.org/resource/image/38/b2/38c150993c199bfc8235a658351b07b2.png" alt="" />

数据面板主要以 SideCar 模式与 Service 部署在一起，保障请求数据在 Service 间进行可靠传输，提供服务治理以及对请求状态数据的提取与检测等。而控制面板主要控制服务间流量流向、请求路由，提供一种对数据面板访问策略实施的干预机制以及对服务依赖、服务状态的监控等功能。下面我们就从这两个角度来看看 WeiboMesh 是如何做的。

## WeiboMesh 的数据面板

Motan-go 的 Agent 作为 WeiboMesh 的数据面板，由 Server Agent 和  Client Agent 两部分组成，Server Agent 作为服务提供方的反向代理使用，Client Agent 则作为服务依赖方调用服务的正向代理，使用时只需配置对应的 Service（所提供的服务）和 Referer（所依赖的服务）即可，两个角色的功能由同一个 Agent 进程提供，只是执行过程中使用不同的配置段而已，所以当需要更新升级的时候，可以完全独立于应用进行，发布的方式也比较灵活友好。

熟悉 Motan 的同学应该对这些配置项（比如 Service、Referer 等）不陌生。Agent 以 SideCar 模式与 Service 部署在一起，Agent 之间通过 Motan 协议进行通信（这里指 Motan-2 协议），而 Service 与 Agent 间可以使用自己喜欢的方式来进行通信，目前默认也是推荐的方式是 Service 通过 Motan-Client 与 Agent 通信，不过我们同样支持 gRPC、HTTP 等，如果需要支持其他协议也很简单，因为 Motan 中协议的扩展是特别方便的。

<img src="https://static001.geekbang.org/resource/image/d7/d3/d7eb09220ecf69ee6052974184f66bd3.png" alt="" />

这种通信模式下 Server Agent 相当于原来 Motan Server 的角色，Client Agent 相当于原来 Motan Client 的角色，而 Agent 间调用就变成普通的 Motan 请求。

所以我们很自然地复用了 Motan 原有的服务治理体系，比如服务注册与发现、负载均衡、高可用、服务监控、跟踪、依赖关系刻画、指令调度系统等。如果以后需要扩展其他的策略或功能，也只需要在这一层扩展即可，这也就是 Service Mesh 的奥妙所在吧。

## WeiboMesh 的控制面板

WeiboMesh 的控制面板前身是 Motan 的指令控制体系，为了实现对服务流量的切换、降级、分组以及支持各种发布方式，Motan 原本就结合注册中心实现了一套完善的指令控制系统，以提供一种对流量进行干预的机制。

WeiboMesh 基于这种机制并配合决策系统实现了对流量的动态控制和策略执行，同时又扩展了 Filter 机制，实现了身份验证、安全管理、监控和可视化数据收集等，收集到的遥测数据实时反馈到决策系统作为决策依据。

这里我们以一次由 IDC-A 流量暴涨导致其超负荷运转而需要将 20% 的流量自动切换到 IDC-B 为背景来描述下 WeiboMesh 控制面板在流量自动调度这方面的运行方式。

WeiboMesh 启动时除了从注册中心订阅相关依赖的服务外，还会订阅各种控制策略执行所需的指令，双向 Agent 作为数据面板承接了所有服务的访问、请求的流入流出，掌握所有请求的实时数据、请求状态，并通过 Metrics 等组件将这些状态和数据推送到决策系统（包括动态流量调度和服务治理管理体系），决策系统通过对不同类型服务预先建立的容量评估模型，实时计算出集群当前容量状态，当发现 IDC-A 的容量超过容量预警时，会自动生成相关的流量切换指令，并将生成的指令同步到注册中心，Client Agent 的服务发现模块将会实时同步更新该指令并触发 Cluster 更新所持的 Endpoint 列表，请求调用过程中负载均衡器就会根据指令将 20% 的 IDC-A 的流量自动路由到 IDC-B。

<img src="https://static001.geekbang.org/resource/image/73/17/73284623e7ffb2eb53b40173ab44ef17.png" alt="" />

WeiboMesh 就是这样通过 Agent、注册中心、决策系统在传输层和应用层间实现了对请求的高效稳定传输以及服务的统一治理。

下一篇文章中，我将从实践角度以一个请求处理为例，简要分析 WeiboMesh 在 Service Mesh 规范的实现中一些具体细节处理。希望能帮你更好地入门 WeiboMesh 以及 Service Mesh。

阅读过程中你有哪些问题和感悟？欢迎留言与大家交流，我们也会请作者解答你的疑问。
