## 前言

Service Mesh，也叫服务网格，号称是下一代微服务架构技术，能有效地解决当前微服务架构中关于服务治理的痛点问题，从 2016 年推出至今，一直都是架构领域的热点。

[前一篇文章](https://mp.weixin.qq.com/s?__biz=MzA5OTI1NDE0Mw==&mid=2652494152&idx=1&sn=c359fb63b1121e4a94e62ac9bbdc3a7f&scene=21#wechat_redirect)我们聊了容器化，容器化因为实现了整个部署过程的标准化，从而解决了微服务规模化部署的问题。但容器化无法解决服务运行时的问题，而 Service Mesh 则可以实现服务通信和服务治理的标准化，从而减少多业务之间由于服务治理标准不一致带来的沟通和转换成本，提升全局服务治理的效率。

下面，我们就来深入了解 Service Mesh，我们先从它的概念定义开始。

## 概念定义

**Service Mesh** 这个概念最早是由 **Buoyant** 公司的 CEO 「William Morgan」提出来的。最初，William 是在公司内部分享会上讲到了 Service Mesh。在 2017 年的时候，William 又发表了一篇文章 《What’s a service mesh？And why do I need one? 》，对 Service Mesh 做了权威的定义：

> *A service mesh is a dedicated infrastructure layer for handling service-to-service communication. It’s responsible for the reliable delivery of requests through the complex topology of services that comprise a modern, cloud native application. In practice, the service mesh is typically implemented as an array of lightweight network proxies that are deployed alongside application code, without the application needing to be aware.*

翻译过来意思就是：**服务网格是一个专注于处理服务间通信的基础设施层**。云原生应用有着复杂的服务拓扑结构，而服务网格保证请求可以在这些拓扑中可靠地穿梭。实际上，服务网格通常实现为一组轻量级网络代理，这些代理与应用程序代码一起部署，对应用程序透明。

不过，该文章其实在 2020 年 10 月 12 号更新过，且对 Service Mesh 的定义也变了：

> *A service mesh is a tool for adding observability, security, and reliability features to applications by inserting these features at the platform layer rather than the application layer.*

意思为：**服务网格是一种工具，用于向应用程序中添加可观察性、安全性和可靠性等特性，通过将这些特性插入到平台层而不是应用层的方式。**

很显然，这个定义所包含的功能范围更广了，不只是处理服务间通信了。实际上也如此，Service Mesh 已经发展为**标准化、体系化、无侵入的分布式服务治理平台**，是云原生技术栈的关键组件之一。

当然，从定义上来理解 Service Mesh，还是晦涩难懂。下面，我们从演进历程的角度来了解 Service Mesh，这会通俗易懂得多。

## 演进历程

讲 Service Mesh 的演进历程最经典的文章，我觉得还是当属 `Phil Calçado` 的一篇文章《Pattern: Service Mesh》。这一小节下面的内容大部分也是基于该文的内容而提炼并总结出来的。

先从第一代网络计算机系统说起，通信模式的演进历程如下图：

![图片](https://mmbiz.qpic.cn/mmbiz_png/Xibk1Sk7nmicmsZBjMSkm7ET8EPyEz6JfLxkNhOtswC6icozxq9ib43BtIohPxaKU9ZuPABfFMbyUE6rurtyE9DMlQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



最初，计算机是稀有且昂贵的，因此两个节点之间的每个链接都是经过精心设计和维护的。随着计算机变得越来越便宜和越来越流行，连接的数量和通过它们的数据量急剧增加。随着人们越来越依赖网络系统，工程师需要确保他们构建的软件能够满足其用户所需的服务质量。

为了达到理想的质量水平，有许多问题需要回答。人们需要找到机器彼此查找的方式，通过同一根导线处理多个同时的连接，允许机器在未直接连接时彼此通信，在网络上路由数据包，加密流量等。其中，也包括**流量控制（Flow Control）**，用于防止 A 服务器向下游的 B 服务器发送超载的数据包。最初实现是需要服务自己处理的，因此服务本身除了包含自己的业务逻辑，也包含了流量控制的处理逻辑，两者耦合在一起。

后来，为了避免每个服务都需要自己实现一套相似的网络传输处理逻辑，`TCP/IP` 等网络协议出现了，它解决了网络传输中通用的流量控制问题以及很多其他问题，将这些技术栈下移，从服务的实现中抽离出来，成为操作系统网络层的一部分。

微服务时代也面临着类似的一些东西，每个微服务除了要处理自己的业务逻辑之外，还需要处理服务发现、熔断、负载均衡、监控、跟踪等非业务功能的问题。一开始的时候，这些非业务功能的代码实现也都是由负责各服务的开发人员们自己实现的，和业务逻辑的代码耦合在一起，如下图：

![图片](https://mmbiz.qpic.cn/mmbiz_png/Xibk1Sk7nmicmsZBjMSkm7ET8EPyEz6JfLtvItf6lGfS4jbiatH2AKWHWPT4DzKicia2kkEKDf0REObMdexhbDiaIicVg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

后来，为了避免在每个服务中重写相同逻辑，一些面向微服务架构的开发框架出现了，如 Twitter 的 **Finagle**、Facebook 的 **Proxygen**、**Dubbox**、**Spring Cloud** 等，这些框架实现了服务发现、熔断等这些功能，从而开发人员使用较少的框架代码就可以将这些功能添加到服务中。

![图片](https://mmbiz.qpic.cn/mmbiz_png/Xibk1Sk7nmicmsZBjMSkm7ET8EPyEz6JfLDmn0tJR2BTG7YpbQ3ufgFPeI8NCMWptuC73wsN3XjlTb1yMtRdUphA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

这看上去很完美，但实际上存在几个比较致命的痛点：

- **框架内容太多，学习门槛很高。** 虽然框架本身屏蔽了分布式系统通信的一些通用功能实现细节，但开发者却要花更多精力去掌握和管理复杂的框架本身，在实际应用中，去追踪和解决框架出现的问题也绝非易事。就说使用最广泛的 Spring Cloud，包含了 10 几款组件，大部分人需要三到六个月才能熟练掌握。
- **关注服务间通信，影响业务迭代速度。** 业务开发团队最核心的目标应该是实现业务需求，但如今却需要花很多时间和精力关注非业务问题，业务迭代的速度明显被拖慢。
- **跨语言。** 微服务有一个重要的特性就是语言无关，但开发框架通常却只支持一种或几种特定语言，而那些没有框架支持的语言编写的服务，很难融入面向微服务的架构体系，想因地制宜的用多种语言实现架构体系中的不同模块也很难做到。
- **框架组件升级困难。** 框架以 lib 库的形式和服务联编，复杂项目依赖时的库版本兼容问题非常棘手，同时，框架库的升级也无法对服务透明，服务会因为和业务无关的 lib 库升级而被迫升级。

在这种情况下，自然会产生一个想法：**既然可以将流量控制等网络访问的技术栈下移，那是否也可以将微服务框架的这些技术栈下移成为单独的一个基础层？**

![图片](https://mmbiz.qpic.cn/mmbiz_png/Xibk1Sk7nmicmsZBjMSkm7ET8EPyEz6JfLib6sFicwqwulznZ0ZvQPQOyXuA4jicyCiaU14IrdiazRHgMjgic8AJ9GpibWQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

最理想的情况就是更改网络堆栈以添加此层，但因为标准问题，该方案并不可行。

有一些先驱者发现的解决方案是将其作为一组代理来实现。主要思路是：服务不会直接连接到其下游依赖项，而是所有流量将通过一小部分软件透明地添加所需的功能。这些代理服务通常被部署为 **sidecar**，和业务服务部署在一起，为业务服务提供额外功能。

![图片](https://mmbiz.qpic.cn/mmbiz_png/Xibk1Sk7nmicmsZBjMSkm7ET8EPyEz6JfLe5UbIE3dHia9t8UlAqfPbCSpFTSJVJQGLSd9rUWuwQUVcPJ5GGTWAMQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

在这种模型中，每个服务都将有一个伴随的代理服务，服务之间通过 `sidecar` 代理相互通信，因此，从全局视角来看，就会得到如下的部署图：

![图片](https://mmbiz.qpic.cn/mmbiz_png/Xibk1Sk7nmicmsZBjMSkm7ET8EPyEz6JfLEhjEOl6lNqw2Ch08ITKpk2ibsqhGQV9iaNfoBkVtC8FK7AYRYwKr7AhQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

可以发现，这些代理服务之间的互连形成了一个 **mesh network（网状网络）**，这就是所谓的 **Service Mesh**。不过，这只是**第一代 Service Mesh**，代表的产品有 Linkerd、Envoy、NginMesh。其特点是将所有服务通信和治理功能都放到这个代理服务中处理，导致代理很重。代理服务承载了太多的特性和功能，就会使得代理服务的更新和修改特别频繁，频繁的更新和升级就会导致代理服务出问题的概率增大，影响代理服务的稳定性。同时，代理服务承载了微服务通信的全部流量，对稳定性要求极高，这个服务的任何故障都会对整个系统的稳定性产生很大的影响。

为了解决上述频繁升级和稳定性之间的矛盾，将策略和配置决策逻辑从代理服务中脱离出来，形成了独立的**控制平面（Control Plane）**，而代理服务则被称为**数据平面（Data Plane）**，这就是**第二代 Service Mesh**，其模型图如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/Xibk1Sk7nmicmsZBjMSkm7ET8EPyEz6JfLr3htUfxpHr3E6pE2qVdVByTU5Ub3QWx5ic1q5GfCUDXYLaSPkRxXdOQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

数据平面负责代理微服务之间的通信，具体包含 RPC 通信、服务发现、负载均衡、降级熔断、限流容错等。数据平面可以认为是将 Spring Cloud、Dubbo 等语言相关的微服务框架中通信和服务治理能力独立出来的一个语言无关的进程，并且更注重通用性和扩展性。

控制平面则负责对数据平面进行管理，定义服务发现、路由、流量控制、遥测统计等策略，这些策略可以是全局的，也可以通过配置某个数据平面节点单独指定。控制平面通过一定的机制将策略下发到各个数据平面节点，数据平面节点在通信时会使用这些策略。

第二代 Service Mesh 以 **Istio** 为代表。

## Linkerd

**Linkerd** 是第一个 Service Mesh 项目，由 **Buoyant** 公司所开发，于 2018 年初加入了 `CNCF`。目前其实有两个版本：Linkerd 1.x 和 [Linkerd 2.x](Linkerd 1.x)。

**Linkerd 1.x** 版本属于第一代 Service Mesh 的实现，于 2016 年 1 月初次发布，开发语言使用 Scale，最后的版本为 1.7.4。Linkerd 1.x 提供了两种部署模型：**per-host** 和 **sidecar**。per-host 模型是每个主机部署一个 Linkerd 实例，并且该主机上的所有应用服务实例都通过该实例路由通信。

![图片](https://mmbiz.qpic.cn/mmbiz_png/Xibk1Sk7nmicmsZBjMSkm7ET8EPyEz6JfL0MerG9FoRu8gEIodThtKkYavqF58C3DMZb6pUKqwAGo6nMXM0ib8Tvw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

sidecar 模型则是每个应用服务的每个实例都部署一个 Linkerd 实例，对于主要基于实例或容器而不是基于主机的部署，此模型很有用。

![图片](https://mmbiz.qpic.cn/mmbiz_png/Xibk1Sk7nmicmsZBjMSkm7ET8EPyEz6JfLQh89qH4okU6gr3g052POgFl4bEhGcjM7ntlUVibnJbibDXTiaucxJtcjA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

应用服务和 Linkerd 如何相互通信也有三种配置：**服务到 Linkerd、Linkerd 到服务、Linkerd 到 Linkerd**。

Linkerd 1.x 版本还是比较简单的，但现在其实不推荐使用了，部分设计现在显得比较落后，内存开销也比较大，也不支持 TCP 请求。

现在比较推荐使用第二代 Service Mesh 产品，**Linkerd 2.x** 版本就是第二代的。但 Linkerd 2.x 并不是在 1.x 版本的基础上升级实现的，而是用 **Golang** 和 **Rust** 完全重写的，而且专门用于 **Kubernetes**。Linkerd 2.x 的架构如下图：

![图片](https://mmbiz.qpic.cn/mmbiz_png/Xibk1Sk7nmicmsZBjMSkm7ET8EPyEz6JfLQRa8bXJDwk28bSUoYA9GSjYKVaQKiacumCxdYTqSAKfhmU9udwUm3AA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

可以看到，Linkerd 2.x 分为了**数据平面**和**控制平面**两部分。数据平面由轻量级的代理所组成，这些代理作为 sidecar 容器和应用服务实例一起部署。控制平面则包含了一系列组件，这些组件运行在专用的 Kubernetes namespace 中。这些组件完成了很多事情：聚集遥测数据；提供面向用户的API；向数据平面代理提供控制数据等，它们共同驱动数据平面的行为。

Linkerd 2.x 的数据平面和控制平面是紧耦合的，这样做的优点是简化了配置，复杂度低，缺点自然就是扩展性差了。**尤其是如今的发展趋势明显是数据平面和控制平面将会分离，两者通过标准的 API 进行通信。**

## Envoy

**Envoy** 是一个开源的边缘和服务代理，专为云原生应用程序设计。Envoy 最初是在 `Lyft` 上构建的，后来也加入了 `CNCF`，是一个高性能 C++ 分布式代理，专门为单个服务和应用程序设计，以及为大型微服务“服务网格”体系结构设计的通信总线和“**通用数据平面**”。Envoy 的设计学习了 Nginx、HAProxy、硬件负载均衡器和云负载均衡器等解决方案，Envoy 与每个应用程序一起运行，并通过与平台无关的方式提供通用功能来抽象化网络。当基础设施中的所有服务流量都通过 Envoy 网格流动时，通过一致的可观察性，调整总体性能以及在单个位置添加基板功能，即可轻松可视化问题区域。

在 Service Mesh 中，Envoy 只做通用的数据平面。虽然 Envoy 没有自己的控制平面，但提供了标准 API 供其他控制平面接入。这非常关键，也因为此，Envoy 的热度远超过 Linkerd 1.x。如今，可以说，Envoy 已经是云原生时代数据平面的事实标准，`Istio、Kuma、AWS App Mesh` 等都使用 Envoy 作为了默认的数据平面。

Envoy 的架构如下图所示：

![图片](https://mmbiz.qpic.cn/mmbiz_png/Xibk1Sk7nmicmsZBjMSkm7ET8EPyEz6JfL59ZAiakVxibNnqFIlDiaerBPLfwiasG8BicnWcrrU7BXmdwYvrDdtEu0NwQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

Envoy 接收到请求后，会先走 `FilterChain`，通过各种 L3/L4/L7 Filter 对请求进行微处理，然后再路由到指定的集群，并通过负载均衡获取一个目标地址，最后再转发出去。其中，每一个环节可以静态配置，也可以动态服务发现。而动态服务发现就是通过 `xDS` 实现。图中最下面的一系列 **Discovery Service**，也称为**发现服务**，就统称为 **xDS**。

xDS API 中的每个配置资源都有与之关联的类型，目前支持了 8 种资源类型，其中，最为核心的有四种资源：**Listener、Router、Cluster、Filter**。

- **Listener**：Envoy 工作的基础。简单理解，Listener 是 Envoy 打开的一个监听端口，用于接收来自 Downstream 的连接。可以支持多个 Listener，多个 Listener 之间几乎所有的配置都是隔离的。Listener 配置的核心包括监听地址、FilterChain 等。Listener 对应的 `xDS` 称之为 **LDS（Listener Discovery Service）**。LDS 是 Envoy 正常工作的基础，没有 LDS，Envoy 就不能实现端口监听（如果启动配置也没有提供静态 Listener 的话），其他所有 xDS 服务也失去了作用。
- **Cluster**：对 Upstream 上游服务的抽象，每个 Upstream 上游服务都被抽象成一个 Cluster。Cluster 包含该服务的连接池、超时时间、endpoints 地址、端口、类型（类型决定了 Envoy 获取该 Cluster 具体可以访问的 endpoint 方法）等等。Cluster 对应的 `xDS` 称之为 **CDS（Cluster Discovery Service）**。一般情况下，CDS 服务会将其发现的所有可访问服务全量推送给 Envoy。与 CDS 紧密相关的另一种服务称之为 **EDS（Endpoint Discovery Service）**。CDS 服务负责 Cluster 资源的推送。而当该 Cluster 类型为 EDS 时，说明该 Cluster 的所有 endpoints 需要由 xDS 服务下发，而不使用 DNS 等去解析。下发 endpoints 的服务就称之为 EDS。
- **Router**：上下游之间的桥梁。Listener 在接收到下游连接和数据之后，由 Router 决定应该将数据交给哪一个 Cluster 处理，它定义了数据分发的规则。虽然说到 Router 大部分时候都可以默认理解为 HTTP 路由，但是 Envoy 支持多种协议，如 Dubbo、Redis 等，所以此处 Router 泛指所有用于桥接 Listener 和后端服务（不限定 HTTP）的规则与资源集合。Route 对应的 `xDS` 称之为 **RDS（Route Discovery Service）**。Router 中最核心配置包含匹配规则和目标 Cluster，此外，也可能包含重试、分流、限流等。
- **Filter**：通俗的讲，就是插件。通过 Filter 机制，Envoy 提供了极为强大的可扩展能力。在 Envoy 中，很多核心功能都使用 Filter 来实现。比如对于 HTTP 流量和服务的治理就是依赖 HttpConnectionManager（Network Filter，负责协议解析）以及 Router（负责流量分发）两个插件来实现。利用 Filter 机制，Envoy 理论上可以实现任意协议的支持以及协议之间的转换，可以对请求流量进行全方位的修改和定制。强大的 Filter 机制带来的不仅仅是强大的可扩展性，同时还有优秀的可维护性。Filter 机制让 Envoy 的使用者可以在不侵入社区源码的基础上对 Envoy 做各个方面的增强。Filter 本身并没有专门的 xDS 来发现配置。Filter 所有配置都是嵌入在 LDS、RDS 以及 CDS中的。

这四种核心资源和对应的 xDS 之间的关系如下图：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Xibk1Sk7nmicmsZBjMSkm7ET8EPyEz6JfLfkG50JjoZcG8SyACibEia07C8puarfwyMMZoXKLClsHeaOXG42RSiaicFQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

另外，CNCF 从 2019 年 5 月就已经筹建工作组，初始成员包括 Envoy 和 gRPC 项目的代表，以制定数据平面的标准 API，叫 **UDPA（Universal Data Plane API）**。UDPA 为 L4/L7 数据平面配置提供事实上的标准，类似于 SDN 中 L2/L3/L4 的 OpenFlow 所扮演的角色。UDPA 也是从 **Envoy xDS API** 逐步演进的，API 将涵盖服务发现、负载均衡分配、路由发现、监听器配置、安全发现、负载报告、运行状况检查委托等。

UDPA 目前的进展比较缓慢，但可以确定的是，**xDS 当前正在逐渐向 UDPA 靠拢，未来将基于 UDPA**。

## Istio

**Istio** 作为第二代 Service Mesh 的代表，出身名门，由 Google、IBM 和 Lyft 联手创建，Google 和 IBM 是绝对开发主力，而 Lyft 的贡献主要集中在 Envoy，Envoy 作为 Istio 的数据平面。Istio 发布之后，就一直得到社区的积极响应。如今也可以说，已经是云原生时代控制平面的事实标准。

Istio 的功能主要包括四块：

- **连接（Connect）**：智能地控制服务之间的流量和 API 调用流，进行一系列测试，并通过红黑部署逐步升级。
- **保护（Secure）**：通过托管身份验证、授权和服务之间通信加密自动保护您的服务。
- **控制（Control）**：应用策略并确保其得到执行，使得资源在消费者之间公平分配。
- **观测（Observe）**：对您的一切服务进行多样化、自动化的追踪、监控以及记录日志，以便实时了解正在发生的事情。

再来看看 Istiio 逻辑上的整体架构：

![图片](https://mmbiz.qpic.cn/mmbiz_png/Xibk1Sk7nmicmsZBjMSkm7ET8EPyEz6JfLxonU7zrMDECUPEqq7w5oNMNIS0cHHLVJhL1QicIFpNOqXEee8guV4pg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

其中，数据平面由一组智能代理（**Envoy**）所组成，被部署为 **sidecar**，负责协调和控制微服务之间的所有网络通信，并收集和报告所有网格流量的遥测数据。控制平面主要包括了 **Pilot、Citadel、Galley** 等组件，负责管理并配置代理来进行流量路由。

Pilot 为 Envoy sidecar 提供服务发现、用于智能路由的流量管理功能（例如，A/B 测试、金丝雀发布等）以及弹性功能（超时、重试、熔断器等）。Pilot 将控制流量行为的高级路由规则转换为特定于环境的配置，并在运行时将它们传播到 sidecar。Pilot 将特定于平台的服务发现机制抽象出来，并将它们合成为任何符合 Envoy API 的 sidecar 都可以使用的标准格式。

![图片](https://mmbiz.qpic.cn/mmbiz_png/Xibk1Sk7nmicmsZBjMSkm7ET8EPyEz6JfLDyWoNsozwYObcMWjEOm2QrNRUN0agXoYw1zfNCQdKBr0N8w603n5rA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

Citadel 通过内置的身份和证书管理，可以支持强大的服务到服务以及最终用户的身份验证。您可以使用 Citadel 来升级服务网格中的未加密流量。使用 Citadel，operator 可以执行基于服务身份的策略，而不是相对不稳定的 3 层或 4 层网络标识。从 0.5 版开始，您可以使用 Istio 的授权特性来控制谁可以访问您的服务。

Galley 是 Istio 的配置验证、提取、处理和分发组件。它负责将其余的 Istio 组件与从底层平台（例如 Kubernetes）获取用户配置的细节隔离开来。

另外，补充说明一下，控制平面的这些组件其实是被打包合并为一个被称为 **istiod** 的二进制文件的，这是从 Istio 1.5 版本开始的，在这之前，这些组件都被部署为单独的微服务。

## 落地

前面聊了那么多，最重要的还是如何才能将 Service Mesh 落地到生产项目中。要知道，大部分生产项目落地实践 Service Mesh 都不是从零开始的，而是在原有微服务架构的基础上进行改造升级的。

我们知道，在原有微服务架构中，微服务框架占据了核心位置，而改造为 Service Mesh 架构，就需要把微服务框架给替换掉，这就等于做一次“换心手术”。而且，从传统微服务架构转型为 Service Mesh 架构，还需要在不中断业务的情况下，完成平滑过渡。所以，面临的挑战其实不小。

要做到平滑过渡，就不能一步到位将所有微服务都从微服务框架切换到 Service Mesh。过渡的过程中，微服务框架和 Service Mesh 应该是共存的。可以先从较边缘的某块独立业务开始作为试点进行切入，运行一段时间没什么问题了，再逐步扩大至其他业务板块，包括核心业务，直至所有板块全都完成了迁移。

Service Mesh 的技术选型就没什么好说的，Istio + Envoy 基本已经成为标配。另外，Envoy 代理以 sidecar 的方式与微服务运行在一起，且通过 iptables 对服务的流量进行拦截转发，对服务来说是无感知的。而服务治理的各项功能，包括服务发现、负载均衡、熔断、限流、监控等，都是基于流量去做的，所以，实现这些功能也就对微服务没有侵入性。而微服务框架，我们都知道对业务代码是具有侵入性的。

将系统架构从传统微服务切换到 Service Mesh，主要工作可以分为四步：

1. **容器环境构建；**
2. **Service Mesh 环境构建；**
3. **移除微服务框架功能；**
4. **将微服务注入到 Service Mesh 平台。**

因为 Service Mesh 环境构建依赖于容器环境构建，所以第一步需要先容器化。而因为微服务框架对服务具有侵入性，而 Service Mesh 对服务则没有侵入性，所以对业务代码的改造主要还是移除对微服务框架的依赖。

最后，强调一下，实际上我自己也还缺乏 Service Mesh 落地的实践经验，所以上面这些也只是我个人的一点思考，不一定正确，如果有错误欢迎有经验的大佬指出。

## 总结

Service Mesh 的发展势头正旺，整个生态也肯定会逐渐向标准化的方向发展，就和容器生态类似。大势所趋之下，还是有必要深入了解这门技术，为未来做好准备。