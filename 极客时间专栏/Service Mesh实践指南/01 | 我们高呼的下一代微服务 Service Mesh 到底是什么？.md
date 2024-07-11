
## 内容概要

本系列文章是微博从服务化改造到 Service Mesh实践整个过程的分享（以微博自研 Service Mesh 解决方案 WeiboMesh 为例），主要是我们在这个过程中遇到的一些问题以及我个人关于服务化和 Service Mesh 的思考。

考虑到有的同学之前可能没有接触过 Service Mesh 这个概念，所以这里我先对 Service Mesh 做一个简单的介绍，作为后续内容的基础。

## 什么是 Service Mesh

Service Mesh 这个概念最早由开发 Linkerd 的 Buoyant, Inc 公司提出，2016年9月29日的 SF Microservices MeetUp 上他们的 CTO Oliver Gould 在关于微服务的分享中首次公开使用了这个术语。

加入 Buoyant 之前， Oliver Gould 是 Twitter 的技术 Leader，也是 Twitter RPC 框架 Finagle 的核心开发者，加入 Buoyant 后他创建了 Linkerd，Linkerd 的 Github 创建日期为 2016年1月10日，可见 Service Mesh 这个概念在 Buoyant, Inc 公司内部已经流传很久。

<img src="https://static001.geekbang.org/resource/image/a2/eb/a22cfe42468d7b9c369329152d1bbbeb.jpg" alt="" />

而 Service Mesh 这个概念的定义则是 Buoyant, Inc 公司的 CEO William Morgan 于 2017 年 4月25日在公司官网发布的题为&quot;What’s a service mesh? And why do I need one?&quot;的文章中给出的。下面我们来看一下定义的内容：

> 
WHAT IS A SERVICE MESH?


> 
A service mesh is a dedicated infrastructure layer for handling service-to-service communication. It’s responsible for the reliable delivery of requests through the complex topology of services that comprise a modern, cloud native application. In practice, the service mesh is typically implemented as an array of lightweight network proxies that are deployed alongside application code, without the application needing to be aware. (But there are variations to this idea, as we’ll see.)


原文翻译：Service Mesh 是一个基础设施层，用于处理服务间通信。云原生应用有着复杂的服务拓扑，Service Mesh 保证请求可以在这些拓扑中可靠地穿梭。在实际应用当中，Service Mesh 通常是由一系列轻量级的网络代理组成的，它们与应用程序部署在一起，但应用程序不需要知道它们的存在。

关于这个定义有以下两个值得我们关注的核心点：

1. Service Mesh 是一个专门负责请求可靠传输的基础设施层；
1. Service Mesh 的实现为一组同应用部署在一起并且对应用透明的轻量网络代理。

文中 William Morgan 对 Service Mesh 概念的补充说明也进一步明确了 Service Mesh 的职责边界。我们来看看这段说明原文：

> 
The concept of the service mesh as a separate layer is tied to the rise of the cloud native application. In the cloud native model, a single application might consist of hundreds of services; each service might have thousands of instances; and each of those instances might be in a constantly-changing state as they are dynamically scheduled by an orchestrator like Kubernetes. Not only is service communication in this world incredibly complex, it’s a pervasive and fundamental part of runtime behavior. Managing it is vital to ensuring end-to-end performance and reliability.


原文翻译：随着云原生应用的崛起，Service Mesh 逐渐成为一个独立的基础设施层。在云原生模型里，一个应用可以由数百个服务组成，每个服务可能有数千个实例，而每个实例可能会持续地发生变化。服务间通信不仅异常复杂，而且也是运行时行为的基础。管理好服务间通信对于保证端到端的性能和可靠性来说是非常重要的。

结合前面的定义，用一句话总结：服务治理和请求可靠传输就是 Service Mesh 这个基础设施层的职能边界。

## 为什么是 Service Mesh

前面对 Service Mesh 这个概念做了一个简单的说明，可能对微服务了解得不深的同学不是很好理解，为什么是 Service Mesh？之前没有 Service Mesh 的时候不也一样 OK 吗？换句话说，Service Mesh 解决了微服务架构的哪些问题呢？

要回答这个问题，我们先来回顾一下近些年大部分服务化架构演进的常见历程。

为了保证项目能快速成型并上线，起初的应用可能是一个大一统的单体服务，所有业务逻辑、数据层依赖等都在一个项目中完成，这么做的好处在于：开发独立、快速，利用资源合理，服务调用更直接、性能更高等等。

但缺点也不少，最致命的问题就是耦合，而且随着业务的不断堆叠这种耦合会越来越严重，项目越来越臃肿。

所以下一步通常都会往模块化的服务演进，比如 MVC 模式的使用、RPC 技术的引入等，主要解决业务或者功能模块之间耦合的问题。

随着业务量的增长和组织的扩大，性能和扩展性将是不得不面对的大问题，所以通常会按照 SOA 的思路进行面向服务的体系结构拆分。

随着云化技术的发展，容器技术的普及，考虑到集群的弹性运维以及业务运营维护成本、业务运营演进效率等问题，微服务架构就显得更贴近现实。

<img src="https://static001.geekbang.org/resource/image/6e/d9/6eaa78e4649cc321b4721ceebe4595d9.jpg" alt="" />

**图片来源于网络**

微服务对比之前的各种传统架构模式带来了一沓子的好处，比如降低了单体服务的复杂性，随即提升了开发速度，而且代码更容易理解和维护，开发团队对技术的选择变得更自由，每个微服务都独立部署所以能很方便调配部署的方式及规模等等。

尤其是一些新兴的互联网公司，创业初期基于运营成本考虑可能都不会去自建机房、买服务器上架，而是直接走云原生应用的路子，这样既能节省成本又能高质高效地应对快速增长的用户。

但也要考虑这个问题，随着微服务化的逐步发展和细化（这里我们暂且不考虑微服务拆分的问题，因为拆分的粒度往往没有规范可依，我个人认为适合自己的才是最好的，当然更不考虑拆分维度或者康威定律），越来越多微服务，每个微服务部署成倍数以上的微服务实例，这些实例数可能轻轻松松就会达到成百上千个，各服务间的依赖变成复杂的网状拓扑结构。

与传统基于 DNS 和中心化的 Server 端负载均衡来实现服务依赖和治理的形式相比，基于注册中心和一个去中心化的 Client 端负载均衡来实现服务注册发现和治理的微服务则是更加复杂的分布式应用，不管是部署形式还是服务的发现、服务依赖调用和请求传输等问题都变得非常难以处理。因为整个体系有太多的不稳定因素，比如网络的分区可能导致服务得不到有效治理。

这里的重点就是要面对微服务架构下服务的治理与请求的可靠传输等问题，这就是为什么需要有 Service Mesh 这样的一个基础设施层来对应用透明化地解决这些问题。

## 下一代的微服务 Service Mesh

随着云计算技术的成熟和普及以及容器技术的大量应用，为了更快地交付、降低试错风险、更快地拓展业务，云原生应用将会是未来服务演进的方向。像微博这种访问体量极大的公司，从容器技术发展前期的 2014、2015 年就开始着力发展混合云架构。另外，现在各种云计算厂商大力推广的云服务也不难佐证这一观点。

<img src="https://static001.geekbang.org/resource/image/0b/ac/0bf77f6cc89f1c81b5e7c2d049b3edac.png" alt="" />

同时，与 Kubernets 类似的容器调度编排技术的不断成熟，使得容器作为微服务的最小部署单元，这更有利于发挥微服务架构的巨大优势。并且在 Service Mesh 方向上不断涌现的各种优秀的项目比如 istio 和 Conduit（Conduit 目前仅支持 Kubernets 平台）的出现补足了 Kubernetes 等平台在微服务间服务通讯上的短板。

更关键的是，这些优秀项目的背后不乏 Google、IBM 等大公司背书，所以很容易相信未来微服务架构将围绕 Kubernetes 等容器生态来展开，这也就不难得出 Service Mesh 将是下一代微服务这样的结论。

通过本文，希望大家对 Service Mesh 有个初步的了解，后面的一系列文章我将分享我们在 Service Mesh 实践中的一些收获和感悟，希望能帮助大家搭上下一代微服务这趟快车。

阅读过程中你有哪些问题和感悟？欢迎留言与大家交流，我们也会请作者解答你的疑问。
