# Istio Handbook之Istio 服务网格进阶实战

# 一 序

Istio](https://istio.io/zh) 是由 Google、IBM、Lyft 等共同开源的 Service Mesh（服务网格）框架，于2017年初开始进入大众视野，作为云原生时代下承 Kubernetes、上接 Serverless 架构的重要基础设施层，地位至关重要。[ServiceMesher 社区](https://www.servicemesher.com/)作为中国最早的一批在研究和推广 Service Mesh 技术的开源社区决定整合社区资源，合作撰写一本开源电子书作为服务网格智库。

# 二 服务网格——后 Kubernetes 时代的微服务

听说过 Service Mesh 并试用过 [Istio](https://istio.io/zh) 的人可能都会有以下几个疑问：

1. 为什么 Istio 一定要绑定 Kubernetes 呢？
2. Kubernetes 和 Service Mesh 分别在云原生中扮演什么角色？
3. Istio 扩展了 Kubernetes 的哪些方面？解决了哪些问题？
4. Kubernetes、Envoy（xDS 协议）与 Istio 之间又是什么关系？
5. 到底该不该上 Service Mesh？

本文试图带您梳理清楚 Kubernetes、Envoy（xDS 协议）以及 Istio Service Mesh 之间的关系及内在联系。此外，本文还介绍了 Kubernetes 中的负载均衡方式，Envoy 的 xDS 协议对于 Service Mesh 的意义以及为什么说有了 Kubernetes 还需要 Istio。

使用 Service Mesh 并不是说与 Kubernetes 决裂，而是水到渠成的事情。Kubernetes 的本质是通过声明式配置对应用进行生命周期管理，而 Service Mesh 的本质是提供应用间的流量和安全性管理以及可观察性。假如你已经使用 Kubernetes 构建了稳定的微服务平台，那么如何设置服务间调用的负载均衡和流量控制？

Envoy 对于 Service Mesh 或者说 Cloud Native 最大的贡献就是定义了 xDS，Envoy 虽然本质上是一个 proxy，但是它的配置协议被众多开源软件所支持，如 [Istio](https://github.com/istio/istio)、[Linkerd](https://linkerd.io/)、[AWS App Mesh](https://aws.amazon.com/app-mesh/)、[SOFAMesh](https://github.com/alipay/sofa-mesh) 等。

## 2.1 重要观点

如果你想要提前了解下文的所有内容，那么可以先阅读下面列出的本文中的一些主要观点：

- Kubernetes 的本质是应用的生命周期管理，具体来说就是部署和管理（扩缩容、自动恢复、发布）。
- Kubernetes 为微服务提供了可扩展、高弹性的部署和管理平台。
- Service Mesh 的基础是透明代理，通过 sidecar proxy 拦截到微服务间流量后再通过控制平面配置管理微服务的行为。
- Service Mesh 将流量管理从 Kubernetes 中解耦，Service Mesh 内部的流量无需 `kube-proxy` 组件的支持，通过为更接近微服务应用层的抽象，管理服务间的流量、安全性和可观察性。
- Envoy xDS 定义了 Service Mesh 配置的协议标准。
- Service Mesh 是对 Kubernetes 中的 service 更上层的抽象，它的下一步是 serverless。

## 2.2 Kubernetes vs Service Mesh

下图展示的是 Kubernetes 与 Service Mesh 中的的服务访问关系，本文仅针对 sidecar per-pod 模式，详情请参考[服务网格的实现模式](https://jimmysong.io/istio-handbook/concepts/service-mesh-patterns.html)。

![image-20200118113133358](/Users/xuel/Library/Application Support/typora-user-images/image-20200118113133358.png)

Kubernetes 集群的每个节点都部署了一个 `kube-proxy` 组件，该组件会与 Kubernetes API Server 通信，获取集群中的 [service](https://jimmysong.io/kubernetes-handbook/concepts/service.html) 信息，然后设置 iptables 规则，直接将对某个 service 的请求发送到对应的 Endpoint（属于同一组 service 的 pod）上。

Istio Service Mesh 中沿用了 Kubernetes 中的 service 做服务注册，通过 Control Plane 来生成数据平面的配置（使用 CRD 声明，保存在 etcd 中），数据平面的**透明代理**（transparent proxy）以 sidecar 容器的形式部署在每个应用服务的 pod 中，这些 proxy 都需要请求 Control Plane 来同步代理配置，之所以说是透明代理，是因为应用程序容器完全无感知代理的存在，该过程 kube-proxy 组件一样需要拦截流量，只不过 `kube-proxy` 拦截的是进出 Kubernetes 节点的流量，而 sidecar proxy 拦截的是进出该 Pod 的流量，详见[理解 Istio Service Mesh 中 Envoy Sidecar 代理的路由转发](https://jimmysong.io/posts/envoy-sidecar-routing-of-istio-service-mesh-deep-dive/)。

### 2.2.1 **Service Mesh 的劣势**

因为 Kubernetes 每个节点上都会运行众多的 Pod，将原先 `kube-proxy` 方式的路由转发功能置于每个 pod 中，将导致大量的配置分发、同步和最终一致性问题。为了细粒度地进行流量管理，必将添加一系列新的抽象，从而会进一步增加用户的学习成本，但随着技术的普及，这样的情况会慢慢地得到缓解。 

### 2.2.2 **Service Mesh 的优势**

`kube-proxy` 的设置都是全局生效的，无法对每个服务做细粒度的控制，而 Service Mesh 通过 sidecar proxy 的方式将 Kubernetes 中对流量的控制从 service 一层抽离出来，可以做更多的扩展

## 2.3 kube-proxy 组件

在 Kubernetes 集群中，每个 Node 运行一个 `kube-proxy` 进程。`kube-proxy` 负责为 `Service` 实现了一种 VIP（虚拟 IP）的形式。 在 Kubernetes v1.0 版本，代理完全在 userspace 实现。Kubernetes v1.1 版本新增了 [iptables 代理模式](https://jimmysong.io/kubernetes-handbook/concepts/service.html#iptables-代理模式)，但并不是默认的运行模式。从 Kubernetes v1.2 起，默认使用 iptables 代理。在 Kubernetes v1.8.0-beta.0 中，添加了 [ipvs 代理模式](https://jimmysong.io/kubernetes-handbook/concepts/service.html#ipvs-代理模式)。关于 kube-proxy 组件的更多介绍请参考 [kubernetes 简介：service 和 kube-proxy 原理](https://cizixs.com/2017/03/30/kubernetes-introduction-service-and-kube-proxy/) 和 [使用 IPVS 实现 Kubernetes 入口流量负载均衡](https://jishu.io/kubernetes/ipvs-loadbalancer-for-kubernetes/)。

### 2.3.1 kube-proxy 的缺陷

在上面的链接中作者指出了 [kube-proxy 的不足之处](https://cizixs.com/2017/03/30/kubernetes-introduction-service-and-kube-proxy/)：

> 首先，如果转发的 pod 不能正常提供服务，它不会自动尝试另一个 pod，当然这个可以通过 [`liveness probes`](https://jimmysong.io/kubernetes-handbook/guide/configure-liveness-readiness-probes.html) 来解决。每个 pod 都有一个健康检查的机制，当有 pod 健康状况有问题时，kube-proxy 会删除对应的转发规则。另外，`nodePort` 类型的服务也无法添加 TLS 或者更复杂的报文路由机制。

Kube-proxy 实现了流量在 Kubernetes service 多个 pod 实例间的负载均衡，但是如何对这些 service 间的流量做细粒度的控制，比如按照百分比划分流量到不同的应用版本（这些应用都属于同一个 service，但位于不同的 deployment 上），做金丝雀发布（灰度发布）和蓝绿发布？Kubernetes 社区给出了 [使用 Deployment 做金丝雀发布的方法](https://kubernetes.io/docs/concepts/cluster-administration/manage-deployment/#canary-deployments)，该方法本质上就是通过修改 pod 的 [label](https://jimmysong.io/kubernetes-handbook/concepts/label.html) 来将不同的 pod 划归到 Deployment 的 Service 上。

## 2.4 Kubernetes Ingress vs Istio Gateway

Kubernetes 中的 Ingress 资源对象跟 Istio Service Mesh 中的 Gateway 的功能类似，都是负责集群南北流量（从集群外部进入集群内部的流量）。

`kube-proxy` 只能路由 Kubernetes 集群内部的流量，而我们知道 Kubernetes 集群的 Pod 位于 [CNI](https://jimmysong.io/kubernetes-handbook/concepts/cni.html) 创建的外网络中，集群外部是无法直接与其通信的，因此 Kubernetes 中创建了 [ingress](https://jimmysong.io/kubernetes-handbook/concepts/ingress.html) 这个资源对象，它由位于 Kubernetes [边缘节点](https://jimmysong.io/kubernetes-handbook/practice/edge-node-configuration.html)（这样的节点可以是很多个也可以是一组）的 Ingress controller 驱动，负责管理**南北向流量**（从集群外部进入 Kubernetes 集群的流量），Ingress 必须对接各种 Ingress Controller 才能使用，比如 [nginx ingress controller](https://github.com/kubernetes/ingress-nginx)、[traefik](https://traefik.io/)。Ingress 只适用于 HTTP 流量，使用方式也很简单，只能对 service、port、HTTP 路径等有限字段匹配来路由流量，这导致它无法路由如 MySQL、redis 和各种私有 RPC 等 TCP 流量。要想直接路由南北向的流量，只能使用 Service 的 LoadBalancer 或 NodePort，前者需要云厂商支持而且可能需要付费，后者需要进行额外的端口管理。有些 Ingress controller 支持暴露 TCP 和 UDP 服务，但是只能使用 Service 来暴露，Ingress 本身是不支持的，例如 [nginx ingress controller](https://kubernetes.github.io/ingress-nginx/user-guide/exposing-tcp-udp-services/)，服务的暴露的端口是通过创建 ConfigMap 的方式来配置的。

Istio `Gateway` 描述的负载均衡器用于承载进出网格边缘的连接。该规范中描述了一系列开放端口和这些端口所使用的协议、负载均衡的 SNI 配置等内容。Gateway 是一种 [CRD 扩展](https://jimmysong.io/kubernetes-handbook/concepts/crd.html)，它同时复用了 Envoy proxy 的能力，详细配置请参考 [Istio 官网](https://istio.io/zh/docs/reference/config/istio.networking.v1alpha3/#gateway)。

## 2.5 xDS 协议

下面这张图大家在了解 Service Mesh 的时候可能都看到过，每个方块代表一个服务的示例，例如 Kubernetes 中的一个 Pod（其中包含了 sidecar proxy），xDS 协议控制了 Istio Service Mesh 中所有流量的具体行为，即将下图中的方块链接到了一起。

![image-20200118115002660](/Users/xuel/Library/Application Support/typora-user-images/image-20200118115002660.png)

图片 - Service Mesh 示意图

xDS 协议是由 [Envoy](https://envoyproxy.io/) 提出的，在 Envoy v2 版本 API 中最原始的 xDS 协议只指 CDS、EDS、LDS 和 RDS。

下面我们以两个 service，每个 service 都有两个实例的例子来看下 Envoy 的 xDS 协议。

![image-20200118115036328](/Users/xuel/Library/Application Support/typora-user-images/image-20200118115036328.png)





上图中的箭头不是流量在进入 Enovy Proxy 后的路径或路由，而是想象的一种 Envoy 中 xDS 接口处理的顺序并非实际顺序，其实 xDS 之间也是有交叉引用的。

Envoy 通过查询文件或管理服务器来动态发现资源。概括地讲，对应的发现服务及其相应的 API 被称作 *xDS*。Envoy 通过**订阅（subscription）**方式来获取资源，订阅方式有以下三种：

- **文件订阅**：监控指定路径下的文件，发现动态资源的最简单方式就是将其保存于文件，并将路径配置在 [ConfigSource](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/core/config_source.proto#core-configsource) 中的 `path` 参数中。
- **gRPC 流式订阅**：每个 xDS API 可以单独配置 [`ApiConfigSource`](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/core/config_source.proto#core-apiconfigsource)，指向对应的上游管理服务器的集群地址。
- **轮询 REST-JSON 轮询订阅**：单个 xDS API 可对 REST 端点进行的同步（长）轮询。

以上的 xDS 订阅方式详情请参考 [xDS 协议解析](https://jimmysong.io/istio-handbook/concepts/envoy-xds-protocol.html)。Istio 使用的 gRPC 流式订阅的方式配置所有的数据平面的 sidecar proxy。

关于 xDS 协议的详细分解请参考丁轶群博士的这几篇文章：

- [Service Mesh深度学习系列part1—istio源码分析之pilot-agent模块分析](http://www.servicemesher.com/blog/istio-service-mesh-source-code-pilot-agent-deepin)
- [Service Mesh深度学习系列part2—istio源码分析之pilot-discovery模块分析](http://www.servicemesher.com/blog/istio-service-mesh-source-code-pilot-discovery-module-deepin)
- [Service Mesh深度学习系列part3—istio源码分析之pilot-discovery模块分析（续）](http://www.servicemesher.com/blog/istio-service-mesh-source-code-pilot-discovery-module-deepin-part2)

文章中介绍了 Istio pilot 的总体架构、Envoy 配置的生成、pilot-discovery 模块的功能，以及 xDS 协议中的 CDS、EDS 及 ADS，关于 ADS 详情请参考 [Enovy 官方文档](https://www.envoyproxy.io/docs/envoy/latest/configuration/overview/v2_overview#aggregated-discovery-service)。



### 2.5.1 xDS 协议要点

最后总结下关于 xDS 协议的要点：

- CDS、EDS、LDS、RDS 是最基础的 xDS 协议，它们可以分别独立更新的。
- 所有的发现服务（Discovery Service）可以连接不同的 Management Server，也就是说管理 xDS 的服务器可以是多个。
- Envoy 在原始 xDS 协议的基础上进行了一些列扩充，增加了 SDS（秘钥发现服务）、ADS（聚合发现服务）、HDS（健康发现服务）、MS（Metric 服务）、RLS（速率限制服务）等 API。
- 为了保证数据一致性，若直接使用 xDS 原始 API 的话，需要保证这样的顺序更新：CDS --> EDS --> LDS --> RDS，这是遵循电子工程中的**先合后断**（Make-Before-Break）原则，即在断开原来的连接之前先建立好新的连接，应用在路由里就是为了防止设置了新的路由规则的时候却无法发现上游集群而导致流量被丢弃的情况，类似于电路里的断路。
- CDS 设置 Service Mesh 中有哪些服务。
- EDS 设置哪些实例（Endpoint）属于这些服务（Cluster）。
- LDS 设置实例上监听的端口以配置路由。
- RDS 最终服务间的路由关系，应该保证最后更新 RDS。

## 2.6 Envoy



Envoy 是 Istio Service Mesh 中默认的 Sidecar，Istio 在 Enovy 的基础上按照 Envoy 的 xDS 协议扩展了其控制平面，在讲到 Envoy xDS 协议之前还需要我们先熟悉下 Envoy 的基本术语。下面列举了 Envoy 里的基本术语及其数据结构解析，关于 Envoy 的详细介绍请参考 [Envoy 官方文档](http://www.servicemesher.com/envoy/)，至于 Envoy 在 Service Mesh（不仅限于 Istio） 中是如何作为转发代理工作的请参考网易云刘超的这篇[深入解读 Service Mesh 背后的技术细节 ](https://www.cnblogs.com/163yun/p/8962278.html)以及[理解 Istio Service Mesh 中 Envoy 代理 Sidecar 注入及流量劫持](https://jimmysong.io/posts/envoy-sidecar-injection-in-istio-service-mesh-deep-dive/)，本文引用其中的一些观点，详细内容不再赘述。

![image-20200118115725446](/Users/xuel/Library/Application Support/typora-user-images/image-20200118115725446.png)

### 2.6.1 基本术语

下面是您应该了解的 Enovy 里的基本术语：

- **Downstream（下游）**：下游主机连接到 Envoy，发送请求并接收响应，即发送请求的主机。
- **Upstream（上游）**：上游主机接收来自 Envoy 的连接和请求，并返回响应，即接受请求的主机。
- **Listener（监听器）**：监听器是命名网地址（例如，端口、unix domain socket 等)，下游客户端可以连接这些监听器。Envoy 暴露一个或者多个监听器给下游主机连接。
- **Cluster（集群）**：集群是指 Envoy 连接的一组逻辑相同的上游主机。Envoy 通过[服务发现](http://www.servicemesher.com/envoy/intro/arch_overview/service_discovery.html#arch-overview-service-discovery)来发现集群的成员。可以选择通过[主动健康检查](http://www.servicemesher.com/envoy/intro/arch_overview/health_checking.html#arch-overview-health-checking)来确定集群成员的健康状态。Envoy 通过[负载均衡策略](http://www.servicemesher.com/envoy/intro/arch_overview/load_balancing.html#arch-overview-load-balancing)决定将请求路由到集群的哪个成员。

Envoy 中可以设置多个 Listener，每个 Listener 中又可以设置 filter chain（过滤器链表），而且过滤器是可扩展的，这样就可以更方便我们操作流量的行为，例如设置加密、私有 RPC 等。

xDS 协议是由 Envoy 提出的，现在是 Istio 中默认的 sidecar proxy，但只要实现 xDS 协议理论上都是可以作为 Istio 中的 sidecar proxy 的，例如蚂蚁金服开源的 [SOFAMosn](https://github.com/alipay/sofa-mosn) 和 nginx 开源的 [nginmesh](https://github.com/nginxinc/nginmesh)。

## 2.7 Istio Service Mesh

![image-20200118120308772](/Users/xuel/Library/Application Support/typora-user-images/image-20200118120308772.png)

图片 - Istio service mesh 架构图

Istio 是一个功能十分丰富的 Service Mesh，它包括如下功能：

- 流量管理：这是 Istio 的最基本的功能。
- 策略控制：通过 Mixer 组件和各种适配器来实现，实现访问控制系统、遥测捕获、配额管理和计费等。
- 可观测性：通过 Mixer 来实现。
- 安全认证：Citadel 组件做密钥和证书管理。

### 2.7.1 Istio 中的流量管理

Istio 中定义了如下的 [CRD](https://jimmysong.io/kubernetes-handbook/concepts/custom-resource.html) 来帮助用户进行流量管理：

- **Gateway**：Gateway 描述了在网络边缘运行的负载均衡器，用于接收传入或传出的HTTP / TCP连接。
- **VirtualService**：[VirtualService](https://istio.io/zh/docs/reference/config/istio.networking.v1alpha3/#virtualservice) 实际上将 Kubernetes 服务连接到 Istio Gateway。它还可以执行更多操作，例如定义一组流量路由规则，以便在主机被寻址时应用。
- **DestinationRule**：`DestinationRule` 所定义的策略，决定了经过路由处理之后的流量的访问策略。简单的说就是定义流量如何路由。这些策略中可以定义负载均衡配置、连接池尺寸以及外部检测（用于在负载均衡池中对不健康主机进行识别和驱逐）配置。
- **EnvoyFilter**：`EnvoyFilter` 对象描述了针对代理服务的过滤器，这些过滤器可以定制由 Istio Pilot 生成的代理配置。这个配置初级用户一般很少用到。
- **ServiceEntry**：默认情况下 Istio Service Mesh 中的服务是无法发现 Mesh 外的服务的，`ServiceEntry` 能够在 Istio 内部的服务注册表中加入额外的条目，从而让网格中自动发现的服务能够访问和路由到这些手工加入的服务。



## 2.8 Kubernetes vs Envoy xDS vs Istio

在阅读完上文对 Kubernetes 的 `kube-proxy` 组件、Envoy xDS 和 Istio 中流量管理的抽象概念之后，下面将带您仅就流量管理方面比较下三者对应的组件/协议（注意，三者不可以完全等同）。

| Kubernetes | Envoy xDS | Istio Service Mesh |
| ---------- | --------- | ------------------ |
| Endpoint   | Endpoint  | -                  |
| Service    | Route     | VirtualService     |
| kube-proxy | Route     | DestinationRule    |
| kube-proxy | Listener  | EnvoyFilter        |
| Ingress    | Listener  | Gateway            |
| Service    | Cluster   | ServiceEntry       |

## 2.9 总结

如果说 Kubernetes 管理的对象是 Pod，那么 Service Mesh 中管理的对象就是一个个 Service，所以说使用 Kubernetes 管理微服务后再应用 Service Mesh 就是水到渠成了，如果连 Service 你也不想管了，那就用如 [knative](https://github.com/knative/) 这样的 serverless 平台，但这就是后话了。

Envoy 的功能也不只是做流量转发，以上概念只不过是 Istio 在 Kubernetes 之上新增一层抽象层中的冰山一角，但因为流量管理是服务网格最基础也是最重要的功能，所以这将成为本书的开始。



# 三 概念原理

## 3.1 Istio概念原理

本章主要介绍 Istio 和 Service Mesh（服务网格）的概念和原理。

## 3.2 什么是服务网格？

Service mesh 又译作 “服务网格”，作为服务间通信的基础设施层。Buoyant 公司的 CEO Willian Morgan 在他的这篇文章 [WHAT’S A SERVICE MESH? AND WHY DO I NEED ONE?](https://buoyant.io/2017/04/25/whats-a-service-mesh-and-why-do-i-need-one/) 中解释了什么是 Service Mesh，为什么云原生应用需要 Service Mesh。

服务网格是用于处理服务间通信的专用基础设施层。它负责通过包含现代云原生应用程序的复杂服务拓扑来可靠地传递请求。实际上，服务网格通常通过一组轻量级网络代理来实现，这些代理与应用程序代码一起部署，而不需要感知应用程序本身。—— [Willian Morgan](https://twitter.com/wm) Buoyant CEO

服务网格（Service Mesh）这个术语通常用于描述构成这些应用程序的微服务网络以及应用之间的交互。随着规模和复杂性的增长，服务网格越来越难以理解和管理。它的需求包括服务发现、负载均衡、故障恢复、指标收集和监控以及通常更加复杂的运维需求，例如 A/B 测试、金丝雀发布、限流、访问控制和端到端认证等。

### 3.2.1 服务网格的特点

服务网格有如下几个特点：

- 应用程序间通讯的中间层
- 轻量级网络代理
- 应用程序无感知
- 解耦应用程序的重试/超时、监控、追踪和服务发现

目前两款流行的服务网格开源软件 [Linkerd](https://linkerd.io/) 和 [Istio](https://istio.io/) 都可以直接在 Kubernetes 中集成，其中 Linkerd 已经成为 CNCF 成员，Istio 在 2018年7月31日宣布 [1.0](https://istio.io/zh/blog/2018/announcing-1.0/)。

### 3.2.2 理解服务网格

如果用一句话来解释什么是服务网格，可以将它比作是应用程序或者说微服务间的 TCP/IP，负责服务之间的网络调用、限流、熔断和监控。对于编写应用程序来说一般无须关心 TCP/IP 这一层（比如通过 HTTP 协议的 RESTful 应用），同样使用服务网格也就无须关心服务之间的那些原来是通过应用程序或者其他框架实现的事情，比如 Spring Cloud、OSS，现在只要交给服务网格就可以了。

[Phil Calçado](http://philcalcado.com/) 在他的这篇博客 [Pattern: Service Mesh](http://philcalcado.com/2017/08/03/pattern_service_mesh.html) 中详细解释了服务网格的来龙去脉：

1. 从最原始的主机之间直接使用网线相连
2. 网络层的出现
3. 集成到应用程序内部的控制流
4. 分解到应用程序外部的控制流
5. 应用程序中集成服务发现和断路器
6. 出现了专门用于服务发现和断路器的软件包/库，如 [Twitter 的 Finagle](https://finagle.github.io/) 和 [Facebook 的 Proxygen](https://code.fb.com/networking-traffic/introducing-proxygen-facebook-s-c-http-framework/)，这时候还是集成在应用程序内部
7. 出现了专门用于服务发现和断路器的开源软件，如 [Netflix OSS](http://netflix.github.io/)、Airbnb 的 [synapse](https://github.com/airbnb/synapse) 和 [nerve](https://github.com/airbnb/nerve)
8. 最后作为微服务的中间层服务网格出现

服务网格的架构如下图所示：

![image-20200118120928192](/Users/xuel/Library/Application Support/typora-user-images/image-20200118120928192.png)

图片来自：[Pattern: Service Mesh](http://philcalcado.com/2017/08/03/pattern_service_mesh.html)

服务网格作为 sidecar 运行，对应用程序来说是透明，所有应用程序间的流量都会通过它，所以对应用程序流量的控制都可以在 service mesh 中实现。

### 3.2.3 服务网格如何工作？

下面以 Istio 为例讲解服务网格如何在 Kubernetes 中工作。

1. Istio 将服务请求路由到目的地址，根据其中的参数判断是到生产环境、测试环境还是 staging 环境中的服务（服务可能同时部署在这三个环境中），是路由到本地环境还是公有云环境？所有的这些路由信息可以动态配置，可以是全局配置也可以为某些服务单独配置。
2. 当 Istio 确认了目的地址后，将流量发送到相应服务发现端点，在 Kubernetes 中是 service，然后 service 会将服务转发给后端的实例。
3. Istio 根据它观测到最近请求的延迟时间，选择出所有应用程序的实例中响应最快的实例。
4. Istio 将请求发送给该实例，同时记录响应类型和延迟数据。
5. 如果该实例挂了、不响应了或者进程不工作了，Istio 将把请求发送到其他实例上重试。
6. 如果该实例持续返回 error，Istio 会将该实例从负载均衡池中移除，稍后再周期性的重试。
7. 如果请求的截止时间已过，Istio 主动以失败的方式结束该请求，而不是再次尝试添加负载。
8. Istio 以 metric 和分布式追踪的形式捕获上述行为的各个方面，这些追踪信息将发送到集中 metric 系统。

### 3.2.4 为何使用服务网格？

服务网格并没有给我们带来新功能，它是用于解决其他工具已经解决过的问题，只不过这次是在云原生的 Kubernetes 环境下的实现。

在传统的 MVC 三层 Web 应用程序架构下，服务之间的通讯并不复杂，在应用程序内部自己管理即可，但是在现今的复杂的大型网站情况下，单体应用被分解为众多的微服务，服务之间的依赖和通讯十分复杂，出现了 twitter 开发的 [Finagle](https://twitter.github.io/finagle/)、Netflix 开发的 [Hystrix](https://github.com/Netflix/Hystrix) 和 Google 的 Stubby 这样的 “胖客户端” 库，这些就是早期的服务网格，但是它们都仅适用于特定的环境和特定的开发语言，并不能作为平台级的服务网格支持。

在云原生架构下，容器的使用给予了异构应用程序的更多可行性，Kubernetes 增强的应用的横向扩容能力，用户可以快速的编排出复杂环境、复杂依赖关系的应用程序，同时开发者又无须过多关心应用程序的监控、扩展性、服务发现和分布式追踪这些繁琐的事情而专注于程序开发，赋予开发者更多的创造性。

## 3.3 服务网格架构

下图是[Conduit](https://condiut.io/) Service Mesh（现在已合并到Linkerd2中了）的架构图，这是Service Mesh的一种典型的架构。

![image-20200118121600826](/Users/xuel/Library/Application Support/typora-user-images/image-20200118121600826.png)

服务网格中分为**控制平面**和**数据平面**，当前流行的两款开源的服务网格 Istio 和 Linkerd 实际上都是这种架构，只不过 Istio 的划分更清晰，而且部署更零散，很多组件都被拆分，控制平面中包括 Mixer、Pilot、Citadel，数据平面默认是用 Envoy；而 Linkerd 中只分为 Linkerd 做数据平面，namerd 作为控制平面。



控制平面的特点：

- 不直接解析数据包
- 与数据平面中的代理通信，下发策略和配置
- 负责网络行为的可视化
- 通常提供 API 或者命令行工具可用于配置版本化管理，便于持续集成和部署

### 

数据平面的特点：

- 通常是按照无状态目标设计的，但实际上为了提高流量转发性能，需要缓存一些数据，因此无状态也是有争议的
- 直接处理入站和出站数据包，转发、路由、健康检查、负载均衡、认证、鉴权、产生监控数据等
- 对应用来说透明，即可以做到无感知部署

### 3.3.1 服务网格的实现模式

我们在前面看到了通过**客户端库**来治理服务的架构图，那是我们在改造成 Service Mesh 架构前使用微服务架构通常的形式，下图是使用 Service Mesh 架构的最终形式。

​	![image-20200118122002081](/Users/xuel/Library/Application Support/typora-user-images/image-20200118122002081.png)

当然在达到这一最终形态之前我们需要将架构一步步演进，下面给出的是参考的演进路线。

#### 3.3.1.1 Ingress 或边缘代理

如果你使用的是 Kubernetes 做容器编排调度，那么在进化到 Service Mesh 架构之前，通常会使用 Ingress Controller，做集群内外流量的反向代理，如使用 Traefik 或 Nginx Ingress Controller。

![image-20200118122043155](/Users/xuel/Library/Application Support/typora-user-images/image-20200118122043155.png)

这样只要利用 Kubernetes 的原有能力，当你的应用微服务化并容器化需要开放外部访问且只需要 L7 代理的话这种改造十分简单，但问题是无法管理服务间流量。

#### 3.3.1.2 路由器网格

Ingress 或者边缘代理可以处理进出集群的流量，为了应对集群内的服务间流量管理，我们可以在集群内加一个 `Router` 层，即路由器层，让集群内所有服务间的流量都通过该路由器。

![image-20200118122140763](/Users/xuel/Library/Application Support/typora-user-images/image-20200118122140763.png)

这个架构无需对原有的单体应用和新的微服务应用做什么改造，可以很轻易的迁移进来，但是当服务多了管理起来就很麻烦。

#### 3.3.1.3 Proxy per Node

这种架构是在每个节点上都部署一个代理，如果使用 Kubernetes 来部署的话就是使用 `DaemonSet` 对象，Linkerd 第一代就是使用这种方式部署的，一代的 Linkerd 使用 Scala 开发，基于 JVM 比较消耗资源，二代的 Linkerd 使用 Go 开发。

![image-20200118122212926](/Users/xuel/Library/Application Support/typora-user-images/image-20200118122212926.png)

这种架构有个好处是每个节点只需要部署一个代理即可，比起在每个应用中都注入一个 sidecar 的方式更节省资源，而且更适合基于物理机/虚拟机的大型单体应用，但是也有一些副作用，比如粒度还是不够细，如果一个节点出问题，该节点上的所有服务就都会无法访问，对于服务来说不是完全透明的。

#### 3.3.1.4 Sidecar 代理/Fabric 模型

这个一般不会成为典型部署类型，当企业的服务网格架构演进到这一步时通常只会持续很短时间，然后就会增加控制平面。跟前几个阶段最大的不同就是，应用程序和代理被放在了同一个部署单元里，可以对应用程序的流量做更细粒度的控制。

![image-20200118122322232](/Users/xuel/Library/Application Support/typora-user-images/image-20200118122322232.png)

这已经是最接近 Service Mesh 架构的一种形态了，唯一缺的就是控制平面了。所有的 sidecar 都支持热加载，配置的变更可以很容易的在流量控制中反映出来，但是如何操作这么多 sidecar 就需要一个统一的控制平面了。

#### 3.3.1.5 Sidecar 代理/控制平面

下面的示意图是目前大多数 Service Mesh 的架构图，也可以说是整个 Service Mesh 架构演进的最终形态。

![image-20200118122414509](/Users/xuel/Library/Application Support/typora-user-images/image-20200118122414509.png)

这种架构将代理作为整个服务网格中的一部分，使用 Kubernetes 部署的话，可以通过以 sidecar 的形式注入，减轻了部署的负担，可以对每个服务做细粒度权限与流量控制。但有一点不好就是为每个服务都注入一个代理会占用很多资源，因此要想方设法降低每个代理的资源消耗。

#### 3.3.1.6 多集群部署和扩展

以上都是单个服务网格集群的架构，所有的服务都位于同一个集群中，服务网格管理进出集群和集群内部的流量，当我们需要管理多个集群或者是引入外部的服务时就需要[网格扩展](https://preliminary.istio.io/zh/docs/setup/kubernetes/additional-setup/mesh-expansion/)和[多集群配置](https://istio.io/docs/setup/kubernetes/multicluster-install/)。

### 3.3.2 Istio 架构解析

下面是以漫画的形式说明 Istio 是什么。

![image-20200118122458137](/Users/xuel/Library/Application Support/typora-user-images/image-20200118122458137.png)

![image-20200118122528688](/Users/xuel/Library/Application Support/typora-user-images/image-20200118122528688.png)

- Istio 是独立于平台的，可以在 Kubernetes 、 Consul 、虚拟机上部署的服务
- Istio 的组成
  - Envoy：智能代理、流量控制
  - Pilot：服务发现、流量管理
  - Mixer：访问控制、遥测
  - Citadel：终端用户认证、流量加密
  - Galley（1.1新增）：验证、处理和分配配置
- Service mesh 关注的方面
  - 可观察性
  - 安全性
  - 可运维性
  - 可拓展性
- Istio 的策略执行组件可以扩展和定制，同时也是可拔插的
- Istio 在数据平面为每个服务中注入一个 Envoy 代理以 Sidecar 形式运行来劫持所有进出服务的流量，同时对流量加以控制，通俗的讲就是应用程序你只管处理你的业务逻辑，其他的事情 Sidecar 会汇报给 Istio 控制平面处理
- 应用程序只需关注于业务逻辑（这才能生钱）即可，非功能性需求交给 Istio

#### 3.3.2.1 设计目标

Istio 的架构设计中有几个关键目标，这些目标对于使系统能够应对大规模流量和高性能地服务处理至关重要。

- 最大化透明度：若想 Istio 被采纳，应该让运维和开发人员只需付出很少的代价就可以从中受益。为此，Istio 将自身自动注入到服务间所有的网络路径中。Istio 使用 sidecar 代理来捕获流量，并且在尽可能的地方自动编程网络层，以路由流量通过这些代理，而无需对已部署的应用程序代码进行任何改动。在 Kubernetes中，代理被注入到 pod 中，通过编写 iptables 规则来捕获流量。注入 sidecar 代理到 pod 中并且修改路由规则后，Istio 就能够调解所有流量。这个原则也适用于性能。当将 Istio 应用于部署时，运维人员可以发现，为提供这些功能而增加的资源开销是很小的。所有组件和 API 在设计时都必须考虑性能和规模。
- 可扩展性：随着运维人员和开发人员越来越依赖 Istio 提供的功能，系统必然和他们的需求一起成长。虽然我们期望继续自己添加新功能，但是我们预计最大的需求是扩展策略系统，集成其他策略和控制来源，并将网格行为信号传播到其他系统进行分析。策略运行时支持标准扩展机制以便插入到其他服务中。此外，它允许扩展词汇表，以允许基于网格生成的新信号来执行策略。
- 可移植性：使用 Istio 的生态系统将在很多维度上有差异。Istio 必须能够以最少的代价运行在任何云或预置环境中。将基于 Istio 的服务移植到新环境应该是轻而易举的，而使用 Istio 将一个服务同时部署到多个环境中也是可行的（例如，在多个云上进行冗余部署）。
- 策略一致性：在服务间的 API 调用中，策略的应用使得可以对网格间行为进行全面的控制，但对于无需在 API 级别表达的资源来说，对资源应用策略也同样重要。例如，将配额应用到 ML 训练任务消耗的 CPU 数量上，比将配额应用到启动这个工作的调用上更为有用。因此，策略系统作为独特的服务来维护，具有自己的 API，而不是将其放到代理/sidecar 中，这容许服务根据需要直接与其集成。

### 3.3.3 从边车模式到 Service Mesh

所谓边车模式（ Sidecar pattern ），也译作挎斗模式，是分布式架构中云设计模式的一种。因为其非常类似于生活中的边三轮摩托车而得名。该设计模式通过给应用程序加上一个“边车”的方式来拓展应用程序现有的功能。这种设计模式出现的很早，实现的方式也多种多样。现在这个模式更是随着微服务的火热与 Service Mesh 的逐渐成熟而进入人们的视野。

#### 3.3.3.1 什么是边车模式

![image-20200118123006091](/Users/xuel/Library/Application Support/typora-user-images/image-20200118123006091.png)

在 [Azure Architecture Center](https://docs.microsoft.com/en-us/azure/architecture/patterns/) 的云设计模式中是这么介绍边车模式的：

> Deploy components of an application into a separate process or container to provide isolation and encapsulation.
>
> --- **Sidecar pattern**

**这里要注意的是： 这里的 Sidecar 是分布式架构中云设计模式的一种，与我们目前在使用的 Istio 或 Linkerd 中的 Sidecar 是设计与实现的区别，后文中提到的边车模式均是指这种设计模式，请勿与 Istio 或 其他 Service Mesh 软件 中的 Sidecar 混淆。**

**边车模式**是一种分布式架构的设计模式。如上图所示，边车就是加装在摩托车旁来达到拓展功能的目的，比如行驶更加稳定，可以拉更多的人和货物，坐在边车上的人可以给驾驶员指路等。边车模式通过给应用服务加装一个“边车”来达到**控制**和**逻辑**的分离的目的。

比如日志记录、监控、流量控制、服务注册、服务发现、服务限流、服务熔断等在业务服务中不需要实现的控制面功能，可以交给“边车”，业务服务只需要专注实现业务逻辑即可。如上图那样，应用服务你只管开好你的车，打仗的事情就交给边车上的代理就好。这与分布式和微服务架构完美契合，真正的实现了控制和逻辑的分离与解耦。

#### 3.3.3.2 边车模式设计

在设计上边车模式与网关模式有类似之处，但是其粒度更细。其为每个服务都配备一个“边车”，这个“边车“可以理解为一个 agent ，这个服务所有的通信都是通过这个 agent 来完成的，这个 agent 同服务一起创建，一起销毁。像服务注册、服务发现、监控、流量控制、日志记录、服务限流和服务熔断等功能完全可以做成标准化的组件和模块，不需要在单独实现其功能来消耗业务开发的精力和时间来开发和调试这些功能，这样可以开发出真正高内聚低耦合的软件。

这里有两种方法来实现边车模式：

**通过 SDK 、 Lib 等软件包的形式，在开发时引入该软件包依赖，使其与业务服务集成起来。**

这种方法可以与应用密切集成，提高资源利用率并且提高应用性能。但是这种方法是对代码有侵入的，受到编程语言和软件开发人员水平的限制，但当该依赖有 bug 或者需要升级时，业务代码需要重新编译和发布。同时，如果该依赖宣布停止维护或者闭源，那么会给该服务带来不小的影响。

**以 Sidecar 的形式，在运维的时候与应用服务集成在一起。**

这种方式对应用服务没有侵入性，不受编程语言和开发人员水平的限制，做到了控制与逻辑分开部署。但是会增加应用延迟，并且管理和部署的复杂度会增加。

#### 3.3.3.3 边车模式解决了什么问题

边车模式在概念上是比较简单的，但是在实践中还是要了解边车模式到底解决了什么问题，我们为什么要使用边车模式？

**控制与逻辑分离的问题**

边车模式是基于将控制与逻辑分离和解耦的思想，通俗的讲就是让专业的人做专业的事，业务代码只需要关心其复杂的业务逻辑，其他的事情”边车“会帮其处理，从这个角度看，可能叫跟班或者秘书模式也不错 :)

日志记录、监控、流量控制、服务注册、服务发现、服务限流、服务熔断、鉴权、访问控制和服务调用可视化等，这些功能从本质上和业务服务的关系并不大，而传统的软件工程在开发层面完成这些功能，这导致了各种各样维护上的问题。

就好像一个厨师不是必须去关心食材的产地、饭店的选址、是给大厅的客人上菜还是给包房的客人上菜...他只需要做好菜就好，虽然上面的这些事他都可以做。而传统的软件工程就像是一个小饭店的厨师，他即是老板又是厨师，既需要买菜又需要炒菜，所有的事情都要他一个人做，如果客人一多，就会变的手忙脚乱；而控制与逻辑分离的软件，其逻辑部分就像是高档酒店的厨师，他只需要将菜做好即可，其他的事情由像”边车“这样的成员帮其处理。

**解决服务之间调用越来越复杂的问题**

随着分布式架构越来越复杂和微服务越拆越细，我们越来越迫切的希望有一个统一的控制面来管理我们的微服务，来帮助我们维护和管理所有微服务，这时传统开发层面上的控制就远远不够了。而边车模式可以很好的解决这个问题。

#### 3.3.3.4 从边车模式到 Service Mesh

边车模式有效的分离了系统控制和业务逻辑，可以将所有的服务进行统一管理，让开发人员更专注于业务开发，显著的提升开发效率。而遵循这种模式进行实践从很早以前就开始了，开发人员一直试图将上文中我们提到的功能（如：流量控制、服务注册、服务发现、服务限流、服务熔断等）提取成一个标准化的 Sidecar ，通过 Sidecar 代理来与其他系统进行交互，这样可以大大简化业务开发和运维。而随着分布式架构和微服务被越来越多的公司和开发者接受并使用，这一需求日益凸显。

这就是 Service Mesh 服务网格诞生的契机，它是 CNCF（Cloud Native Computing Foundation，云原生基金会）目前主推的新一代微服务架构。 William Morgan 在 [What's a service mesh? And why do I need one?](https://buoyant.io/2017/04/25/whats-a-service-mesh-and-why-do-i-need-one/) 【[译文](https://blog.maoxianplay.com/posts/whats-a-service-mesh-and-why-do-i-need-one/)】中解释了什么是 Service Mesh 。

Service Mesh 有如下几个特点：

- 应用程序间通讯的中间层
- 轻量级网络代理
- 应用程序无感知
- 解耦应用程序的重试/超时、监控、追踪和服务发现

Service Mesh 将底层那些难以控制的网络通讯统一管理，诸如：流量管控，丢包重试，访问控制等。而上层的应用层协议只需关心业务逻辑即可。Service Mesh 是一个用于处理服务间通信的基础设施层，它负责为构建复杂的云原生应用传递可靠的网络请求。

#### 3.3.3.5 你真的需要 Service Mesh 吗？

正如 NGINX 在其博客上发表的一篇文章名叫 [Do I Need a Service Mesh? ](https://www.nginx.com/blog/do-i-need-a-service-mesh/)【[译文](http://www.servicemesher.com/blog/do-i-need-a-service-mesh/)】 的文章中提到：

> As the complexity of the application increases, service mesh becomes a realistic alternative to implementing capabilities service-by-service.
>
> 随着应用程序复杂性的增加，服务网格将成为实现服务到服务的能力的现实选择。

![image-20200118123407579](/Users/xuel/Library/Application Support/typora-user-images/image-20200118123407579.png)

图片来自：[Do I Need a Service Mesh?](https://www.nginx.com/blog/do-i-need-a-service-mesh/)

随着我们的微服务越来越细分，我们所要管理的服务正在成倍的增长着，Kubernetes 提供了丰富的功能，使得我们可以快速的部署和调度这些服务，同时也提供了我们熟悉的方式来实现那些复杂的功能，但是当临界点到来时，可能就是我们真正要去考虑使用 Service Mesh 的时候了。

## 3.4 Sidecar 模式

Sidecar 模式是 Istio 服务网格采用的模式，在服务网格出现之前该模式就一直存在，尤其是当微服务出现后开始盛行，本文讲解 Sidecar 模式。



**什么是Sidecar模式 ** 

将应用程序的功能划分为单独的进程可以被视为 **Sidecar 模式**。Sidecar 设计模式允许你为应用程序添加许多功能，而无需额外第三方组件的配置和代码的修改。

就如 Sidecar 连接着摩托车一样，类似地在软件架构中， Sidecar 应用是连接到父应用并且为其扩展或者增强功能。Sidecar 应用与主应用程序松散耦合。

让我用一个例子解释一下。想象一下假如你有6个微服务相互通信以确定一个包裹的成本。

每个微服务都需要具有可观察性、监控、日志记录、配置、断路器等功能。所有这些功能都是根据一些行业标准的第三方库在每个微服务中实现的。

但再想一想，这不是多余吗？它不会增加应用程序的整体复杂性吗？如果你的应用程序是用不同的语言编写时会发生什么——如何合并那些特定用于 .Net、Java、Python 等语言的第三方库。



**使用 Sidecar 模式的优势**

- 通过抽象出与功能相关的共同基础设施到一个不同层降低了微服务代码的复杂度。
- 因为你不再需要编写相同的第三方组件配置文件和代码，所以能够降低微服务架构中的代码重复度。
- 降低应用程序代码和底层平台的耦合度。



**Sidecar 模式如何工作**

Sidecar 是容器应用模式的一种，也是在 Service Mesh 中发扬光大的一种模式，详见 [Service Mesh 架构解析](http://www.servicemesher.com/blog/service-mesh-architectures/)，其中详细描述了**节点代理**和 **Sidecar** 模式的 Service Mesh 架构。

使用 Sidecar 模式部署服务网格时，无需在节点上运行代理（因此您不需要基础结构的协作），但是集群中将运行多个相同的 Sidecar 副本。从另一个角度看：我可以为一组微服务部署到一个服务网格中，你也可以部署一个有特定实现的服务网格。在 Sidecar 部署方式中，你会为每个应用的容器部署一个伴生容器。Sidecar 接管进出应用容器的所有流量。在 Kubernetes 的 Pod 中，在原有的应用容器旁边运行一个 Sidecar 容器，可以理解为两个容器共享存储、网络等资源，可以广义的将这个注入了 Sidecar 容器的 Pod 理解为一台主机，两个容器共享主机资源。

例如下图 [SOFAMesh & SOFA MOSN—基于 Istio 构建的用于应对大规模流量的 Service Mesh 解决方案](https://jimmysong.io/posts/sofamesh-and-mosn-proxy-sidecar-service-mesh-by-ant-financial/)的架构图中描述的，MOSN 作为 Sidecar 的方式和应用运行在同一个 Pod 中，拦截所有进出应用容器的流量，[SOFAMesh](https://github.com/alipay/sofa-mesh) 兼容 Istio，其中使用 Go 语言开发的 [SOFAMosn](https://github.com/alipay/sofa-mosn) 替换了 Envoy。

![image-20200118123643733](/Users/xuel/Library/Application Support/typora-user-images/image-20200118123643733.png)

### 3.4.1 Istio 中的 Sidecar 注入与流量劫持详解

在讲解 Istio 如何将 Envoy 代理注入到应用程序 Pod 中之前，我们需要先了解以下几个概念：

- Sidecar 模式：容器应用模式之一，Service Mesh 架构的一种实现方式。
- Init 容器：Pod 中的一种专用的容器，在应用程序容器启动之前运行，用来包含一些应用镜像中不存在的实用工具或安装脚本。
- iptables：流量劫持是通过 iptables 转发实现的。

#### 3.4.1.1 Init 容器

Init 容器是一种专用容器，它在应用程序容器启动之前运行，用来包含一些应用镜像中不存在的实用工具或安装脚本。

一个 Pod 中可以指定多个 Init 容器，如果指定了多个，那么 Init 容器将会按顺序依次运行。只有当前面的 Init 容器必须运行成功后，才可以运行下一个 Init 容器。当所有的 Init 容器运行完成后，Kubernetes 才初始化 Pod 和运行应用容器。

Init 容器使用 Linux Namespace，所以相对应用程序容器来说具有不同的文件系统视图。因此，它们能够具有访问 Secret 的权限，而应用程序容器则不能。

在 Pod 启动过程中，Init 容器会按顺序在网络和数据卷初始化之后启动。每个容器必须在下一个容器启动之前成功退出。如果由于运行时或失败退出，将导致容器启动失败，它会根据 Pod 的 `restartPolicy` 指定的策略进行重试。然而，如果 Pod 的 `restartPolicy` 设置为 Always，Init 容器失败时会使用 `RestartPolicy` 策略。

在所有的 Init 容器没有成功之前，Pod 将不会变成 `Ready` 状态。Init 容器的端口将不会在 Service 中进行聚集。 正在初始化中的 Pod 处于 `Pending` 状态，但应该会将 `Initializing` 状态设置为 true。Init 容器运行完成以后就会自动终止。

关于 Init 容器的详细信息请参考 [Init 容器 - Kubernetes 中文指南/云原生应用架构实践手册](https://jimmysong.io/kubernetes-handbook/concepts/init-containers.html)。

#### 3.4.1.2 Sidecar 注入示例分析

我们看下 Istio 官方示例 `bookinfo` 中 `productpage` 的 YAML 配置，关于 `bookinfo` 应用的详细 YAML 配置请参考 [bookinfo.yaml](https://github.com/rootsongjc/kubernetes-vagrant-centos-cluster/blob/master/yaml/istio-bookinfo/bookinfo.yaml)。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: productpage
  labels:
    app: productpage
spec:
  ports:
  - port: 9080
    name: http
  selector:
    app: productpage
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: productpage-v1
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: productpage
        version: v1
    spec:
      containers:
      - name: productpage
        image: istio/examples-bookinfo-productpage-v1:1.8.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 9080
```

再查看下 `productpage` 容器的 [Dockerfile](https://github.com/istio/istio/blob/master/samples/bookinfo/src/productpage/Dockerfile)。

```docker
FROM python:2.7-slim

COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

COPY productpage.py /opt/microservices/
COPY templates /opt/microservices/templates
COPY requirements.txt /opt/microservices/
EXPOSE 9080
WORKDIR /opt/microservices
CMD python productpage.py 9080
```

我们看到 `Dockerfile` 中没有配置 `ENTRYPOINT`，所以 `CMD` 的配置 `python productpage.py 9080` 将作为默认的 `ENTRYPOINT`，记住这一点，再看下注入 sidecar 之后的配置。

```bash
$ istioctl kube-inject -f yaml/istio-bookinfo/bookinfo.yaml
```

我们只截取其中与 `productpage` 相关的 `Service` 和 `Deployment` 配置部分。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: productpage
  labels:
    app: productpage
spec:
  ports:
  - port: 9080
    name: http
  selector:
    app: productpage
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  creationTimestamp: null
  name: productpage-v1
spec:
  replicas: 1
  strategy: {}
  template:
    metadata:
      annotations:
        sidecar.istio.io/status: '{"version":"fde14299e2ae804b95be08e0f2d171d466f47983391c00519bbf01392d9ad6bb","initContainers":["istio-init"],"containers":["istio-proxy"],"volumes":["istio-envoy","istio-certs"],"imagePullSecrets":null}'
      creationTimestamp: null
      labels:
        app: productpage
        version: v1
    spec:
      containers:
      - image: istio/examples-bookinfo-productpage-v1:1.8.0
        imagePullPolicy: IfNotPresent
        name: productpage
        ports:
        - containerPort: 9080
        resources: {}
      - args:
        - proxy
        - sidecar
        - --configPath
        - /etc/istio/proxy
        - --binaryPath
        - /usr/local/bin/envoy
        - --serviceCluster
        - productpage
        - --drainDuration
        - 45s
        - --parentShutdownDuration
        - 1m0s
        - --discoveryAddress
        - istio-pilot.istio-system:15007
        - --discoveryRefreshDelay
        - 1s
        - --zipkinAddress
        - zipkin.istio-system:9411
        - --connectTimeout
        - 10s
        - --statsdUdpAddress
        - istio-statsd-prom-bridge.istio-system:9125
        - --proxyAdminPort
        - "15000"
        - --controlPlaneAuthPolicy
        - NONE
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: INSTANCE_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: ISTIO_META_POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: ISTIO_META_INTERCEPTION_MODE
          value: REDIRECT
        image: jimmysong/istio-release-proxyv2:1.0.0
        imagePullPolicy: IfNotPresent
        name: istio-proxy
        resources:
          requests:
            cpu: 10m
        securityContext:
          privileged: false
          readOnlyRootFilesystem: true
          runAsUser: 1337
        volumeMounts:
        - mountPath: /etc/istio/proxy
          name: istio-envoy
        - mountPath: /etc/certs/
          name: istio-certs
          readOnly: true
      initContainers:
      - args:
        - -p
        - "15001"
        - -u
        - "1337"
        - -m
        - REDIRECT
        - -i
        - '*'
        - -x
        - ""
        - -b
        - 9080,
        - -d
        - ""
        image: jimmysong/istio-release-proxy_init:1.0.0
        imagePullPolicy: IfNotPresent
        name: istio-init
        resources: {}
        securityContext:
          capabilities:
            add:
            - NET_ADMIN
          privileged: true
      volumes:
      - emptyDir:
          medium: Memory
        name: istio-envoy
      - name: istio-certs
        secret:
          optional: true
          secretName: istio.default
status: {}
```

我们看到 Service 的配置没有变化，所有的变化都在 `Deployment` 里，Istio 给应用 Pod 注入的配置主要包括：

- Init 容器 `istio-init`：用于给 Sidecar 容器即 Envoy 代理做初始化，设置 iptables 端口转发
- Envoy sidecar 容器 `istio-proxy`：运行 Envoy 代理

接下来将分别解析下这两个容器。

##### 3.4.1.2.1 Init 容器解析

Istio 在 Pod 中注入的 Init 容器名为 `istio-init`，我们在上面 Istio 注入完成后的 YAML 文件中看到了该容器的启动参数：

```bash
-p 15001 -u 1337 -m REDIRECT -i '*' -x "" -b 9080 -d ""
```

我们再检查下该容器的 [Dockerfile](https://github.com/istio/istio/blob/master/pilot/docker/Dockerfile.proxy_init) 看看 `ENTRYPOINT` 是什么以确定启动时执行的命令。

```docker
FROM ubuntu:xenial
RUN apt-get update && apt-get install -y \
    iproute2 \
    iptables \
 && rm -rf /var/lib/apt/lists/*

ADD istio-iptables.sh /usr/local/bin/
ENTRYPOINT ["/usr/local/bin/istio-iptables.sh"]
```

我们看到 `istio-init` 容器的入口是 `/usr/local/bin/istio-iptables.sh` 脚本，再按图索骥看看这个脚本里到底写的什么，该脚本的位置在 Istio 源码仓库的 [tools/deb/istio-iptables.sh](https://github.com/istio/istio/blob/master/tools/deb/istio-iptables.sh)，一共 300 多行，就不贴在这里了。下面我们就来解析下这个启动脚本。

##### 4.3.1.2.2 Init 容器启动入口

Init 容器的启动入口是 `/usr/local/bin/istio-iptables.sh` 脚本，该脚本的用法如下：

```bash
$ istio-iptables.sh -p PORT -u UID -g GID [-m mode] [-b ports] [-d ports] [-i CIDR] [-x CIDR] [-h]
  -p: 指定重定向所有 TCP 流量的 Envoy 端口（默认为 $ENVOY_PORT = 15001）
  -u: 指定未应用重定向的用户的 UID。通常，这是代理容器的 UID（默认为 $ENVOY_USER 的 uid，istio_proxy 的 uid 或 1337）
  -g: 指定未应用重定向的用户的 GID。（与 -u param 相同的默认值）
  -m: 指定入站连接重定向到 Envoy 的模式，“REDIRECT” 或 “TPROXY”（默认为 $ISTIO_INBOUND_INTERCEPTION_MODE)
  -b: 逗号分隔的入站端口列表，其流量将重定向到 Envoy（可选）。使用通配符 “*” 表示重定向所有端口。为空时表示禁用所有入站重定向（默认为 $ISTIO_INBOUND_PORTS）
  -d: 指定要从重定向到 Envoy 中排除（可选）的入站端口列表，以逗号格式分隔。使用通配符“*” 表示重定向所有入站流量（默认为 $ISTIO_LOCAL_EXCLUDE_PORTS）
  -i: 指定重定向到 Envoy（可选）的 IP 地址范围，以逗号分隔的 CIDR 格式列表。使用通配符 “*” 表示重定向所有出站流量。空列表将禁用所有出站重定向（默认为 $ISTIO_SERVICE_CIDR）
  -x: 指定将从重定向中排除的 IP 地址范围，以逗号分隔的 CIDR 格式列表。使用通配符 “*” 表示重定向所有出站流量（默认为 $ISTIO_SERVICE_EXCLUDE_CIDR）。

环境变量位于 $ISTIO_SIDECAR_CONFIG（默认在：/var/lib/istio/envoy/sidecar.env）
```

通过查看该脚本你将看到，以上传入的参数都会重新组装成 [`iptables` 命令](https://wangchujiang.com/linux-command/c/iptables.html)的参数。

再参考 `istio-init` 容器的启动参数，完整的启动命令如下：

```bash
$ /usr/local/bin/istio-iptables.sh -p 15001 -u 1337 -m REDIRECT -i '*' -x "" -b 9080 -d ""
```

该容器存在的意义就是让 Envoy 代理可以拦截所有的进出 Pod 的流量，即将入站流量重定向到 Sidecar，再拦截应用容器的出站流量经过 Sidecar 处理后再出站。

**命令解析**

这条启动命令的作用是：

- 将应用容器的所有流量都转发到 Envoy 的 15001 端口。
- 使用 `istio-proxy` 用户身份运行， UID 为 1337，即 Envoy 所处的用户空间，这也是 `istio-proxy`容器默认使用的用户，见 YAML 配置中的 `runAsUser` 字段。
- 使用默认的 `REDIRECT` 模式来重定向流量。
- 将所有出站流量都重定向到 Envoy 代理。
- 将所有访问 9080 端口（即应用容器 `productpage` 的端口）的流量重定向到 Envoy 代理。

因为 Init 容器初始化完毕后就会自动终止，因为我们无法登陆到容器中查看 iptables 信息，但是 Init 容器初始化结果会保留到应用容器和 Sidecar 容器中。

##### 3.4.1.2.3 istio-proxy 容器解析

为了查看 iptables 配置，我们需要登陆到 Sidecar 容器中使用 root 用户来查看，因为 `kubectl` 无法使用特权模式来远程操作 docker 容器，所以我们需要登陆到 `productpage` Pod 所在的主机上使用 `docker`命令登陆容器中查看。

查看 `productpage` Pod 所在的主机。

```bash
$ kubectl -n default get pod -l app=productpage -o wide
NAME                              READY     STATUS    RESTARTS   AGE       IP             NODE
productpage-v1-745ffc55b7-2l2lw   2/2       Running   0          1d        172.33.78.10   node3
```

从输出结果中可以看到该 Pod 运行在 `node3` 上，使用 `vagrant` 命令登陆到 `node3` 主机中并切换为 root 用户。

```bash
$ vagrant ssh node3
$ sudo -i
```

查看 iptables 配置，列出 NAT（网络地址转换）表的所有规则，因为在 Init 容器启动的时候选择给 `istio-iptables.sh` 传递的参数中指定将入站流量重定向到 Envoy 的模式为 “REDIRECT”，因此在 iptables 中将只有 NAT 表的规格配置，如果选择 `TPROXY` 还会有 `mangle` 表配置。`iptables` 命令的详细用法请参考 [iptables](https://wangchujiang.com/linux-command/c/iptables.html)，规则配置请参考 [iptables 规则配置](http://www.zsythink.net/archives/1517)。

#### 3.4.1.3 理解 iptables

`iptables` 是 Linux 内核中的防火墙软件 netfilter 的管理工具，位于用户空间，同时也是 netfilter 的一部分。Netfilter 位于内核空间，不仅有网络地址转换的功能，也具备数据包内容修改、以及数据包过滤等防火墙功能。

在了解 Init 容器初始化的 iptables 之前，我们先来了解下 iptables 和规则配置。

下图展示了 iptables 调用链。

![iptables è°ç¨é¾](https://www.servicemesher.com/istio-handbook/images/0069RVTdly1fv5hukl647j30k6145gnt.jpg)

##### 3.4.1.3.1 iptables 中的表

Init 容器中使用的的 iptables 版本是 `v1.6.0`，共包含 5 张表：

1. `raw` 用于配置数据包，`raw` 中的数据包不会被系统跟踪。
2. `filter` 是用于存放所有与防火墙相关操作的默认表。
3. `nat` 用于 [网络地址转换](https://en.wikipedia.org/wiki/Network_address_translation)（例如：端口转发）。
4. `mangle` 用于对特定数据包的修改（参考[损坏数据包](https://en.wikipedia.org/wiki/Mangled_packet)）。
5. `security` 用于[强制访问控制](https://wiki.archlinux.org/index.php/Security#Mandatory_access_control) 网络规则。

**注**：在本示例中只用到了 `nat` 表。

不同的表中的具有的链类型如下表所示：

| 规则名称    | raw  | filter | nat  | mangle | security |
| ----------- | ---- | ------ | ---- | ------ | -------- |
| PREROUTING  | ✓    |        | ✓    | ✓      |          |
| INPUT       |      | ✓      | ✓    | ✓      | ✓        |
| OUTPUT      |      | ✓      | ✓    | ✓      | ✓        |
| POSTROUTING |      |        | ✓    | ✓      |          |
| FORWARD     | ✓    | ✓      |      | ✓      | ✓        |

下图是 iptables 的调用链顺序。

![iptables è°ç¨é¾](https://www.servicemesher.com/istio-handbook/images/0069RVTdgy1fv5dq2bptdj31110begnl.jpg)

##### 3.4.1.3.2 iptables 命令

`iptables` 命令的主要用途是修改这些表中的规则。`iptables` 命令格式如下：

```bash
$ iptables [-t 表名] 命令选项［链名]［条件匹配］[-j 目标动作或跳转］
```

Init 容器中的 `/istio-iptables.sh` 启动入口脚本就是执行 iptables 初始化的。

##### 3.4.1.3.3 理解 iptables 规则

查看 `istio-proxy` 容器中的默认的 iptables 规则，默认查看的是 filter 表中的规则。

```bash
$ iptables -L -v
Chain INPUT (policy ACCEPT 350K packets, 63M bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 18M packets, 1916M bytes)
 pkts bytes target     prot opt in     out     source               destination
```

我们看到三个默认的链，分别是 INPUT、FORWARD 和 OUTPUT，每个链中的第一行输出表示链名称（在本例中为INPUT/FORWARD/OUTPUT），后跟默认策略（ACCEPT）。

下图是 iptables 的建议结构图，流量在经过 INPUT 链之后就进入了上层协议栈，比如

![image-20200118151703493](/Users/xuel/Library/Application Support/typora-user-images/image-20200118151703493.png)

图片来自[常见iptables使用规则场景整理](https://unixso.com/Linux/iptables.html)

每条链中都可以添加多条规则，规则是按照顺序从前到后执行的。我们来看下规则的表头定义。

- **pkts**：处理过的匹配的报文数量
- **bytes**：累计处理的报文大小（字节数）
- **target**：如果报文与规则匹配，指定目标就会被执行。
- **prot**：协议，例如 `tdp`、`udp`、`icmp` 和 `all`。
- **opt**：很少使用，这一列用于显示 IP 选项。
- **in**：入站网卡。
- **out**：出站网卡。
- **source**：流量的源 IP 地址或子网，后者是 `anywhere`。
- **destination**：流量的目的地 IP 地址或子网，或者是 `anywhere`。

还有一列没有表头，显示在最后，表示规则的选项，作为规则的扩展匹配条件，用来补充前面的几列中的配置。`prot`、`opt`、`in`、`out`、`source` 和 `destination` 和显示在 `destination` 后面的没有表头的一列扩展条件共同组成匹配规则。当流量匹配这些规则后就会执行 `target`。

关于 iptables 规则请参考[常见iptables使用规则场景整理](https://unixso.com/Linux/iptables.html)。

**target 支持的类型**

`target` 类型包括 `ACCEPT`、`REJECT`、`DROP`、`LOG`、`SNAT`、`MASQUERADE`、`DNAT`、`REDIRECT`、`RETURN` 或者跳转到其他规则等。只要执行到某一条链中只有按照顺序有一条规则匹配后就可以确定报文的去向了，除了 `RETURN` 类型，类似编程语言中的 `return` 语句，返回到它的调用点，继续执行下一条规则。`target` 支持的配置详解请参考 [iptables 详解（1）：iptables 概念](http://www.zsythink.net/archives/1199)。

从输出结果中可以看到 Init 容器没有在 iptables 的默认链路中创建任何规则，而是创建了新的链路。

#### 3.4.1.4 查看 iptables nat 表中注入的规则

Init 容器通过向 iptables nat 表中注入转发规则来劫持流量的，下图显示的是 productpage 服务中的 iptables 流量劫持的详细过程。

![image-20200118151837963](/Users/xuel/Library/Application Support/typora-user-images/image-20200118151837963.png)

Init 容器启动时命令行参数中指定了 `REDIRECT` 模式，因此只创建了 NAT 表规则，接下来我们查看下 NAT 表中创建的规则，这是全文中的**重点部分**，前面讲了那么多都是为它做铺垫的。下面是查看 nat 表中的规则，其中链的名字中包含 `ISTIO` 前缀的是由 Init 容器注入的，规则匹配是根据下面显示的顺序来执行的，其中会有多次跳转。

```bash
# 查看 NAT 表中规则配置的详细信息
$ iptables -t nat -L -v
# PREROUTING 链：用于目标地址转换（DNAT），将所有入站 TCP 流量跳转到 ISTIO_INBOUND 链上
Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
    2   120 ISTIO_INBOUND  tcp  --  any    any     anywhere             anywhere

# INPUT 链：处理输入数据包，非 TCP 流量将继续 OUTPUT 链
Chain INPUT (policy ACCEPT 2 packets, 120 bytes)
 pkts bytes target     prot opt in     out     source               destination

# OUTPUT 链：将所有出站数据包跳转到 ISTIO_OUTPUT 链上
Chain OUTPUT (policy ACCEPT 41146 packets, 3845K bytes)
 pkts bytes target     prot opt in     out     source               destination
   93  5580 ISTIO_OUTPUT  tcp  --  any    any     anywhere             anywhere

# POSTROUTING 链：所有数据包流出网卡时都要先进入POSTROUTING 链，内核根据数据包目的地判断是否需要转发出去，我们看到此处未做任何处理
Chain POSTROUTING (policy ACCEPT 41199 packets, 3848K bytes)
 pkts bytes target     prot opt in     out     source               destination

# ISTIO_INBOUND 链：将所有目的地为 9080 端口的入站流量重定向到 ISTIO_IN_REDIRECT 链上
Chain ISTIO_INBOUND (1 references)
 pkts bytes target     prot opt in     out     source               destination
    2   120 ISTIO_IN_REDIRECT  tcp  --  any    any     anywhere             anywhere             tcp dpt:9080

# ISTIO_IN_REDIRECT 链：将所有的入站流量跳转到本地的 15001 端口，至此成功的拦截了流量到 Envoy 
Chain ISTIO_IN_REDIRECT (1 references)
 pkts bytes target     prot opt in     out     source               destination
    2   120 REDIRECT   tcp  --  any    any     anywhere             anywhere             redir ports 15001

# ISTIO_OUTPUT 链：选择需要重定向到 Envoy（即本地） 的出站流量，所有非 localhost 的流量全部转发到 ISTIO_REDIRECT。为了避免流量在该 Pod 中无限循环，所有到 istio-proxy 用户空间的流量都返回到它的调用点中的下一条规则，本例中即 OUTPUT 链，因为跳出 ISTIO_OUTPUT 规则之后就进入下一条链 POSTROUTING。如果目的地非 localhost 就跳转到 ISTIO_REDIRECT；如果流量是来自 istio-proxy 用户空间的，那么就跳出该链，返回它的调用链继续执行下一条规则（OUPT 的下一条规则，无需对流量进行处理）；所有的非 istio-proxy 用户空间的目的地是 localhost 的流量就跳转到 ISTIO_REDIRECT
Chain ISTIO_OUTPUT (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 ISTIO_REDIRECT  all  --  any    lo      anywhere            !localhost
   40  2400 RETURN     all  --  any    any     anywhere             anywhere             owner UID match istio-proxy
    0     0 RETURN     all  --  any    any     anywhere             anywhere             owner GID match istio-proxy    
    0     0 RETURN     all  --  any    any     anywhere             localhost
   53  3180 ISTIO_REDIRECT  all  --  any    any     anywhere             anywhere

# ISTIO_REDIRECT 链：将所有流量重定向到 Envoy（即本地） 的 15001 端口
Chain ISTIO_REDIRECT (2 references)
 pkts bytes target     prot opt in     out     source               destination
   53  3180 REDIRECT   tcp  --  any    any     anywhere             anywhere             redir ports 15001
```

`iptables` 显示的链的顺序，即流量规则匹配的顺序。其中要特别注意 `ISTIO_OUTPUT` 链中的规则配置。为了避免流量一直在 Pod 中无限循环，所有到 istio-proxy 用户空间的流量都返回到它的调用点中的下一条规则，本例中即 OUTPUT 链，因为跳出 `ISTIO_OUTPUT` 规则之后就进入下一条链 `POSTROUTING`。

`ISTIO_OUTPUT` 链规则匹配的详细过程如下：

- 如果目的地非 localhost 就跳转到 ISTIO_REDIRECT 链
- 所有来自 istio-proxy 用户空间的非 localhost 流量跳转到它的调用点 `OUTPUT` 继续执行 `OUTPUT` 链的下一条规则，因为 `OUTPUT` 链中没有下一条规则了，所以会继续执行 `POSTROUTING` 链然后跳出 iptables，直接访问目的地
- 如果流量不是来自 istio-proxy 用户空间，又是对 localhost 的访问，那么就跳出 iptables，直接访问目的地
- 其它所有情况都跳转到 `ISTIO_REDIRECT` 链

其实在最后这条规则前还可以增加 IP 地址过滤，让某些 IP 地址段不通过 Envoy 代理。

[![istio sidecar iptables 注入](https://www.servicemesher.com/istio-handbook/images/006tNbRwly1fv92uvxu4dj31320giq6u.jpg)](https://www.servicemesher.com/istio-handbook/images/006tNbRwly1fv92uvxu4dj31320giq6u.jpg)图片 - istio sidecar iptables 注入

以上 iptables 规则都是 Init 容器启动的时使用 [istio-iptables.sh](https://github.com/istio/istio/blob/master/tools/deb/istio-iptables.sh) 脚本生成的，详细过程可以查看该脚本。

#### 3.4.1.5 查看 Envoy 运行状态

首先查看 `proxyv2` 镜像的 [Dockerfile](https://github.com/istio/istio/blob/master/pilot/docker/Dockerfile.proxyv2)。

```docker
FROM istionightly/base_debug
ARG proxy_version
ARG istio_version

# 安装 Envoy
ADD envoy /usr/local/bin/envoy

# 使用环境变量的方式明文指定 proxy 的版本/功能
ENV ISTIO_META_ISTIO_PROXY_VERSION "1.1.0"
# 使用环境变量的方式明文指定 proxy 明确的 sha，用于指定版本的配置和调试
ENV ISTIO_META_ISTIO_PROXY_SHA $proxy_version
# 环境变量，指定明确的构建号，用于调试
ENV ISTIO_META_ISTIO_VERSION $istio_version

ADD pilot-agent /usr/local/bin/pilot-agent

ADD envoy_pilot.yaml.tmpl /etc/istio/proxy/envoy_pilot.yaml.tmpl
ADD envoy_policy.yaml.tmpl /etc/istio/proxy/envoy_policy.yaml.tmpl
ADD envoy_telemetry.yaml.tmpl /etc/istio/proxy/envoy_telemetry.yaml.tmpl
ADD istio-iptables.sh /usr/local/bin/istio-iptables.sh

COPY envoy_bootstrap_v2.json /var/lib/istio/envoy/envoy_bootstrap_tmpl.json

RUN chmod 755 /usr/local/bin/envoy /usr/local/bin/pilot-agent

# 将 istio-proxy 用户加入 sudo 权限以允许执行 tcpdump 和其他调试命令
RUN useradd -m --uid 1337 istio-proxy && \
    echo "istio-proxy ALL=NOPASSWD: ALL" >> /etc/sudoers && \
    chown -R istio-proxy /var/lib/istio

# 使用 pilot-agent 来启动 Envoy
ENTRYPOINT ["/usr/local/bin/pilot-agent"]
```

该容器的启动入口是 `pilot-agent` 命令，根据 YAML 配置中传递的参数，详细的启动命令入下：

```bash
/usr/local/bin/pilot-agent proxy sidecar --configPath /etc/istio/proxy --binaryPath /usr/local/bin/envoy --serviceCluster productpage --drainDuration 45s --parentShutdownDuration 1m0s --discoveryAddress istio-pilot.istio-system:15007 --discoveryRefreshDelay 1s --zipkinAddress zipkin.istio-system:9411 --connectTimeout 10s --statsdUdpAddress istio-statsd-prom-bridge.istio-system:9125 --proxyAdminPort 15000 --controlPlaneAuthPolicy NONE
```

主要配置了 Envoy 二进制文件的位置、服务发现地址、服务集群名、监控指标上报地址、Envoy 的管理端口、热重启时间等，详细用法请参考 [Istio官方文档 pilot-agent 的用法](https://istio.io/docs/reference/commands/pilot-agent/)。

`pilot-agent` 是容器中 PID 为 1 的启动进程，它启动时又创建了一个 Envoy 进程，如下：

```bash
/usr/local/bin/envoy -c /etc/istio/proxy/envoy-rev0.json --restart-epoch 0 --drain-time-s 45 --parent-shutdown-time-s 60 --service-cluster productpage --service-node sidecar~172.33.78.10~productpage-v1-745ffc55b7-2l2lw.default~default.svc.cluster.local --max-obj-name-len 189 -l warn --v2-config-only
```

我们分别解释下以上配置的意义。

- `-c /etc/istio/proxy/envoy-rev0.json`：配置文件，支持 `.json`、`.yaml`、`.pb` 和 `.pb_text` 格式，`pilot-agent` 启动的时候读取了容器的环境变量后创建的。
- `--restart-epoch 0`：Envoy 热重启周期，第一次启动默认为 0，每热重启一次该值加 1。
- `--drain-time-s 45`：热重启期间 Envoy 将耗尽连接的时间。
- `--parent-shutdown-time-s 60`： Envoy 在热重启时关闭父进程之前等待的时间。
- `--service-cluster productpage`：Envoy 运行的本地服务集群的名字。
- `--service-node sidecar~172.33.78.10~productpage-v1-745ffc55b7-2l2lw.default~default.svc.cluster.local`：定义 Envoy 运行的本地服务节点名称，其中包含了该 Pod 的名称、IP、DNS 域等信息，根据容器的环境变量拼出来的。
- `-max-obj-name-len 189`：cluster/route_config/listener 中名称字段的最大长度（以字节为单位）
- `-l warn`：日志级别
- `--v2-config-only`：只解析 v2 引导配置文件

详细配置请参考 [Envoy 的命令行选项](http://www.servicemesher.com/envoy/operations/cli.html)。

查看 Envoy 的配置文件 `/etc/istio/proxy/envoy-rev0.json`。

```json
{
  "node": {
    "id": "sidecar~172.33.78.10~productpage-v1-745ffc55b7-2l2lw.default~default.svc.cluster.local",
    "cluster": "productpage",

    "metadata": {
          "INTERCEPTION_MODE": "REDIRECT",
          "ISTIO_PROXY_SHA": "istio-proxy:6166ae7ebac7f630206b2fe4e6767516bf198313",
          "ISTIO_PROXY_VERSION": "1.0.0",
          "ISTIO_VERSION": "1.0.0",
          "POD_NAME": "productpage-v1-745ffc55b7-2l2lw",
      "istio": "sidecar"
    }
  },
  "stats_config": {
    "use_all_default_tags": false
  },
  "admin": {
    "access_log_path": "/dev/stdout",
    "address": {
      "socket_address": {
        "address": "127.0.0.1",
        "port_value": 15000
      }
    }
  },
  "dynamic_resources": {
    "lds_config": {
        "ads": {}
    },
    "cds_config": {
        "ads": {}
    },
    "ads_config": {
      "api_type": "GRPC",
      "refresh_delay": {"seconds": 1, "nanos": 0},
      "grpc_services": [
        {
          "envoy_grpc": {
            "cluster_name": "xds-grpc"
          }
        }
      ]
    }
  },
  "static_resources": {
    "clusters": [
    {
    "name": "xds-grpc",
    "type": "STRICT_DNS",
    "connect_timeout": {"seconds": 10, "nanos": 0},
    "lb_policy": "ROUND_ROBIN",

    "hosts": [
    {
    "socket_address": {"address": "istio-pilot.istio-system", "port_value": 15010}
    }
    ],
    "circuit_breakers": {
        "thresholds": [
      {
        "priority": "default",
        "max_connections": "100000",
        "max_pending_requests": "100000",
        "max_requests": "100000"
      },
      {
        "priority": "high",
        "max_connections": "100000",
        "max_pending_requests": "100000",
        "max_requests": "100000"
      }]
    },
    "upstream_connection_options": {
      "tcp_keepalive": {
        "keepalive_time": 300
      }
    },
    "http2_protocol_options": { }
    }


    ,
      {
        "name": "zipkin",
        "type": "STRICT_DNS",
        "connect_timeout": {
          "seconds": 1
        },
        "lb_policy": "ROUND_ROBIN",
        "hosts": [
          {
            "socket_address": {"address": "zipkin.istio-system", "port_value": 9411}
          }
        ]
      }

    ]
  },

  "tracing": {
    "http": {
      "name": "envoy.zipkin",
      "config": {
        "collector_cluster": "zipkin"
      }
    }
  },


  "stats_sinks": [
    {
      "name": "envoy.statsd",
      "config": {
        "address": {
          "socket_address": {"address": "10.254.109.175", "port_value": 9125}
        }
      }
    }
  ]

}
```

下图是使用 Istio 管理的 bookinfo 示例的访问请求路径图。

[![Istio bookinfo](https://www.servicemesher.com/istio-handbook/images/0069RVTdgy1fv5df9lq1aj317o0o6wia.jpg)](https://www.servicemesher.com/istio-handbook/images/0069RVTdgy1fv5df9lq1aj317o0o6wia.jpg)图片 - Istio bookinfo

图片来自 [Istio 官方网站](https://istio.io/zh/docs/examples/bookinfo/)

对照 bookinfo 示例的 productpage 查看建立的连接。在 `productpage-v1-745ffc55b7-2l2lw` Pod 的 `istio-proxy` 容器中使用 root 用户查看打开的端口。

```bash
$ lsof -i
COMMAND PID        USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
envoy    11 istio-proxy    9u  IPv4  73951      0t0  TCP localhost:15000 (LISTEN) # Envoy admin 端口
envoy    11 istio-proxy   17u  IPv4  74320      0t0  TCP productpage-v1-745ffc55b7-2l2lw:46862->istio-pilot.istio-system.svc.cluster.local:15010 (ESTABLISHED) # 15010：istio-pilot 的 grcp-xds 端口
envoy    11 istio-proxy   18u  IPv4  73986      0t0  UDP productpage-v1-745ffc55b7-2l2lw:44332->istio-statsd-prom-bridge.istio-system.svc.cluster.local:9125 # 给 Promethues 发送 metric 的端口
envoy    11 istio-proxy   52u  IPv4  74599      0t0  TCP *:15001 (LISTEN) # Envoy 的监听端口
envoy    11 istio-proxy   53u  IPv4  74600      0t0  UDP productpage-v1-745ffc55b7-2l2lw:48011->istio-statsd-prom-bridge.istio-system.svc.cluster.local:9125 # 给 Promethues 发送 metric 端口
envoy    11 istio-proxy   54u  IPv4 338551      0t0  TCP productpage-v1-745ffc55b7-2l2lw:15001->172.17.8.102:52670 (ESTABLISHED) # 52670：Ingress gateway 端口
envoy    11 istio-proxy   55u  IPv4 338364      0t0  TCP productpage-v1-745ffc55b7-2l2lw:44046->172.33.78.9:9091 (ESTABLISHED) # 9091：istio-telemetry 服务的 grpc-mixer 端口
envoy    11 istio-proxy   56u  IPv4 338473      0t0  TCP productpage-v1-745ffc55b7-2l2lw:47210->zipkin.istio-system.svc.cluster.local:9411 (ESTABLISHED) # 9411: zipkin 端口
envoy    11 istio-proxy   58u  IPv4 338383      0t0  TCP productpage-v1-745ffc55b7-2l2lw:41564->172.33.84.8:9080 (ESTABLISHED) # 9080：details-v1 的 http 端口
envoy    11 istio-proxy   59u  IPv4 338390      0t0  TCP productpage-v1-745ffc55b7-2l2lw:54410->172.33.78.5:9080 (ESTABLISHED) # 9080：reviews-v2 的 http 端口
envoy    11 istio-proxy   60u  IPv4 338411      0t0  TCP productpage-v1-745ffc55b7-2l2lw:35200->172.33.84.5:9091 (ESTABLISHED) # 9091:istio-telemetry 服务的 grpc-mixer 端口
envoy    11 istio-proxy   62u  IPv4 338497      0t0  TCP productpage-v1-745ffc55b7-2l2lw:34402->172.33.84.9:9080 (ESTABLISHED) # reviews-v1 的 http 端口
envoy    11 istio-proxy   63u  IPv4 338525      0t0  TCP productpage-v1-745ffc55b7-2l2lw:50592->172.33.71.5:9080 (ESTABLISHED) # reviews-v3 的 http 端口
```

从输出经过上可以验证 Sidecar 是如何接管流量和与 istio-pilot 通信，及向 Mixer 做遥测数据汇聚的。感兴趣的读者可以再去看看其他几个服务的 istio-proxy 容器中的 iptables 和端口信息。

### 3.4.2 Sidecar 的自动注入过程详解

在 [Sidecar 注入与流量劫持详解](https://www.servicemesher.com/istio-handbook/concepts-and-principle/sidecar-injection-deep-dive.html)中我只是简单介绍了 Sidecar 注入的步骤，但是没有涉及到具体的 Sidecar 注入流程与细节，这一篇将带大家了解 Istio 为数据平面自动注入 Sidecar 的详细过程。

#### 3.4.2.1 Sidecar 注入过程

如 Istio 官方文档中对 [Istio sidecar 注入](https://istio.io/zh/docs/setup/kubernetes/additional-setup/sidecar-injection/)的描述，你可以使用 istioctl 命令手动注入 Sidecar，也可以为 Kubernetes 集群自动开启 sidecar 注入，这主要用到了 [Kubernetes 的准入控制器](https://jimmysong.io/kubernetes-handbook/concepts/admission-controller.html)中的 webhook，参考 Istio 官网中对 [Istio sidecar 注入](https://istio.io/zh/docs/setup/kubernetes/additional-setup/sidecar-injection/)的描述。

[![Sidecar 注入流程图](https://www.servicemesher.com/istio-handbook/images/006tKfTcly1g0bvoxmfvuj311i0fy0vh.jpg)](https://www.servicemesher.com/istio-handbook/images/006tKfTcly1g0bvoxmfvuj311i0fy0vh.jpg)图片 - Sidecar 注入流程图

### 手动注入 sidecar 与自动注入 sidecar 的区别

不论是手动注入还是自动注入，sidecar 的注入过程都需要遵循如下步骤：

1. Kubernetes 需要了解待注入的 sidecar 所连接的 Istio 集群及其配置；
2. Kubernetes 需要了解待注入的 sidecar 容器本身的配置，如镜像地址、启动参数等；
3. Kubernetes 根据 sidecar 注入模板和以上配置填充 sidecar 的配置参数，将以上配置注入到应用容器的一侧；

Istio 和 sidecar 配置保存在 `istio` 和 `istio-sidecar-injector` 这两个 ConfigMap 中，其中包含了 Go template，所谓自动 sidecar 注入就是将生成 Pod 配置从应用 YAML 文件期间转移到 [mutable webhook](https://jimmysong.io/kubernetes-handbook/concepts/admission-controller.html)中。

## 3.5 Istio CNI Plugin

### 3.5.1 设计目标

当前实现将用户 pod 流量转发到 proxy 的默认方式是使用 privileged 权限的 istio-init 这个 init container 来做的（运行脚本写入 iptables），Istio CNI 插件的主要设计目标是消除这个 privileged 权限的 init container，换成利用 Kubernetes CNI 机制来实现相同功能的替代方案。

### 3.5.2 原理

- Istio CNI Plugin 不是 istio 提出类似 Kubernetes CNI 的插件扩展机制，而是 Kubernetes CNI 的一个具体实现
- Kubernetes CNI 插件是一条链，在创建和销毁pod的时候会调用链上所有插件来安装和卸载容器的网络，istio CNI Plugin 即为 CNI 插件的一个实现，相当于在创建销毁pod这些hook点来针对istio的pod做网络配置：写入iptables，让该 pod 所在的 network namespace 的网络流量转发到 proxy 进程
- 当然也就要求集群启用 CNI，kubelet 启动参数: `--network-plugin=cni` （该参数只有两个可选项：`kubenet`, `cni`）

### 3.5.3 实现方式

- 运行一个名为 istio-cni-node 的 daemonset 运行在每个节点，用于安装 istio CNI 插件
- 该 CNI 插件负责写入 iptables 规则，让用户 pod 所在 netns 的流量都转发到这个 pod 中 proxy 的进程
- 当启用 istio cni 后，sidecar 的自动注入或`istioctl kube-inject`将不再注入 initContainers (istio-init)

### 3.5.4 istio-cni-node 工作流程

- 复制 Istio CNI 插件二进制程序到CNI的bin目录（即kubelet启动参数`--cni-bin-dir`指定的路径，默认是`/opt/cni/bin`）
- 使用istio-cni-node自己的ServiceAccount信息为CNI插件生成kubeconfig，让插件能与apiserver通信(ServiceAccount信息会被自动挂载到`/var/run/secrets/kubernetes.io/serviceaccount`)
- 生成CNI插件的配置并将其插入CNI配置插件链末尾（CNI的配置文件路径是kubelet启动参数`--cni-conf-dir`所指定的目录，默认是`/etc/cni/net.d`）
- watch CNI 配置(`cni-conf-dir`)，如果检测到被修改就重新改回来
- watch istio-cni-node 自身的配置(configmap)，检测到有修改就重新执行CNI配置生成与下发流程（当前写这篇文章的时候是istio 1.1.1，还没实现此功能）

### 3.5.5 设计提案

- Istio CNI Plugin 提案创建时间：2018-09-28
- Istio CNI Plugin 提案文档存放在：Istio 的 Google Team Drive
  - Istio TeamDrive 地址：https://drive.google.com/corp/drive/u/0/folders/0AIS5p3eW9BCtUk9PVA
  - Istio CNI Plugin 提案文档路径：`Working Groups/Networking/Istio CNI Plugin`
  - 查看文件需要申请权限，申请方法：加入istio-team-drive-access这个google网上论坛group
  - istio-team-drive-access group 地址: https://groups.google.com/forum/#!forum/istio-team-drive-access

# 四 数据平台

## 4.1 数据平面介绍

数据平面由一组以 sidecar 方式部署的智能代理组成。 这些代理可以调节和控制微服务之间所有的网络通信。 数据平面真正触及到对网络数据包的相关操作，是上层控制平面策略的具体执行者。

在服务网格中，数据平面 sidecar 代理主要负责执行如下任务：

- 服务发现：探测所有可用的上游或后端服务实例
- 健康检测：探测上游或后端服务实例是否健康，是否准备好接收网络流量
- 流量路由：将网络请求路由到正确的上游或后端服务
- 负载均衡：在对上游或后端服务进行请求时，选择合适的服务实例接收请求，同时负责处理超时、断路、重试等情况
- 身份验证和授权：对网络请求进行身份验证、权限验证，以决定是否响应以及如何响应，使用 mTLS 或其他机制对链路进行加密等
- 链路追踪：对于每个请求，生成详细的统计信息、日志记录和分布式追踪数据，以便操作人员能够理解调用路径并在出现问题时进行调试

简单来说，数据平面就是负责有条件地转换、转发以及观察进出服务实例的每个网络包。

典型的数据平面实现有：[Linkerd](https://linkerd.io/)、[NGINX](https://www.nginx.com/)、[HAProxy](https://www.haproxy.com/)、[Envoy](https://envoyproxy.github.io/)、[Traefik](https://traefik.io/)

## 4.2 Envoy 中的基本术语

**Host**：能够进行网络通信的实体（在手机或服务器等上的应用程序）。在 Envoy 中主机是指逻辑网络应用程序。只要每台主机都可以独立寻址，一块物理硬件上就运行多个主机。

**Downstream**：下游（downstream）主机连接到 Envoy，发送请求并获得响应。

**Upstream**：上游（upstream）主机获取来自 Envoy 的连接接请求和响应。

**Cluster**: 集群（cluster）是 Envoy 连接到的一组逻辑上相似的上游主机。Envoy 通过[服务发现](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/service_discovery#arch-overview-service-discovery)发现集群中的成员。Envoy 可以通过[主动运行状况检查](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/health_checking#arch-overview-health-checking)来确定集群成员的健康状况。Envoy 如何将请求路由到集群成员由[负载均衡策略](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/load_balancing#arch-overview-load-balancing)确定。

**Mesh**：一组互相协调以提供一致网络拓扑的主机。Envoy mesh 是指一组 Envoy 代理，它们构成了由多种不同服务和应用程序平台组成的分布式系统的消息传递基础。

**运行时配置**：与 Envoy 一起部署的带外实时配置系统。可以在无需重启 Envoy 或更改 Envoy 主配置的情况下，通过更改设置来影响操作。

**Listener**: 监听器（listener）是可以由下游客户端连接的命名网络位置（例如，端口、unix域套接字等）。Envoy 公开一个或多个下游主机连接的监听器。一般是每台主机运行一个 Envoy，使用单进程运行，但是每个进程中可以启动任意数量的 Listener（监听器），目前只监听 TCP，每个监听器都独立配置一定数量的（L3/L4）网络过滤器。Listenter 也可以通过 Listener Discovery Service（**LDS**）动态获取。

**Listener filter**：Listener 使用 listener filter（监听器过滤器）来操作连接的元数据。它的作用是在不更改 Envoy 的核心功能的情况下添加更多的集成功能。Listener filter 的 API 相对简单，因为这些过滤器最终是在新接受的套接字上运行。在链中可以互相衔接以支持更复杂的场景，例如调用速率限制。Envoy 已经包含了多个监听器过滤器。

**Http Route Table**：HTTP 的路由规则，例如请求的域名，Path 符合什么规则，转发给哪个 Cluster。

**Health checking**：健康检查会与SDS服务发现配合使用。但是，即使使用其他服务发现方式，也有相应需要进行主动健康检查的情况。详见 [health checking](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/health_checking)。

## 4.3 Istio sidecar proxy 配置

假如您使用 [kubernetes-vagrant-centos-cluster](https://github.com/rootsongjc/kubernetes-vagrant-centos-cluster) 部署了 Kubernetes 集群并开启了 [Istio Service Mesh](https://istio.io/zh)，再部署 [bookinfo 示例](https://istio.io/zh/docs/examples/bookinfo/)，那么在 `default` 命名空间下有一个名字类似于 `ratings-v1-7c9949d479-dwkr4` 的 Pod，使用下面的命令查看该 Pod 的 Envoy sidecar 的全量配置：

```bash
kubectl -n default exec ratings-v1-7c9949d479-dwkr4 -c istio-proxy curl http://localhost:15000/config_dump > dump-rating.json
```

将 Envoy 的运行时配置 dump 出来之后你将看到一个长 6000 余行的配置文件。关于该配置文件的介绍请参考 [Envoy v2 API 概览](http://www.servicemesher.com/envoy/configuration/overview/v2_overview.html)。

下图展示的是 Enovy 的配置。

[![Envoy 配置](https://www.servicemesher.com/istio-handbook/images/006tNbRwly1fyb74brsd5j30xg0lojvt.jpg)](https://www.servicemesher.com/istio-handbook/images/006tNbRwly1fyb74brsd5j30xg0lojvt.jpg)图片 - Envoy 配置

Istio 会在为 Service Mesh 中的每个 Pod 注入 Sidecar 的时候同时为 Envoy 注入 Bootstrap 配置，其余的配置是通过 Pilot 下发的，注意整个数据平面即 Service Mesh 中的 Envoy 的动态配置应该是相同的。您也可以使用上面的命令检查其他 sidecar 的 Envoy 配置是否跟最上面的那个相同。

使用下面的命令检查 Service Mesh 中的所有有 Sidecar 注入的 Pod 中的 proxy 配置是否同步。

```bash
$ istioctl proxy-status
PROXY                                                 CDS        LDS        EDS               RDS          PILOT                            VERSION
details-v1-876bf485f-sx7df.default                    SYNCED     SYNCED     SYNCED (100%)     SYNCED       istio-pilot-5bf6d97f79-6lz4x     1.0.0
...
```

[istioctl](https://istio.io/zh/docs/reference/commands/istioctl/) 这个命令行工具就像 [kubectl](https://jimmysong.io/kubernetes-handbook/guide/kubectl-cheatsheet.html) 一样有很多神奇的魔法，通过它可以高效的管理 Istio 和 debug。

## 4.4 Envoy proxy 配置详解

Istio envoy sidecar proxy 配置中包含以下四个部分。

- **bootstrap**：Envoy proxy 启动时候加载的静态配置。
- **listeners**：监听器配置，使用 LDS 下发。
- **clusters**：集群配置，静态配置中包括 xds-grpc 和 zipkin 地址，动态配置使用 CDS 下发。
- **routes**：路由配置，静态配置中包括了本地监听的服务的集群信息，其中引用了 cluster，动态配置使用 RDS 下发。

每个部分中都包含静态配置与动态配置，其中 bootstrap 配置又是在集群启动的时候通过 sidecar 启动参数注入的，配置文件在 `/etc/istio/proxy/envoy-rev0.json`。

Enovy 的配置 dump 出来后的结构如下图所示。

[![Envoy 配置](https://www.servicemesher.com/istio-handbook/images/006tNbRwly1fy2x2zk1hhj30ee0h9jtg.jpg)](https://www.servicemesher.com/istio-handbook/images/006tNbRwly1fy2x2zk1hhj30ee0h9jtg.jpg)图片 - Envoy 配置

由于 bootstrap 中的配置是来自 Envoy 启动时加载的静态文件，主要配置了节点信息、tracing、admin 和统计信息收集等信息，这不是本文的重点，大家可以自行研究。

[![bootstrap 配置](https://www.servicemesher.com/istio-handbook/images/006tNbRwly1fy2xid761zj30c70ai0tj.jpg)](https://www.servicemesher.com/istio-handbook/images/006tNbRwly1fy2xid761zj30c70ai0tj.jpg)图片 - bootstrap 配置

上图是 bootstrap 的配置信息。

[Bootstrap](https://www.envoyproxy.io/docs/envoy/latest/api-v2/config/bootstrap/v2/bootstrap.proto.html#envoy-api-msg-config-bootstrap-v2-bootstrap) 是 Envoy 中配置的根本来源，[Bootstrap](https://www.envoyproxy.io/docs/envoy/latest/api-v2/config/bootstrap/v2/bootstrap.proto.html#envoy-api-msg-config-bootstrap-v2-bootstrap) 消息中有一个关键的概念，就是静态和动态资源的之间的区别。例如 [Listener](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/lds.proto.html#envoy-api-msg-listener) 或 [Cluster](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/cds.proto.html#envoy-api-msg-cluster) 这些资源既可以从 [static_resources](https://www.envoyproxy.io/docs/envoy/latest/api-v2/config/bootstrap/v2/bootstrap.proto.html#envoy-api-field-config-bootstrap-v2-bootstrap-static-resources) 静态的获得也可以从 [dynamic_resources](https://www.envoyproxy.io/docs/envoy/latest/api-v2/config/bootstrap/v2/bootstrap.proto.html#envoy-api-field-config-bootstrap-v2-bootstrap-dynamic-resources) 中配置的 [LDS](http://www.servicemesher.com/envoy/configuration/listeners/lds.html#config-listeners-lds) 或 [CDS](http://www.servicemesher.com/envoy/configuration/cluster_manager/cds.html#config-cluster-manager-cds) 之类的 xDS 服务获取。

### Listener

Listener 顾名思义，就是监听器，监听 IP 地址和端口，然后根据策略转发。

**Listener 的特点**

- 每个 Envoy 进程中可以有多个 Listener，Envoy 与 Listener 之间是一对多的关系。
- 每个 Listener 中可以配置一条 filter 链表（filter_chains），Envoy 会根据 filter 顺序执行过滤。
- Listener 可以监听下游的端口，也可以接收来自其他 listener 的数据，形成链式处理。
- filter 是可扩展的。
- 可以静态配置，也可以使用 LDS 动态配置。
- 目前只能监听 TCP，UDP 还未支持。

**Listener 的数据结构**

Listener 的数据结构如下，除了 `name`、`address` 和 `filter_chains` 为必须配置之外，其他都为可选的。

```json
{
  "name": "...",
  "address": "{...}",
  "filter_chains": [],
  "use_original_dst": "{...}",
  "per_connection_buffer_limit_bytes": "{...}",
  "metadata": "{...}",
  "drain_type": "...",
  "listener_filters": [],
  "transparent": "{...}",
  "freebind": "{...}",
  "socket_options": [],
  "tcp_fast_open_queue_length": "{...}",
  "bugfix_reverse_write_filter_order": "{...}"
}
```

下面是关于上述数据结构中的常用配置解析。

- **name**：该 listener 的 UUID，唯一限定名，默认60个字符，例如 `10.254.74.159_15011`，可以使用命令参数指定长度限制。

- **address**：监听的逻辑/物理地址和端口号，例如

  ```json
  "address": {
         "socket_address": {
          "address": "10.254.74.159",
          "port_value": 15011
         }
  }
  ```

- **filter_chains**：这是一个列表，Envoy 中内置了一些通用的 filter，每种 filter 都有特定的数据结构，Enovy 会根据该配置顺序执行 filter。Envoy 中内置的 filter 有：[envoy.client_ssl_auth](https://www.envoyproxy.io/docs/envoy/v1.8.0/configuration/network_filters/client_ssl_auth_filter#config-network-filters-client-ssl-auth)、[envoy.echo](https://www.envoyproxy.io/docs/envoy/v1.8.0/configuration/network_filters/echo_filter#config-network-filters-echo)、[enovy.http_connection_manager](https://www.envoyproxy.io/docs/envoy/v1.8.0/configuration/http_conn_man/http_conn_man#config-http-conn-man)、[envoy.mongo_proxy](https://www.envoyproxy.io/docs/envoy/v1.8.0/configuration/network_filters/mongo_proxy_filter#config-network-filters-mongo-proxy)、[envoy.rate_limit](https://www.envoyproxy.io/docs/envoy/v1.8.0/configuration/network_filters/rate_limit_filter#config-network-filters-rate-limit)、[enovy.redis_proxy](https://www.envoyproxy.io/docs/envoy/v1.8.0/configuration/network_filters/redis_proxy_filter#config-network-filters-redis-proxy)、[envoy.tcp_proxy](https://www.envoyproxy.io/docs/envoy/v1.8.0/configuration/network_filters/tcp_proxy_filter#config-network-filters-tcp-proxy)、[http_filters](https://www.envoyproxy.io/docs/envoy/v1.8.0/intro/arch_overview/http_filters)、[thrift_filters](https://www.envoyproxy.io/docs/envoy/v1.8.0/configuration/thrift_filters/thrift_filters)等。这些 filter 可以单独使用也可以组合使用，还可以自定义扩展，例如使用 Istio 中的 [EnvoyFilter 配置](https://istio.io/zh/docs/reference/config/istio.networking.v1alpha3/#envoyfilter)。

- **use_original_dst**：这是一个布尔值，如果使用 iptables 重定向连接，则代理接收的端口可能与[原始目的地址](http://www.servicemesher.com/envoy/configuration/listener_filters/original_dst_filter.html)的端口不一样。当此标志设置为 true 时，Listener 将重定向的连接切换到与原始目的地址关联的 Listener。如果没有与原始目的地址关联的 Listener，则连接由接收它的 Listener 处理。默认为 false。注意：该参数将被废弃，请使用[原始目的地址](http://www.servicemesher.com/envoy/configuration/listener_filters/original_dst_filter.html)的 Listener filter 替代。该参数的主要用途是，Envoy 通过监听 15001 端口将应用的流量截取后再由其他 Listener 处理而不是直接转发出去，详情见 [Virtual Listener](https://zhaohuabing.com/post/2018-09-25-istio-traffic-management-impl-intro/#virtual-listener)。

关于 Listener 的详细介绍请参考 [Envoy v2 API reference - listener](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/lds.proto#envoy-api-msg-listener)。

### Cluster

Cluster 是指 Envoy 连接的一组逻辑相同的上游主机。Envoy 通过[服务发现](http://www.servicemesher.com/envoy/intro/arch_overview/service_discovery.html)来发现 cluster 的成员。可以选择通过[主动健康检查](http://www.servicemesher.com/envoy/intro/arch_overview/health_checking.html#arch-overview-health-checking)来确定集群成员的健康状态。Envoy 通过[负载均衡策略](http://www.servicemesher.com/envoy/intro/arch_overview/load_balancing.html#arch-overview-load-balancing)决定将请求路由到 cluster 的哪个成员。

**Cluster 的特点**

- 一组逻辑上相同的主机构成一个 cluster。
- 可以在 cluster 中定义各种负载均衡策略。
- 新加入的 cluster 需要一个热身的过程才可以给路由引用，该过程是原子的，即在 cluster 热身之前对于 Envoy 及 Service Mesh 的其余部分来说是不可见的。
- 可以通过多种方式来配置 cluster，例如静态类型、严格限定 DNS、逻辑 DNS、EDS 等。

**Cluster 的数据结构**

Cluster 的数据结构如下，除了 `name` 字段，其他都是可选的。

```json
{
  "name": "...",
  "alt_stat_name": "...",
  "type": "...",
  "eds_cluster_config": "{...}",
  "connect_timeout": "{...}",
  "per_connection_buffer_limit_bytes": "{...}",
  "lb_policy": "...",
  "hosts": [],
  "load_assignment": "{...}",
  "health_checks": [],
  "max_requests_per_connection": "{...}",
  "circuit_breakers": "{...}",
  "tls_context": "{...}",
  "common_http_protocol_options": "{...}",
  "http_protocol_options": "{...}",
  "http2_protocol_options": "{...}",
  "extension_protocol_options": "{...}",
  "dns_refresh_rate": "{...}",
  "dns_lookup_family": "...",
  "dns_resolvers": [],
  "outlier_detection": "{...}",
  "cleanup_interval": "{...}",
  "upstream_bind_config": "{...}",
  "lb_subset_config": "{...}",
  "ring_hash_lb_config": "{...}",
  "original_dst_lb_config": "{...}",
  "least_request_lb_config": "{...}",
  "common_lb_config": "{...}",
  "transport_socket": "{...}",
  "metadata": "{...}",
  "protocol_selection": "...",
  "upstream_connection_options": "{...}",
  "close_connections_on_host_health_failure": "...",
  "drain_connections_on_host_removal": "..."
}
```

下面是关于上述数据结构中的常用配置解析。

- **name**：如果你留意到作为 Sidecar 启动的 Envoy 的参数的会注意到 `--max-obj-name-len 189`，该选项用来用来指定 cluster 的名字，例如 `inbound|9080||ratings.default.svc.cluster.local`。该名字字符串由 `|` 分隔成四个部分，分别是 `inbound` 或 `outbound` 代表入向流量或出向流量、端口号、subcluster 名称、FQDN，其中 subcluster 名称将对应于 Istio `DestinationRule` 中配置的 `subnet`，如果是按照多版本按比例路由的话，该值可以是版本号。
- **type**：即[服务发现类型](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/service_discovery#arch-overview-service-discovery-types)，支持的参数有 `STATIC`（缺省值）、`STRICT_DNS`、`LOGICAL_DNS`、`EDS`、`ORIGINAL_DST`。
- **hosts**：这是个列表，配置负载均衡的 IP 地址和端口，只有使用了`STATIC`、`STRICT_DNS`、`LOGICAL_DNS` 服务发现类型时才需要配置。
- **eds_cluster_config**：如果使用 `EDS` 做服务发现，则需要配置该项目，其中包括的配置有 `service_name` 和 `ads`。

关于 Cluster 的详细介绍请参考 [Envoy v2 API reference - cluster](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/cds.proto#cluster)。

### Route

我们在这里所说的路由指的是 [HTTP 路由](http://www.servicemesher.com/envoy/intro/arch_overview/http_routing.html)，这也使得 Envoy 可以用来处理网格边缘的流量。HTTP 路由转发是通过路由过滤器实现的。该过滤器的主要职能就是执行[路由表](https://www.envoyproxy.io/docs/envoy/latest/api-v1/route_config/route_config#config-http-conn-man-route-table)中的指令。除了可以做重定向和转发，路由过滤器还需要处理重试、统计之类的任务。

**HTTP 路由的特点**

- 前缀和精确路径匹配规则。
- 可跨越多个上游集群进行基于[权重/百分比的路由](https://www.envoyproxy.io/docs/envoy/latest/api-v1/route_config/route#config-http-conn-man-route-table-route-weighted-clusters)。
- 基于[优先级](http://www.servicemesher.com/envoy/intro/arch_overview/http_routing.html#arch-overview-http-routing-priority)的路由。
- 基于[哈希](https://www.envoyproxy.io/docs/envoy/latest/api-v1/route_config/route#config-http-conn-man-route-table-hash-policy)策略的路由。

**Route 的数据结构**

```json
{
  "name": "...",
  "virtual_hosts": [],
  "internal_only_headers": [],
  "response_headers_to_add": [],
  "response_headers_to_remove": [],
  "request_headers_to_add": [],
  "request_headers_to_remove": [],
  "validate_clusters": "{...}"
}
```

下面是关于上述数据结构中的常用配置解析。

- **name**：该名字跟 `envoy.http_connection_manager` filter 中的 `http_filters.rds.route_config_name` 一致，在 Istio Service Mesh 中为 Envoy 下发的配置中的 Route 是以监听的端口号作为名字，而同一个名字下面的 `virtual_hosts` 可以有多个值（数组形式）。
- **virtual_hosts**：因为 **VirtualHosts** 是 Envoy 中引入的一个重要概念，我们在下文将详细说明 `virtual_hosts` 的数据结构。
- **validate_clusters**：这是一个布尔值，用来设置开启使用 cluster manager 来检测路由表引用的 cluster 是否有效。如果是路由表是通过 [route_config](https://www.envoyproxy.io/docs/envoy/latest/api-v2/config/filter/network/http_connection_manager/v2/http_connection_manager.proto#envoy-api-field-config-filter-network-http-connection-manager-v2-httpconnectionmanager-route-config) 静态配置的则该值默认设置为 true，如果是使用 [rds](https://www.envoyproxy.io/docs/envoy/latest/api-v2/config/filter/network/http_connection_manager/v2/http_connection_manager.proto#envoy-api-field-config-filter-network-http-connection-manager-v2-httpconnectionmanager-rds) 动态配置的话，则该值默认设置为 false。

关于 Route 的详细介绍请参考 [Envoy v2 API reference - HTTP route configuration](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/rds.proto)。

#### route.VirtualHost

VirtualHost 即上文中 Route 配置中的 `virtual_hosts`，VirtualHost 是路由配置中的顶级元素。每个虚拟主机都有一个逻辑名称以及一组根据传入请求的 host header 路由到它的域。这允许单个 Listener 为多个顶级域路径树提供服务。基于域选择了虚拟主机后 Envoy 就会处理路由以查看要路由到哪个上游集群或是否执行重定向。

**VirtualHost 的数据结构**

下面是 VirtualHost 的数据结构，除了 `name` 和 `domains` 是必须配置项外，其他皆为可选项。

```json
{
  "name": "...",
  "domains": [],
  "routes": [],
  "require_tls": "...",
  "virtual_clusters": [],
  "rate_limits": [],
  "request_headers_to_add": [],
  "request_headers_to_remove": [],
  "response_headers_to_add": [],
  "response_headers_to_remove": [],
  "cors": "{...}",
  "per_filter_config": "{...}",
  "include_request_attempt_count": "..."
}
```

下面是关于上述数据结构中的常用配置解析。

- **name**：该 VirtualHost 的名字，一般是 FQDN 加端口，如 `details.default.svc.cluster.local:9080`。
- **domains**：这是个用来匹配 VirtualHost 的域名（host/authority header）列表，也可以使用通配符，但是通配符不能匹配空字符，除了仅使用 `*` 作为 domains，注意列表中的值不能重复和存在交集，只要有一条 domain 被匹配上了，就会执行路由。Istio 会为该值配置所有地址解析形式，包括 IP 地址、FQDN 和短域名等。
- **routes**：针对入口流量的有序路由列表，第一个匹配上的路由将被执行。我们在下文将详细说明 route 的数据结构。

下面是一个实际的 VirtualHost 的例子，该配置来自 [Bookinfo 应用](https://istio.io/zh/docs/examples/bookinfo/)的 details 应用的 Sidecar 服务。

```json
{
            "name": "details.default.svc.cluster.local:9080",
            "domains": [
                "details.default.svc.cluster.local",
                "details.default.svc.cluster.local:9080",
                "details",
                "details:9080",
                "details.default.svc.cluster",
                "details.default.svc.cluster:9080",
                "details.default.svc",
                "details.default.svc:9080",
                "details.default",
                "details.default:9080",
                "10.254.4.113",
                "10.254.4.113:9080"
            ],
            "routes": [
                {
                    "match": {
                        "prefix": "/"
                    },
                    "route": {
                        "cluster": "outbound|9080||details.default.svc.cluster.local",
                        "timeout": "0s",
                        "max_grpc_timeout": "0s"
                    },
                    "decorator": {
                        "operation": "details.default.svc.cluster.local:9080/*"
                    },
                    "per_filter_config": {
                        "mixer": {
                            "forward_attributes": {
                                "attributes": {
                                    "destination.service.uid": {
                                        "string_value": "istio://default/services/details"
                                    },
                                    "destination.service.host": {
                                        "string_value": "details.default.svc.cluster.local"
                                    },
                                    "destination.service.namespace": {
                                        "string_value": "default"
                                    },
                                    "destination.service.name": {
                                        "string_value": "details"
                                    },
                                    "destination.service": {
                                        "string_value": "details.default.svc.cluster.local"
                                    }
                                }
                            },
                            "mixer_attributes": {
                                "attributes": {
                                    "destination.service.host": {
                                        "string_value": "details.default.svc.cluster.local"
                                    },
                                    "destination.service.uid": {
                                        "string_value": "istio://default/services/details"
                                    },
                                    "destination.service.name": {
                                        "string_value": "details"
                                    },
                                    "destination.service.namespace": {
                                        "string_value": "default"
                                    },
                                    "destination.service": {
                                        "string_value": "details.default.svc.cluster.local"
                                    }
                                }
                            },
                            "disable_check_calls": true
                        }
                    }
                }
            ]
        }
```

关于 route.VirtualHost 的详细介绍请参考 [Envoy v2 API reference - route.VirtualHost](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/route/route.proto#envoy-api-msg-route-virtualhost)。

#### route.Route

路由既是如何匹配请求的规范，也是对下一步做什么的指示（例如，redirect、forward、rewrite等）。

**route.Route 的数据结构**

下面是是 route.Route 的数据结构，除了 `match` 之外其余都是可选的。

```json
{
  "match": "{...}",
  "route": "{...}",
  "redirect": "{...}",
  "direct_response": "{...}",
  "metadata": "{...}",
  "decorator": "{...}",
  "per_filter_config": "{...}",
  "request_headers_to_add": [],
  "request_headers_to_remove": [],
  "response_headers_to_add": [],
  "response_headers_to_remove": []
}
```

下面是关于上述数据结构中的常用配置解析。

- **match**：路由匹配参数。例如 URL prefix（前缀）、path（URL 的完整路径）、regex（规则表达式）等。
- **route**：这里面配置路由的行为，可以是 [route](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/route/route.proto#envoy-api-field-route-route-route)、[redirect](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/route/route.proto#envoy-api-field-route-route-redirect) 和 [direct_response](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/route/route.proto#envoy-api-field-route-route-direct-response)，不过这里面没有专门的一个配置项用来配置以上三种行为，而是根据实际填充的配置项来确定的。例如在此处添加 `cluster` 配置则暗示路由动作为”route“，表示将流量路由到该 cluster。详情请参考 [route.RouteAction](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/route/route.proto#route-routeaction)。
- **decorator**：被匹配的路由的修饰符，表示被匹配的虚拟主机和 URL。该配置里有且只有一个必须配置的项 `operation`，例如 `details.default.svc.cluster.local:9080/*`。
- **per_filter_config**：这是一个 map 类型，`per_filter_config` 字段可用于为 filter 提供特定路由的配置。Map 的 key 应与 filleter 名称匹配，例如用于 HTTP buffer filter 的 `envoy.buffer`。该字段是特定于 filter 的，详情请参考 [HTTP filter](https://www.envoyproxy.io/docs/envoy/latest/configuration/http_filters/http_filters#config-http-filters)。

关于 route.Route 的详细介绍请参考 [Envoy v2 API reference - route.Route](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/route/route.proto#envoy-api-msg-route-route)。

## 4.5 Envoy API

Envoy 提供了如下的 API：

- CDS（Cluster Discovery Service）：集群发现服务
- EDS（Endpoint Discovery Service）：端点发现服务
- HDS（Health Discovery Service）：健康发现服务
- LDS（Listener Discovery Service）：监听器发现服务
- MS（Metric Service）：将 metric 推送到远端服务器
- RLS（Rate Limit Service）：速率限制服务
- RDS（Route Discovery Service）：路由发现服务
- SDS（Secret Discovery Service）：秘钥发现服务

所有名称以 DS 结尾的服务统称为 xDS。

本书中仅讨论 v2 版本的 API，因为 Envoy 仍在不断开发和完善中，随着版本迭代也有可能新增一些 API，本章的重点在于 xDS 协议，关于 Envoy 的 API 的更多信息请参考 [Envoy v2 APIs for developers](https://github.com/envoyproxy/envoy/blob/master/api/API_OVERVIEW.md)。

### 4.5.1 Envoy xDS 协议

Envoy xDS 为 Istio 控制平面与数据平面通信的基本协议，只要代理支持该协议表达形式就可以创建自己的 Sidecar 来替换 Envoy。这一章中将带大家了解 Envoy xDS。

Envoy 是 Istio Service Mesh 中默认的 Sidecar，Istio 在 Enovy 的基础上按照 Envoy 的 xDS 协议扩展了其控制平面，在讲到 Envoy xDS 协议之前还需要我们先熟悉下 Envoy 的基本术语。下面列举了 Envoy 里的基本术语及其数据结构解析，关于 Envoy 的详细介绍请参考 [Envoy 官方文档](http://www.servicemesher.com/envoy/)，至于 Envoy 在 Service Mesh（不仅限于 Istio） 中是如何作为转发代理工作的请参考网易云刘超的这篇[深入解读 Service Mesh 背后的技术细节 ](https://www.cnblogs.com/163yun/p/8962278.html)以及[理解 Istio Service Mesh 中 Envoy 代理 Sidecar 注入及流量劫持](https://jimmysong.io/posts/envoy-sidecar-injection-in-istio-service-mesh-deep-dive/)，本文引用其中的一些观点，详细内容不再赘述。

[![Envoy proxy 架构图](https://www.servicemesher.com/istio-handbook/images/006tNc79ly1fz69bsaqk7j314k0tsq90.jpg)](https://www.servicemesher.com/istio-handbook/images/006tNc79ly1fz69bsaqk7j314k0tsq90.jpg)图片 - Envoy proxy 架构图

### 4.5.2 关于 xDS 的版本

有一点需要大家注意，就是 Envoy 的 API 有 v1 和 v2 两个版本，从 Envoy 1.5.0 起 v2 API 就已经生产就绪了，为了能够让用户顺利的向 v2 版本的 API 过度，Envoy 启动的时候设置了一个 `--v2-config-only` 的标志，Enovy 不同版本对 v1/v2 API 的支持详情请参考 [Envoy v1 配置废弃时间表](https://groups.google.com/forum/#!topic/envoy-announce/Lb1QZcSclGQ)。

Envoy 的作者 Matt Klein 在 [Service Mesh 中的通用数据平面 API 设计](http://www.servicemesher.com/blog/the-universal-data-plane-api/)这篇文章中说明了 Envoy API v1 的历史及其缺点，还有 v2 的引入。v2 API 是 v1 的演进，而不是革命，它是 v1 功能的超集。

在 Istio 1.0 及以上版本中使用的是 **Envoy 1.8.0-dev** 版本，其支持 v2 的 API，同时在 Envoy 作为 Sidecar proxy 启动的使用使用了例如下面的命令：

```bash
$ /usr/local/bin/envoy -c /etc/istio/proxy/envoy-rev0.json --restart-epoch 0 --drain-time-s 45 --parent-shutdown-time-s 60 --service-cluster ratings --service-node sidecar~172.33.14.2~ratings-v1-8558d4458d-ld8x9.default~default.svc.cluster.local --max-obj-name-len 189 --allow-unknown-fields -l warn --v2-config-only
```

上面是都 Bookinfo 示例中的 rating pod 中的 sidecar 启动的分析，可以看到其中指定了 `--v2-config-only`，表明 Istio 1.0+ 只支持 xDS v2 的 API。

### 4.5.3 REST-JSON & gPRC API

单个的基本 xDS 订阅服务，如 CDS、EDS、LDS、RDS、SDS 同时支持 REST-JSON 和 gRPC API 配置。高级 API，如 HDS、ADS 和 EDS 多维 LB 仅支持 gRPC。这是为了避免将复杂的双向流语义映射到 REST。详见 [Envoy v2 APIs for developers](https://github.com/envoyproxy/envoy/blob/master/api/API_OVERVIEW.md)。

## 4.6 xDS 协议解析

> 本文译自 [xDS REST and gRPC protocol](https://github.com/envoyproxy/data-plane-api/blob/master/xds_protocol.rst)，译者：狄卫华，审校：宋净超

Envoy 通过查询文件或管理服务器来动态发现资源。概括地讲，对应的发现服务及其相应的 API 被称作 *xDS*。Envoy 通过订阅（*subscription*）方式来获取资源，如监控指定路径下的文件、启动 gRPC 流或轮询 REST-JSON URL。后两种方式会发送 [`DiscoveryRequest`](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/discovery.proto#discoveryrequest) 请求消息，发现的对应资源则包含在响应消息 [`DiscoveryResponse`](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/discovery.proto#discoveryresponse) 中。下面，我们将具体讨论每种订阅类型。

### 4.6.1 文件订阅

发现动态资源的最简单方式就是将其保存于文件，并将路径配置在 [ConfigSource](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/core/config_source.proto#core-configsource) 中的 `path` 参数中。Envoy 使用 `inotify`（Mac OS X 上为 `kqueue`）来监控文件的变化，在文件被更新时，Envoy 读取保存的 `DiscoveryResponse` 数据进行解析，数据格式可以为二进制 protobuf、JSON、YAML 和协议文本等。

> 译者注：core.ConfigSource 配置格式如下：

```json
{
  "path": "...",
  "api_config_source": "{...}",
  "ads": "{...}"
}
```

文件订阅方式可提供统计数据和日志信息，但是缺少 ACK/NACK 更新的机制。如果更新的配置被拒绝，xDS API 则继续使用最后一个的有效配置。

### 4.6.2 gRPC 流式订阅

#### 4.6.2.1 单资源类型发现

每个 xDS API 可以单独配置 [`ApiConfigSource`](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/core/config_source.proto#core-apiconfigsource)，指向对应的上游管理服务器的集群地址。每个 xDS 资源类型会启动一个独立的双向 gRPC 流，可能对应不同的管理服务器。API 交付方式采用最终一致性。可以参考后续聚合服务发现（ADS） 章节来了解必要的显式控制序列。

> 译者注：core.ApiConfigSource 配置格式如下：

```json
{
  "api_type": "...",
  "cluster_names": [],
  "grpc_services": [],
  "refresh_delay": "{...}",
  "request_timeout": "{...}"
}
```

##### 4.6.2.1.1 类型 URL

每个 xDS API 都与给定的资源的类型存在 1:1 对应。关系如下：

- [LDS： `envoy.api.v2.Listener`](https://github.com/envoyproxy/data-plane-api/blob/master/envoy/api/v2/lds.proto)
- [RDS： `envoy.api.v2.RouteConfiguration`](https://github.com/envoyproxy/data-plane-api/blob/master/envoy/api/v2/rds.proto)
- [CDS： `envoy.api.v2.Cluster`](https://github.com/envoyproxy/data-plane-api/blob/master/envoy/api/v2/cds.proto)
- [EDS： `envoy.api.v2.ClusterLoadAssignment`](https://github.com/envoyproxy/data-plane-api/blob/master/envoy/api/v2/eds.proto)
- [SDS：`envoy.api.v2.Auth.Secret`](https://github.com/envoyproxy/data-plane-api/blob/master/envoy/api/v2/auth/cert.proto)

[*类型 URL*](https://developers.google.com/protocol-buffers/docs/proto3#any) 的概念如下所示，其采用 `type.googleapis.com/<resource type>` 的形式，例如 CDS 对应于`type.googleapis.com/envoy.api.v2.Cluster`。在 Envoy 的请求和管理服务器的响应中，都包括了资源类型 URL。

##### 4.6.2.1.2 ACK/NACK 和版本

每个 Envoy 流以 `DiscoveryRequest` 开始，包括了列表订阅的资源、订阅资源对应的类型 URL、节点标识符和空的 `version_info`。EDS 请求示例如下：

```yaml
version_info:
node: { id: envoy }
resource_names:
- foo
- bar
type_url: type.googleapis.com/envoy.api.v2.ClusterLoadAssignment
response_nonce:
```

管理服务器可立刻或等待资源就绪时发送 `DiscoveryResponse`作为响应，示例如下：

```yaml
version_info: X
resources:
- foo ClusterLoadAssignment proto encoding
- bar ClusterLoadAssignment proto encoding
type_url: type.googleapis.com/envoy.api.v2.ClusterLoadAssignment
nonce: A
```

Envoy 在处理 `DiscoveryResponse` 响应后，将通过流发送一个新的请求，请求包含应用成功的最后一个版本号和管理服务器提供的 `nonce`。如果本次更新已成功应用，则 `version_info` 的值设置为 **X**，如下序列图所示：

[![ACK 后的版本更新](https://www.servicemesher.com/istio-handbook/images/7e0ee03agy1fvmxs5aod1j20cc06y74c.jpg)](https://www.servicemesher.com/istio-handbook/images/7e0ee03agy1fvmxs5aod1j20cc06y74c.jpg)图片 - ACK 后的版本更新

在此序列图及后续中，将统一使用以下缩写格式：

- `DiscoveryRequest`： (V=`version_info`，R=`resource_names`，N=`response_nonce`，T=`type_url`)
- `DiscoveryResponse`： (V=`version_info`，R=`resources`，N=`nonce`，T=`type_url`)

> 译者注：在[信息安全](https://zh.wikipedia.org/wiki/資訊安全)中，**Nonce**是一个在加密通信只能使用一次的数字。在认证协议中，它往往是一个[随机](https://zh.wikipedia.org/wiki/随机)或[伪随机](https://zh.wikipedia.org/wiki/伪随机)数，以避免[重放攻击](https://zh.wikipedia.org/wiki/重放攻击)。Nonce也用于[流密码](https://zh.wikipedia.org/wiki/流密码)以确保安全。如果需要使用相同的密钥加密一个以上的消息，就需要Nonce来确保不同的消息与该密钥加密的密钥流不同。（引用自[维基百科](https://zh.wikipedia.org/wiki/Nonce)）在本文中`nonce`是每次更新的数据包的唯一标识。

版本为 Envoy 和管理服务器提供了共享当前应用配置的概念和通过 ACK/NACK 来进行配置更新的机制。如果 Envoy 拒绝配置更新 **X**，则回复 [`error_detail`](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/discovery.proto#envoy-api-field-discoveryrequest-error-detail) 及前一个的版本号，在当前情况下为空的初始版本号，`error_detail` 包含了有关错误的更加详细的信息：

[![NACK 无版本更新](https://www.servicemesher.com/istio-handbook/images/7e0ee03agy1fvmxtjqtcsj20cc06y0ss.jpg)](https://www.servicemesher.com/istio-handbook/images/7e0ee03agy1fvmxtjqtcsj20cc06y0ss.jpg)图片 - NACK 无版本更新

后续，API 更新可能会在新版本 **Y** 上成功：

[![ACK 紧接着 NACK](https://www.servicemesher.com/istio-handbook/images/7e0ee03agy1fvmxtwzc96j20cc0923yp.jpg)](https://www.servicemesher.com/istio-handbook/images/7e0ee03agy1fvmxtwzc96j20cc0923yp.jpg)图片 - ACK 紧接着 NACK

每个流都有自己的版本概念，但不存在跨资源类型的共享版本。在不使用 ADS 的情况下，每个资源类型可能具有不同的版本，因为 Envoy API 允许指向不同的 EDS/RDS 资源配置并对应不同的 `ConfigSources`。

##### 4.6.2.1.3 何时发送更新

管理服务器应该只向 Envoy 客户端发送上次 `DiscoveryResponse` 后更新过的资源。Envoy 则会根据接受或拒绝 `DiscoveryResponse` 的情况，立即回复包含 ACK/NACK 的 `DiscoveryRequest` 请求。如果管理服务器每次发送相同的资源集结果，而不是根据其更新情况，则会导致 Envoy 和管理服务器通讯效率大打折扣。

在同一个流中，新的 `DiscoveryRequests` 将取代此前具有相同的资源类型 `DiscoveryRequest` 请求。**这意味着管理服务器只需要响应给定资源类型最新的 DiscoveryRequest 请求即可。**

##### 4.6.2.1.4 资源提示

`DiscoveryRequest` 中的 `resource_names` 信息作为资源提示出现。一些资源类型，例如 `Cluster` 和 `Listener` 将使用一个空的 `resource_names`，因为 Envoy 需要获取管理服务器对应于节点标识的所有 `Cluster`（CDS）和 `Listener`（LDS）。对于其他资源类型，如 `RouteConfigurations`（RDS）和 `ClusterLoadAssignments`（EDS），则遵循此前的 CDS/LDS 更新，Envoy 能够明确地枚举这些资源。

LDS/CDS 资源提示信息将始终为空，并且期望管理服务器的每个响应都提供 `LDS/CDS` 资源的完整状态。缺席的 `Listener` 或 `Cluster` 将被删除。

对于 EDS/RDS，管理服务器并不需要为每个请求的资源进行响应，而且还可能提供额外未请求的资源。`resource_names` 只是一个提示。Envoy 将默默地忽略返回的多余资源。如果请求的资源中缺少相应的 RDS 或 EDS 更新，Envoy 将保留对应资源的最后的值。管理服务器可能会依据 `DiscoveryRequest` 中 `node` 标识推断其所需的 EDS/RDS 资源，在这种情况下，提示信息可能会被丢弃。从相应的角度来看，空的 EDS/RDS `DiscoveryResponse` 响应实际上是表明在 Envoy 中为一个空的资源。

当 `Listener` 或 `Cluster` 被删除时，其对应的 EDS 和 RDS 资源也需要在 Envoy 实例中删除。为使 EDS 资源被 Envoy 已知或跟踪，就必须存在应用过的 `Cluster` 定义（如通过 CDS 获取）。RDS 和 `Listeners` 之间存在类似的关系（如通过 LDS 获取）。

对于 EDS/RDS ，Envoy 可以为每个给定类型的资源生成不同的流（如每个 `ConfigSource` 都有自己的上游管理服务器的集群）或当指定资源类型的请求发送到同一个管理服务器的时候，允许将多个资源请求组合在一起发送。虽然可以单个实现，但管理服务器应具备处理每个给定资源类型中对单个或多个 `resource_names` 请求的能力。下面的两个序列图对于获取两个 EDS 资源都是有效的 `{foo，bar}`：

[![一个流上多个 EDS 请求](https://www.servicemesher.com/istio-handbook/images/7e0ee03agy1fvmxuviiqsj20eh06ymx9.jpg)](https://www.servicemesher.com/istio-handbook/images/7e0ee03agy1fvmxuviiqsj20eh06ymx9.jpg) [![不同流上的多个 EDS 请求](https://www.servicemesher.com/istio-handbook/images/7e0ee03agy1fvmxv7cv21j20j20a4wet.jpg)](https://www.servicemesher.com/istio-handbook/images/7e0ee03agy1fvmxv7cv21j20j20a4wet.jpg)

##### 4.6.2.1.5 资源更新

如上所述，Envoy 可能会更新 `DiscoveryRequest` 中出现的 `resource_names` 列表，其中 `DiscoveryRequest` 是用来 ACK/NACK 管理服务器的特定的 `DiscoveryResponse` 。此外，Envoy 后续可能会发送额外的 `DiscoveryRequests` ，用于在特定 `version_info` 上使用新的资源提示来更新管理服务器。例如，如果 Envoy 在 EDS 版本 **X** 时仅知道集群 `foo`，但在随后收到的 CDS 更新时额外获取了集群 `bar` ，它可能会为版本 **X** 发出额外的 `DiscoveryRequest` 请求，并将 `{foo，bar}` 作为请求的 `resource_names` 。

[![CDS 响应导致 EDS 资源更新](https://www.servicemesher.com/istio-handbook/images/006tNc79ly1fvph0p7u8zj31fm0lq0ve.jpg)](https://www.servicemesher.com/istio-handbook/images/006tNc79ly1fvph0p7u8zj31fm0lq0ve.jpg)图片 - CDS 响应导致 EDS 资源更新

这里可能会出现竞争状况；如果 Envoy 在版本 **X** 上发布了资源提示更新请求，但在管理服务器处理该请求之前发送了新的版本号为 **Y** 的响应，针对 `version_info` 为 **X** 的版本，资源提示更新可能会被解释为拒绝**Y** 。为避免这种情况，通过使用管理服务器提供的 `nonce`，Envoy 可用来保证每个 `DiscoveryRequest`对应到相应的 `DiscoveryResponse` ：

[![EDS 更新速率激发 nonces](https://www.servicemesher.com/istio-handbook/images/006tNc79ly1fvph04ln3fj31kw0rogqc.jpg)](https://www.servicemesher.com/istio-handbook/images/006tNc79ly1fvph04ln3fj31kw0rogqc.jpg)图片 - EDS 更新速率激发 nonces

管理服务器不应该为含有过期 `nonce` 的 `DiscoveryRequest` 发送 `DiscoveryResponse` 响应。在向 Envoy 发送的 `DiscoveryResponse` 中包含了的新 `nonce` ，则此前的 `nonce` 将过期。在资源新版本就绪之前，管理服务器不需要向 Envoy 发送更新。同版本的早期请求将会过期。在新版本就绪时，管理服务器可能会处理同一个版本号的多个 `DiscoveryRequests`请求。

[![请求变的陈旧](https://www.servicemesher.com/istio-handbook/images/006tNc79ly1fvpgy6xewrj31b415ctcy.jpg)](https://www.servicemesher.com/istio-handbook/images/006tNc79ly1fvpgy6xewrj31b415ctcy.jpg)图片 - 请求变的陈旧

上述资源更新序列表明 Envoy 并不能期待其发出的每个 `DiscoveryRequest` 都得到 `DiscoveryResponse`响应。

##### 4.6.2.1.6 **最终一致性考虑**

由于 Envoy 的 xDS API 采用最终一致性，因此在更新期间可能导致流量被丢弃。例如，如果通过 CDS/EDS 仅获取到了集群 **X**，而且 `RouteConfiguration` 引用了集群 **X**；在 CDS/EDS 更新集群 **Y** 配置之前，如果将 `RouteConfiguration` 将引用的集群调整为 **Y** ，那么流量将被吸入黑洞而丢弃，直至集群 **Y** 被 Envoy 实例获取。

对某些应用程序，可接受临时的流量丢弃，客户端重试或其他 Envoy sidecar 会掩盖流量丢弃。那些对流量丢弃不能容忍的场景，可以通过以下方式避免流量丢失，CDS/EDS 更新同时携带 **X** 和 **Y** ，然后发送 RDS 更新从 **X** 切换到 **Y** ，此后发送丢弃 **X** 的 CDS/EDS 更新。

一般来说，为避免流量丢弃，更新的顺序应该遵循 `make before break` 模型，其中

- 必须始终先推送 CDS 更新（如果有）。
- EDS 更新（如果有）必须在相应集群的 CDS 更新后到达。
- LDS 更新必须在相应的 CDS/EDS 更新后到达。
- 与新添加的监听器相关的 RDS 更新必须在最后到达。
- 最后，删除过期的 CDS 集群和相关的 EDS 端点（不再被引用的端点）。

如果没有新的集群/路由/监听器或者允许更新时临时流量丢失的情况下，可以独立推送 xDS 更新。请注意，在 LDS 更新的情况下，监听器须在接收流量之前被预热，例如如其配置了依赖的路由，则先需先从 RDS 进行获取。添加/删除/更新集群信息时，集群也需要进行预热。另一方面，如果管理平面确保路由更新时所引用的集群已经准备就绪，路由可以不用预热。

#### 4.6.2.2 聚合服务发现（ADS）

当管理服务器进行资源分发时，通过上述保证交互顺序的方式来避免流量丢弃是一项很有挑战的工作。ADS 允许单一管理服务器通过单个 gRPC 流，提供所有的 API 更新。配合仔细规划的更新顺序，ADS 可规避更新过程中流量丢失。使用 ADS，在单个流上可通过类型 URL 来进行复用多个独立的 `DiscoveryRequest`/`DiscoveryResponse` 序列。对于任何给定类型的 URL，以上 `DiscoveryRequest`和 `DiscoveryResponse` 消息序列都适用。 更新序列可能如下所示：

[![EDS/CDS 在一个 ADS 流上多路复用](https://www.servicemesher.com/istio-handbook/images/006tNc79ly1fvpgxnl947j313q0wgq62.jpg)](https://www.servicemesher.com/istio-handbook/images/006tNc79ly1fvpgxnl947j313q0wgq62.jpg)图片 - EDS/CDS 在一个 ADS 流上多路复用

每个 Envoy 实例可使用单独的 ADS 流。

最小化 ADS 配置的 `bootstrap.yaml` 片段示例如下：

```yaml
node:
  id: <node identifier>
dynamic_resources:
  cds_config: {ads: {}}
  lds_config: {ads: {}}
  ads_config:
    api_type: GRPC
    grpc_services:
      envoy_grpc:
        cluster_name: ads_cluster
static_resources:
  clusters:
  - name: ads_cluster
    connect_timeout: { seconds: 5 }
    type: STATIC
    hosts:
    - socket_address:
        address: <ADS management server IP address>
        port_value: <ADS management server port>
    lb_policy: ROUND_ROBIN
    http2_protocol_options: {}
admin:
  ...
```

#### 4.6.2.3 增量 xDS

增量 xDS 是可用于允许的 ADS、CDS 和 RDS 单独 xDS 端点：

- xDS 客户端对跟踪资源列表进行增量更新。这支持 Envoy 按需/惰性地请求额外资源。例如，当与未知集群相对应的请求到达时，可能会发生这种情况。
- xDS 服务器可以增量更新客户端上的资源。这支持 xDS 资源可伸缩性的目标。管理服务器只需交付更改的单个集群，而不是在修改单个集群时交付所有上万个集群。

xDS 增量会话始终位于 gRPC 双向流的上下文中。这允许 xDS 服务器能够跟踪到连接的 xDS 客户端的状态。xDS REST 版本不支持增量。

在增量 xDS 中，nonce 字段是必需的，用于匹配 [`IncrementalDiscoveryResponse`](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/discovery.proto#discoveryrequest) 关联的 ACK 或 NACK [`IncrementalDiscoveryRequest`](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/discovery.proto#discoveryrequest)。可选地，存在响应消息级别的 system_version_info，但仅用于调试目的。

`IncrementalDiscoveryRequest` 可在以下 3 种情况下发送：

1. xDS 双向 gRPC 流的初始消息。
2. 作为对先前的 `IncrementalDiscoveryResponse` 的 ACK 或 NACK 响应。在这种情况下，`response_nonce` 被设置为响应中的 nonce 值。ACK 或 NACK 由可由 `error_detail` 字段是否出现来区分。
3. 客户端自发的 `IncrementalDiscoveryRequest`。此场景下可以采用动态添加或删除被跟踪的 `resource_names` 集。这种场景下，必须忽略 `response_nonce`。

在第一个示例中，客户端连接并接收它的第一个更新并 ACK。第二次更新失败，客户端发送 NACK 拒绝更新。xDS客户端后续会自发地请求 “wc” 相关资源。

[![增量 session 示例](https://www.servicemesher.com/istio-handbook/images/006tNc79ly1fvpgwfbep7j31kw0vldli.jpg)](https://www.servicemesher.com/istio-handbook/images/006tNc79ly1fvpgwfbep7j31kw0vldli.jpg)图片 - 增量 session 示例

在重新连接时，支持增量的 xDS 客户端可能会告诉服务器其已知资源从而避免通过网络重新发送它们。

[![增量重连示例](https://www.servicemesher.com/istio-handbook/images/006tNc79ly1fvpgx05z3kj31kw0phwif.jpg)](https://www.servicemesher.com/istio-handbook/images/006tNc79ly1fvpgx05z3kj31kw0phwif.jpg)图片 - 增量重连示例

### 4.6.3 REST-JSON 轮询订阅

单个 xDS API 可对 REST 端点进行同步（长）轮询。除了无持久流与管理服务器交互外，消息顺序与上述相似。在任何时间点，只存在一个未完成的请求，因此响应消息中的 `nonce` 在 REST-JSON 中是可选的。`DiscoveryRequest` 和 `DiscoveryResponse` 的消息编码遵循 [JSON 变换 proto3](https://developers.google.com/protocol-buffers/docs/proto3#json) 规范。ADS 不支持 REST-JSON 轮询。

当轮询期间设置为较小的值时，则可以等同于长轮询，这时要求避免发送 `DiscoveryResponse`，[除非对请求的资源发生了更改](https://www.servicemesher.com/istio-handbook/data-plane/envoy-xds-protocol.html#何时发送更新)。

### 4.6.4 LDS（监听器发现服务）

Listener 发现服务（LDS）是一个可选的 API，Envoy 将调用它来动态获取 Listener。Envoy 将协调 API 响应，并根据需要添加、修改或删除已知的 Listener。

Listener 更新的语义如下：

- 每个 Listener 必须有一个独特的[名字](https://www.envoyproxy.io/docs/envoy/latest/api-v1/listeners/listeners.md#config-listeners-name)。如果没有提供名称，Envoy 将创建一个 UUID。要动态更新的 Listener ，管理服务必须提供 Listener 的唯一名称。
- 当一个 Listener 被添加，在参与连接处理之前，会先进入“预热”阶段。例如，如果 Listener 引用 [RDS](http://www.servicemesher.com/envoy/configuration/http_conn_man/rds.html#config-http-conn-man-rds)配置，那么在 Listener 迁移到 “active” 之前，将会解析并提取该配置。
- Listener 一旦创建，实际上就会保持不变。因此，更新 Listener 时，会创建一个全新的 Listener （使用相同的侦听套接字）。新增加的监听者都会通过上面所描述的相同“预热”过程。
- 当更新或删除 Listener 时，旧的 Listener 将被置于 “draining（逐出）” 状态，就像整个服务重新启动时一样。Listener 移除之后，该 Listener 所拥有的连接，经过一段时间优雅地关闭（如果可能的话）剩余的连接。逐出时间通过 [`--drain-time-s`](http://www.servicemesher.com/envoy/operations/cli.html#cmdoption-drain-time-s) 选项设置。

**注意**

- Envoy 从 1.9 版本开始已不再支持 v1 API。
- [v2 LDS API](http://www.servicemesher.com/envoy/configuration/overview/v2_overview.html#v2-grpc-streaming-endpoints)



### 4.6.5 RDS（路由发现服务）

路由发现服务（RDS）是 Envoy 里面的一个可选 API，用于动态获取[路由配置](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/rds.proto#envoy-api-msg-routeconfiguration)。路由配置包括 HTTP header 修改、虚拟主机以及每个虚拟主机中包含的单个路由规则配置。每个 [HTTP 连接管理器](https://www.envoyproxy.io/docs/envoy/latest/configuration/http_conn_man/http_conn_man#config-http-conn-man)都可以通过 API 独立地获取自身的路由配置。

- [v2 API 参考](https://www.envoyproxy.io/docs/envoy/latest/configuration/overview/v2_overview#v2-grpc-streaming-endpoints)

**注意**：Envoy 从 1.9 版本开始已不再支持 v1 API。

## 统计

RDS 的统计树以 `http.<stat_prefix>.rds.<route_config_name>.*.`为根，`route_config_name`名称中的任何`:`字符在统计树中被替换为`_`。统计树包含以下统计信息：

| 名字            | 类型    | 描述                                            |
| --------------- | ------- | ----------------------------------------------- |
| config_reload   | Counter | 因配置不同而导致配置重新加载的总次数            |
| update_attempt  | Counter | 尝试调用配置加载 API 的总次数                   |
| update_success  | Counter | 调用配置加载 API 成功的总次数                   |
| update_failure  | Counter | 调用配置加载 API 因网络错误的失败总数           |
| update_rejected | Counter | 调用配置加载 API 因 schema/验证错误的失败总次数 |
| version         | Gauge   | 来自上次成功调用配置加载API的内容哈希           |

### 4.6.6 CDS（集群发现服务）

集群发现服务（CDS）是一个可选的 API，Envoy 将调用该 API 来动态获取 cluster manager 的成员。Envoy 还将根据 API 响应协调集群管理，根据需要完成添加、修改或删除已知的集群。

关于 Envoy 是如何通过 CDS 从 `pilot-discovery` 服务中获取的 cluster 配置，请参考 [Service Mesh深度学习系列part3—istio源码分析之pilot-discovery模块分析（续）](http://www.servicemesher.com/blog/istio-service-mesh-source-code-pilot-discovery-module-deepin-part2)一文中的 CDS 服务部分。

**注意**

- 在 Envoy 配置中静态定义的 cluster 不能通过 CDS API 进行修改或删除。
- Envoy 从 1.9 版本开始已不再支持 v1 API。
- [v2 CDS API](https://www.envoyproxy.io/docs/envoy/latest/configuration/overview/v2_overview#v2-grpc-streaming-endpoints)

#### 4.6.6.1 统计

CDS 的统计树以 `cluster_manager.cds.` 为根，统计如下：

| 名字                          | 类型    | 描述                                                         |
| ----------------------------- | ------- | ------------------------------------------------------------ |
| config_reload                 | Counter | 因配置不同而导致配置重新加载的总次数                         |
| update_attempt                | Counter | 尝试调用配置加载 API 的总次数                                |
| update_success                | Counter | 调用配置加载 API 成功的总次数                                |
| update_failure                | Counter | 调用配置加载 API 因网络错误的失败总数                        |
| update_rejected               | Counter | 调用配置加载 API 因 schema/验证错误的失败总次数              |
| version                       | Gauge   | 来自上次成功调用配置加载API的内容哈希                        |
| control_plane.connected_state | Gauge   | 布尔值，用来表示与管理服务器的连接状态，1表示已连接，0表示断开连接 |

### 4.6.7 EDS（端点发现服务）

EDS 只是 Envoy 中众多的[服务发现](http://www.servicemesher.com/envoy/intro/arch_overview/service_discovery.html)方式的一种。要想了解 EDS 首先我们需要先知道什么是 Endpoint。

**Endpoint**

Endpoint 即上游主机标识。它的数据结构如下：

```json
{
  "address": "{...}",
  "health_check_config": "{...}"
}
```

其中包括端点的地址和健康检查配置。详情请参考 [endpoint.Endpoint](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/endpoint/endpoint.proto)。

终端发现服务（EDS）是一个[基于 gRPC 或 REST-JSON API 服务器的 xDS 管理服务](https://www.envoyproxy.io/docs/envoy/latest/configuration/overview/v2_overview#config-overview-v2-management-server)，在 Envoy 中用来获取集群成员。集群成员在 Envoy 的术语中被称为“终端”。对于每个集群，Envoy 都会通过发现服务来获取成员的终端。由于以下几个原因，EDS 是首选的服务发现机制：

- Envoy 对每个上游主机都有明确的了解（与通过 DNS 解析的负载均衡进行路由相比而言），并可以做出更智能的负载均衡决策。
- 在每个主机的发现 API 响应中携带的额外属性通知 Envoy 负载均衡权重、金丝雀状态、区域等。这些附加属性在负载均衡、统计信息收集等过程中会被 Envoy 网格全局使用。

Envoy 提供了 [Java](https://github.com/envoyproxy/java-control-plane) 和 [Go](https://github.com/envoyproxy/go-control-plane) 语言版本的 EDS 和[其他发现服务](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/dynamic_configuration#arch-overview-dynamic-config)的参考 gRPC 实现。

通常，主动健康检查与最终一致的服务发现服务数据结合使用，以进行负载均衡和路由决策。

### 4.6.8 SDS（秘钥发现服务）

SDS（秘钥发现服务）是 Envoy 1.8.0 版本起开始引入的服务。可以在 `bootstrap.static_resource` 的 [secrets](https://www.envoyproxy.io/docs/envoy/latest/api-v2/config/bootstrap/v2/bootstrap.proto#envoy-api-field-config-bootstrap-v2-bootstrap-staticresources-secrets) 配置中为 Envoy 指定 TLS 证书（[secret](https://www.envoyproxy.io/docs/envoy/latest/api-v2/config/bootstrap/v2/bootstrap.proto#envoy-api-field-config-bootstrap-v2-bootstrap-staticresources-secrets)）。也可以通过秘钥发现服务（SDS）远程获取。Istio 预计将在 1.1 版本中支持 SDS。

SDS 带来的最大的好处就是简化证书管理。要是没有该功能的话，我们就必须使用 Kubernetes 中的 [secret 资源](https://jimmysong.io/kubernetes-handbook/concepts/secret.html)创建证书，然后把证书挂载到代理容器中。如果证书过期，还需要更新 secret 和需要重新部署代理容器。使用 SDS，中央 SDS 服务器将证书推送到所有 Envoy 实例上。如果证书过期，服务器只需将新证书推送到 Envoy 实例，Envoy 可以立即使用新证书而无需重新部署。

如果 listener server 需要从远程获取证书，则 listener server 不会被标记为 active 状态，在获取证书之前不会打开其端口。如果 Envoy 由于连接失败或错误的响应数据而无法获取证书，则 listener server 将被标记为 active，并且打开端口，但是将重置与端口的连接。

上游集群的处理方式类似，如果需要通过 SDS 从远程获取集群客户端证书，则不会将其标记为 active 状态，在获得证书之前也它不会被使用。如果 Envoy 由于连接失败或错误的响应数据而无法获取证书，则集群将被标记为 active，可以处理请求，但路由到该集群的请求都将被拒绝。

使用 SDS 的静态集群需定义 SDS 集群（除非使用不需要集群的 Google gRPC），则必须在使用静态集群之前定义 SDS 集群。

Envoy 代理和 SDS 服务器之间的连接必须是安全的。可以在同一主机上运行 SDS 服务器，使用 Unix Domain Socket 进行连接。否则，需要代理和 SDS 服务器之间的 mTLS。在这种情况下，必须静态配置 SDS 连接的客户端证书。

#### 4.6.8.1 SDS Server

SDS server 需要实现 [SecretDiscoveryService](https://github.com/envoyproxy/envoy/blob/master/api/envoy/service/discovery/v2/sds.proto) 这个 gRPC 服务。遵循与其他 [xDS](https://github.com/envoyproxy/data-plane-api/blob/master/xds_protocol.rst) 相同的协议。

#### 4.6.8.2 SDS 配置

SDS 支持静态配置也支持动态配置。

**静态配置**

可以在`static_resources` 的 `secrets` 中配置 TLS 证书。

**动态配置**

从远程 SDS server 获取 secret。

- 通过 Unix Domain Socket 访问 gRPC SDS server。
- 通过 UDS 访问 gRPC SDS server。
- 通过 Envoy gRPC 访问 SDS server。

配置详情请参考 [Envoy 官方文档](https://www.envoyproxy.io/docs/envoy/latest/configuration/secret)。

### 4.6.9 ADS（聚合发现服务）

虽然 Envoy 本质上采用了最终一致性模型，但 ADS 提供了**对 API 更新推送进行排序**的机会，并确保单个管理服务器对 Envoy 节点的 API 更新具有亲和力。ADS 允许管理服务器在单个双向 gRPC 流上传递一个或多个 API 及其资源。否则，一些 API（如 RDS 和 EDS）可能需要管理多个流并连接到不同的管理服务器。

**ADS** 通过适当得排序 xDS 可以无中断的更新 Enovy 的配置。例如，假设 foo.com 已映射到集群 X。我们希望将路由表中将该映射更改为在集群 Y。为此，必须首先提供 X、Y 这两个集群的 CDS/EDS 更新。

如果没有 ADS，CDS/EDS/RDS 流可能指向不同的管理服务器，或者位于需要协调的不同 gRPC流连接的同一管理服务器上。EDS 资源请求可以跨两个不同的流分开，一个用于 X，一个用于 Y。ADS 将这些流合并到单个流和单个管理服务器，从而无需分布式同步就可以正确地对更新进行排序。使用 ADS，管理服务器将在单个流上提供 CDS、EDS 和 RDS 更新。

**ADS** 仅适用于 gRPC 流（非REST），[本文档](https://github.com/envoyproxy/data-plane-api/blob/master/xds_protocol.rst#aggregated-discovery-service-ads)对此进行了更全面的描述。

### 4.6.10 HDS（健康发现服务）

HDS（健康发现服务）支持管理服务器对其管理的 Envoy 实例进行高效的端点健康发现。单个 Envoy 实例通常会收到 HDS 指令，以检查所有端点的子集（subset）。运行状况检查子集可能并不是 Envoy 实例 EDS 端点的子集。

## 4.7 Envoy 高级 API

除了 xDS API，Envoy 还提供如下高级 API：

- MS（Metric Service）：将 metric 推送到远端服务器
- RLS（Rate Limit Service）：速率限制服务

### 4.7.1 MS（Metric 服务）	



### 4.7.2 RLS（速率限制服务）

# 五 控制层面

## 5.1 组件概览

### 5.1.1 istio 组件构成

以下是istio 1.1 官方架构图:

![image-20200118155802373](/Users/xuel/Library/Application Support/typora-user-images/image-20200118155802373.png)

虽然 Istio 支持多个平台，但将其与 Kubernetes 结合使用，其优势会更大，Istio 对 Kubernetes 平台支持也是最完善的，本文将基于 Istio + Kubernetes 进行展开。

在安装了grafana, prometheus, kiali, jaeger等后端组件的情况下, 一个完整的控制面组件应该包括以下pod：

```bash
% kubectl -n istio-system get pod
NAME                                          READY     STATUS
grafana-5f54556df5-s4xr4                      1/1       Running
istio-citadel-775c6cfd6b-8h5gt                1/1       Running
istio-galley-675d75c954-kjcsg                 1/1       Running
istio-ingressgateway-6f7b477cdd-d8zpv         1/1       Running
istio-pilot-7dfdb48fd8-92xgt                  2/2       Running
istio-policy-544967d75b-p6qkk                 2/2       Running
istio-sidecar-injector-5f7894f54f-w7f9v       1/1       Running
istio-telemetry-777876dc5d-msclx              2/2       Running
istio-tracing-5fbc94c494-558fp                1/1       Running
kiali-7c6f4c9874-vzb4t                        1/1       Running
prometheus-66b7689b97-w9glt                   1/1       Running
```

将istio系统组件细化到进程级别, 大概是这个样子：

![image-20200118155823385](/Users/xuel/Library/Application Support/typora-user-images/image-20200118155823385.png)

Service Mesh 的 Sidecar 模式要求对数据面的用户 Pod 进行代理的注入, 注入的代理容器会去处理服务治理领域的各种「脏活累活」, 使得用户容器可以专心处理业务逻辑。

从上图可以看出, Istio 控制面本身就是一个复杂的微服务系统, 该系统包含多个组件 Pod, 每个组件各司其职, 既有单容器 Pod, 也有多容器 Pod, 既有单进程容器, 也有多进程容器, 每个组件会调用不同的命令, 各组件之间会通过 RPC 进行协作, 共同完成对数据面用户服务的管控。

### 5.1.2  istio 源码, 镜像和命令

Istio 项目代码主要由以下两个 git 仓库组成:

| 仓库地址                       | 语言 | 模块                                                         |
| ------------------------------ | ---- | ------------------------------------------------------------ |
| https://github.com/istio/istio | Go   | 包含 istio 控制面的大部分组件: pilot, mixer, citadel, galley, sidecar-injector 等 |
| https://github.com/istio/proxy | C++  | 包含 istio 使用的边车代理, 这个边车代理包含 envoy 和 mixer client 两块功能 |
| https://github.com/istio/api   | Go   | 包含 istio 组件之间的 API 以及资源配置定义, 使用 protobuf 进行定义 |



#### 5.1.2.1 istio/istio

https://github.com/istio/istio 包含的主要的镜像和命令:

| 容器名                   | 镜像名                 | 启动命令          | 源码入口                          |
| ------------------------ | ---------------------- | ----------------- | --------------------------------- |
| Istio_init               | istio/proxy_init       | istio-iptables.sh | istio/tools/deb/istio-iptables.sh |
| istio-proxy              | istio/proxyv2          | pilot-agent       | istio/pilot/cmd/pilot-agent       |
| sidecar-injector-webhook | istio/sidecar_injector | sidecar-injector  | istio/pilot/cmd/sidecar-injector  |
| discovery                | istio/pilot            | pilot-discovery   | istio/pilot/cmd/pilot-discovery   |
| galley                   | istio/galley           | galley            | istio/galley/cmd/galley           |
| mixer                    | istio/mixer            | mixs              | istio/mixer/cmd/mixs              |
| citadel                  | istio/citadel          | istio_ca          | istio/security/cmd/istio_ca       |

另外还有2个命令不在上图中使用:

| 命令       | 源码入口                      | 作用                                                         |
| ---------- | ----------------------------- | ------------------------------------------------------------ |
| mixc       | istio/mixer/cmd/mixc          | 用于和 Mixer server 交互的客户端                             |
| node_agent | istio/security/cmd/node_agent | 用于 node 上安装安全代理, 这在 Mesh Expansion 特性中会用到, 即 k8s 和 vm 打通 |

#### 5.1.2.2 istio/proxy

https://github.com/istio/proxy 该项目本身不会产出镜像, 它可以编译出一个 `name = "Envoy"` 的二进制程序, 该二进制程序会被加入到 istio 的边车容器镜像 `istio/proxyv2` 中。

istio proxy 项目使用的编译方式是 Google 出品的 bazel , bazel 可以直接在编译中引入第三方库，加载第三方源码。

这个项目包含了对 Envoy 源码的引用，还在此基础上进行了扩展，这些扩展是通过 Envoy filter（过滤器）的形式来提供，这样做的目的是让边车代理将策略执行决策委托给 Mixer，因此可以理解为 istio proxy 这个项目有两大功能模块:

1. Envoy: 使用到 Envoy 的全部功能。
2. mixer client: 策略和遥测相关的客户端实现, 基于 Envoy 做扩展，通过 RPC 与 Mixer server 进行交互, 实现策略管控和遥测。

#### 5.1.2.3 istio/api

https://github.com/istio/api 使用 [protobuf](https://github.com/protocolbuffers/protobuf) 对 Istio API 和资源进行定义, 包括:

- 组件之间的 API, 如 Mesh Configuration Protocol (MCP) 等。
- 所有的 Istio CRD, 如 VirtualService、DestinationRule 等。

该项目会作为依赖包被 istio 主项目引用。

## 5.2 Sidecar Injector

用户空间的 Pod 要想加入 mesh, 首先需要注入 sidecar 容器, istio 提供了 2 种方式实现注入:

- 自动注入: 利用 [Kubernetes Dynamic Admission Webhooks](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/) 对 新建的 pod 进行注入: initContainer + sidecar
- 手动注入: 使用命令`istioctl kube-inject`

「注入」本质上就是修改 Pod 的资源定义, 添加相应的 sidecar 容器定义, 内容包括 2 个新容器:

- 名为`istio-init`的 initContainer: 通过配置 iptables 来劫持 Pod 中的流量
- 名为`istio-proxy`的 sidecar 容器: 两个进程 pilot-agent 和 envoy, pilot-agent 进行初始化并启动 envoy

### 5.2.1 Dynamic Admission Control

kubernetes 的准入控制(Admission Control)有 2 种:

- Built in Admission Control: 这些 Admission 模块可以选择性地编译进 api server, 因此需要修改和重启 kube-apiserver
- Dynamic Admission Control: 可以部署在 kube-apiserver 之外, 同时无需修改或重启 kube-apiserver。

其中, Dynamic Admission Control 包含 2 种形式:

- Admission Webhooks: 该 controller 提供 http server, 被动接受 kube-apiserver 分发的准入请求。

- Initializers: 该 controller 主动 list and watch 关注的资源对象, 对 watch 到的未初始化对象进行相应的改造。

  其中, Admission Webhooks 又包含 2 种准入控制:

- ValidatingAdmissionWebhook

- MutatingAdmissionWebhook

istio 使用了 MutatingAdmissionWebhook 来实现对用户 Pod 的注入, 首先需要保证以下条件满足:

- 确保 kube-apiserver 启动参数 开启了 MutatingAdmissionWebhook
- 给 namespace 增加 label: `kubectl label namespace default istio-injection=enabled`
- 同时还要保证 kube-apiserver 的 aggregator layer 开启: `--enable-aggregator-routing=true` 且证书和 api server 连通性正确设置。

另外还需要一个配置对象, 来告诉 kube-apiserver istio 关心的资源对象类型, 以及 webhook 的服务地址。 如果使用 helm 安装 istio, 配置对象已经添加好了, 查阅 MutatingWebhookConfiguration:

```yaml
% kubectl get mutatingWebhookConfiguration -oyaml
- apiVersion: admissionregistration.k8s.io/v1beta1
  kind: MutatingWebhookConfiguration
  metadata:
    name: istio-sidecar-injector
  webhooks:
    - clientConfig:
      service:
        name: istio-sidecar-injector
        namespace: istio-system
        path: /inject
    name: sidecar-injector.istio.io
    namespaceSelector:
      matchLabels:
        istio-injection: enabled
    rules:
    - apiGroups:
      - ""
      apiVersions:
      - v1
      operations:
      - CREATE
      resources:
      - pods
```

该配置告诉 kube-apiserver: 命名空间 istio-system 中的服务 `istio-sidecar-injector`(默认 443 端口), 通过路由`/inject`, 处理`v1/pods`的 CREATE, 同时 pod 需要满足命名空间`istio-injection: enabled`, 当有符合条件的 pod 被创建时, kube-apiserver 就会对该服务发起调用, 服务返回的内容正是添加了 sidecar 注入的 pod 定义。

### 5.2.2 Sidecar 注入内容分析

查看 Pod `istio-sidecar-injector`的 yaml 定义:

```yaml
% kubectl -n istio-system get pod istio-sidecar-injector-5f7894f54f-w7f9v -oyaml
......
    volumeMounts:
    - mountPath: /etc/istio/inject
      name: inject-config
      readOnly: true

  volumes:
  - configMap:
      items:
      - key: config
        path: config
      name: istio-sidecar-injector
    name: inject-config
```

可以看到该 Pod 利用[projected volume](https://kubernetes.io/docs/concepts/storage/volumes/#projected)将`istio-sidecar-injector`这个 config map 的 config 挂到了自己容器路径`/etc/istio/inject/config`, 该 config map 内容正是注入用户空间 pod 所需的模板。

如果使用 helm 安装 istio, 该 configMap 模板源码位于: https://github.com/istio/istio/blob/master/install/kubernetes/helm/istio/templates/sidecar-injector-configmap.yaml。

该 config map 是在安装 istio 时添加的, kubernetes 会自动维护 projected volume 的更新, 因此 容器 `sidecar-injector`只需要从本地文件直接读取所需配置。

高级用户可以按需修改这个模板内容。

```bash
kubectl -n istio-system get configmap istio-sidecar-injector -o=jsonpath='{.data.config}'
```

查看该 configMap, `data.config`包含以下内容(简化):

```yaml
policy: enabled # 是否开启自动注入
template: |-    # 使用go template 定义的pod patch
  initContainers:
  [[ if ne (annotation .ObjectMeta `sidecar.istio.io/interceptionMode` .ProxyConfig.InterceptionMode) "NONE" ]]
  - name: istio-init
    image: "docker.io/istio/proxy_init:1.1.0"
    ......
    securityContext:
      capabilities:
        add:
        - NET_ADMIN
    ......
  containers:
  - name: istio-proxy
    args:
    - proxy
    - sidecar
    ......
    image: [[ annotation .ObjectMeta `sidecar.istio.io/proxyImage`  "docker.io/istio/proxyv2:1.1.0"  ]]
    ......
    readinessProbe:
      httpGet:
        path: /healthz/ready
        port: [[ annotation .ObjectMeta `status.sidecar.istio.io/port`  0  ]]
    ......
    securityContext:
      capabilities:
        add:
        - NET_ADMIN
      runAsGroup: 1337
  ......
    volumeMounts:
    ......
    - mountPath: /etc/istio/proxy
      name: istio-envoy
    - mountPath: /etc/certs/
      name: istio-certs
      readOnly: true
      ......
  volumes:
  ......
  - emptyDir:
      medium: Memory
    name: istio-envoy
  - name: istio-certs
    secret:
      optional: true
      [[ if eq .Spec.ServiceAccountName "" -]]
      secretName: istio.default
      [[ else -]]
      secretName: [[ printf "istio.%s" .Spec.ServiceAccountName ]]
      ......
```

对 istio-init 生成的部分参数分析:

- `-u 1337` 排除用户 ID 为 1337，即 Envoy 自身的流量
- 解析用户容器`.Spec.Containers`, 获得容器的端口列表, 传入`-b`参数(入站端口控制)
- 指定要从重定向到 Envoy 中排除（可选）的入站端口列表, 默认写入`-d 15020`, 此端口是 sidecar 的 status server
- 赋予该容器`NET_ADMIN` 能力, 允许容器 istio-init 进行网络管理操作

对 istio-proxy 生成的部分参数分析:

- 启动参数`proxy sidecar xxx` 用以定义该节点的代理类型(NodeType)
- 默认的 status server 端口`--statusPort=15020`
- 解析用户容器`.Spec.Containers`, 获取用户容器的 application Ports, 然后设置到 sidecar 的启动参数`--applicationPorts`中, 该参数会最终传递给 envoy, 用以确定哪些端口流量属于该业务容器。
- 设置`/healthz/ready` 作为该代理的 readinessProbe
- 同样赋予该容器`NET_ADMIN`能力

另外`istio-sidecar-injector`还给容器`istio-proxy`挂了 2 个 volumes:

- 名为`istio-envoy`的 emptydir volume, 挂载到容器目录`/etc/istio/proxy`, 作为 envoy 的配置文件目录

- 名为`istio-certs`的 secret volume, 默认 secret 名为`istio.default`, 挂载到容器目录`/etc/certs/`, 存放相关的证书, 包括服务端证书, 和可能的 mtls 客户端证书

  ```bash
  % kubectl exec productpage-v1-6597cb5df9-xlndw -c istio-proxy -- ls /etc/certs/
  cert-chain.pem
  key.pem
  root-cert.pem
  ```

# 六 流量管理

## 6.1 流量管理

这一章节将带大家了解 Istio 流量管理中的各种概念的含义及表示方法。

流量管理是 Istio 中的最基础功能，使用 Istio 的流量管理模型，本质上是将流量与基础设施扩容解耦，让运维人员可以通过 Pilot 指定流量遵循什么规则，而不是指定哪些 pod/VM 应该接收流量——Pilot 和智能 Envoy 代理会帮我们搞定。

所谓流量管理是指：

- **控制服务之间的路由**：通过在 `VirtualService` 中的规则条件匹配来设置路由，可以在服务间拆分流量。
- **控制路由上流量的行为**：设定好路由之后，就可以在路由上指定超时和重试机制，例如超时时间、重试次数等；做错误注入、设置断路器等。可以由 `VirtualService` 和 `DestinationRule` 共同完成。
- **显式地向网格中注册服务**：显示地引入 Service Mesh 内部或外部的服务，纳入服务网格管理。由 `ServiceEntry` 实现。
- **控制网格边缘的南北向流量**：为了管理进入 Istio service mesh 的南北向入口流量，需要创建 `Gateway` 对象并与 `VirtualService` 绑定。

关于流量管理的详细介绍请参考 [Istio 官方文档](https://istio.io/zh/docs/concepts/traffic-management/)。

## 6.2 流量管理基础概念

下面将带您了解 Istio 流量管理相关的基础概念与配置示例。

- [`VirtualService`](https://preliminary.istio.io/zh/docs/reference/config/istio.networking.v1alpha3/#virtualservice) 在 Istio 服务网格中定义路由规则，控制流量路由到服务上的各种行为。
- [`DestinationRule`](https://preliminary.istio.io/zh/docs/reference/config/istio.networking.v1alpha3/#destinationrule) 是 `VirtualService` 路由生效后，配置应用与请求的策略集。
- [`ServiceEntry`](https://preliminary.istio.io/zh/docs/reference/config/istio.networking.v1alpha3/#serviceentry) 通常用于在 Istio 服务网格之外启用的服务请求。
- [`Gateway`](https://preliminary.istio.io/zh/docs/reference/config/istio.networking.v1alpha3/#gateway) 为 HTTP/TCP 流量配置负载均衡器，最常见的是在网格边缘的操作，以启用应用程序的入口流量。
- [`EnvoyFilter`](https://preliminary.istio.io/zh/docs/reference/config/istio.networking.v1alpha3/#envoyfilter) 描述了针对代理服务的过滤器，用来定制由 Istio Pilot 生成的代理配置。一定要谨慎使用此功能。错误的配置内容一旦完成传播，可能会令整个服务网格陷入瘫痪状态。这一配置是用于对 Istio 网络系统内部实现进行变更的。

**注**：本文中的示例引用自 Istio 官方 Bookinfo 示例，见：[Istio 代码库](https://github.com/istio/istio/tree/master/samples/bookinfo/)，且对于配置的讲解都以在 Kubernetes 中部署的服务为准。

### 6.2.1 VirtualService

`VirtualService` 故名思义，就是虚拟服务，在 Istio 1.0 以前叫做 RouteRule。`VirtualService` 中定义了一系列针对指定服务的流量路由规则。每个路由规则都是针对特定协议的匹配规则。如果流量符合这些特征，就会根据规则发送到服务注册表中的目标服务（或者目标服务的子集或版本）。VirtualService 的详细定义和配置请参考[通信路由](https://preliminary.istio.io/zh/docs/reference/config/istio.networking.v1alpha3/#virtualservice)。

**注意**：`VirtualService` 中的规则是按照在 YAML 文件中的顺序执行的，这就是为什么在存在多条规则时，需要慎重考虑优先级的原因。

**配置说明**

下面是 `VirtualService` 的配置说明。

| 字段       | 类型          | 描述                                                         |
| ---------- | ------------- | ------------------------------------------------------------ |
| `hosts`    | `string[]`    | 必要字段：流量的目标主机。可以是带有通配符前缀的 DNS 名称，也可以是 IP 地址。根据所在平台情况，还可能使用短名称来代替 FQDN。这种场景下，短名称到 FQDN 的具体转换过程是要靠下层平台完成的。**一个主机名只能在一个 VirtualService 中定义。**同一个 `VirtualService` 中可以用于控制多个 HTTP 和 TCP 端口的流量属性。Kubernetes 用户注意：当使用服务的短名称时（例如使用 `reviews`，而不是 `reviews.default.svc.cluster.local`），Istio 会根据规则所在的命名空间来处理这一名称，而非服务所在的命名空间。假设 “default” 命名空间的一条规则中包含了一个 `reviews`的 `host` 引用，就会被视为 `reviews.default.svc.cluster.local`，而不会考虑 `reviews` 服务所在的命名空间。**为了避免可能的错误配置，建议使用 FQDN 来进行服务引用。** `hosts` 字段对 HTTP 和 TCP 服务都是有效的。网格中的服务也就是在服务注册表中注册的服务，必须使用他们的注册名进行引用；只有 `Gateway` 定义的服务才可以使用 IP 地址。 |
| `gateways` | `string[]`    | `Gateway` 名称列表，Sidecar 会据此使用路由。`VirtualService`对象可以用于网格中的 Sidecar，也可以用于一个或多个 `Gateway`。这里公开的选择条件可以在协议相关的路由过滤条件中进行覆盖。保留字 `mesh` 用来指代网格中的所有 Sidecar。当这一字段被省略时，就会使用缺省值（`mesh`），也就是针对网格中的所有 Sidecar 生效。如果提供了 `gateways` 字段，这一规则就只会应用到声明的 `Gateway` 之中。要让规则同时对 `Gateway` 和网格内服务生效，需要显式的将 `mesh` 加入 `gateways` 列表。 |
| `http`     | `HTTPRoute[]` | HTTP 流量规则的有序列表。这个列表对名称前缀为 `http-`、`http2-`、`grpc-` 的服务端口，或者协议为 `HTTP`、`HTTP2`、`GRPC` 以及终结的 TLS，另外还有使用 `HTTP`、`HTTP2` 以及 `GRPC` 协议的 `ServiceEntry` 都是有效的。进入流量会使用匹配到的第一条规则。 |
| `tls`      | `TLSRoute[]`  | 一个有序列表，对应的是透传 TLS 和 HTTPS 流量。路由过程通常利用 `ClientHello` 消息中的 SNI 来完成。TLS 路由通常应用在 `https-`、`tls-` 前缀的平台服务端口，或者经 `Gateway` 透传的 HTTPS、TLS 协议端口，以及使用 HTTPS 或者 TLS 协议的 `ServiceEntry` 端口上。**注意：没有关联 VirtualService 的 https- 或者 tls- 端口流量会被视为透传 TCP 流量。** |
| `tcp`      | `TCPRoute[]`  | 一个针对透传 TCP 流量的有序路由列表。TCP 路由对所有 HTTP 和 TLS 之外的端口生效。进入流量会使用匹配到的第一条规则。 |

**示例**

下面的例子中配置了一个名为 `reviews` 的 `VirtualService`，该配置的作用是将所有发送给 `reviews`服务的流量发送到 `v1` 版本的子集。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
```

- 该配置中流量的目标主机是 `reviews`，如果该服务和规则部署在 Kubernetes 的 `default`namespace 下的话，对应于 Kubernetes 中的服务的 DNS 名称就是 `reviews.default.svc.cluster.local`。
- 我们在 `hosts` 配置了服务的名字只是表示该配置是针对 `reviews.default.svc.cluster.local` 的服务的路由规则，但是具体将对该服务的访问的流量路由到哪些服务的哪些实例上，就是要通过 `destination` 的配置了。
- 我们看到上面的 `VirtualService` 的 HTTP 路由中还定义了一个 `destination`。`destination` 用于定义在网络中可寻址的服务，请求或连接在经过路由规则的处理之后，就会被发送给 `destination`。`destination.host` 应该明确指向服务注册表中的一个服务。Istio 的服务注册表除包含平台服务注册表中的所有服务（例如 Kubernetes 服务、Consul 服务）之外，还包含了 [`ServiceEntry`](https://preliminary.istio.io/zh/docs/reference/config/istio.networking.v1alpha3/#serviceentry) 资源所定义的服务。`VirtualService` 中只定义流量发送给哪个服务的路由规则，但是并不知道要发送的服务的地址是什么，这就需要 `DestinationRule` 来定义了。
- `subset` 配置流量目的地的子集，下文会讲到。`VirtualService` 中其实可以除了 `hosts` 字段外其他什么都不配置，路由规则可以在 `DestinationRule` 中单独配置来覆盖此处的默认规则。

### 6.2.3 Subset

`subset` 不属于 Istio 创建的 CRD，但是它是一条重要的配置信息，有必要单独说明下。`subset` 是服务端点的集合，可以用于 A/B 测试或者分版本路由等场景。参考 [`VirtualService`](https://preliminary.istio.io/zh/docs/reference/config/istio.networking.v1alpha3/#virtualservice) 文档，其中会有更多这方面应用的例子。另外在 `subset` 中可以覆盖服务级别的即 `VirtualService` 中的定义的流量策略。

以下是`subset` 的配置信息。对于 Kubernetes 中的服务，一个 `subset` 相当于使用 label 的匹配条件选出来的 `service`。

| 字段            | 类型                                                         | 描述                                                         |
| --------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `name`          | `string`                                                     | 必要字段。服务名和 `subset` 名称可以用于路由规则中的流量拆分。 |
| `labels`        | `map<string,string>`                                         | 必要字段。使用标签对服务注册表中的服务端点进行筛选。         |
| `trafficPolicy` | [`TrafficPolicy`](https://preliminary.istio.io/zh/docs/reference/config/istio.networking.v1alpha3/#trafficpolicy) | 应用到这一 `subset` 的流量策略。缺省情况下 `subset` 会继承 `DestinationRule` 级别的策略，这一字段的定义则会覆盖缺省的继承策略。 |

### 6.2.4 DestinationRule

`DestinationRule` 所定义的策略，决定了经过路由处理之后的流量的访问策略。这些策略中可以定义负载均衡配置、连接池大小以及外部检测（用于在负载均衡池中对不健康主机进行识别和驱逐）配置。

**配置说明**

下面是 `DestinationRule` 的配置说明。

| 字段            | 类型                                                         | 描述                                                         |
| --------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `name`          | `string`                                                     | 必要字段。服务名和 `subset` 名称可以用于路由规则中的流量拆分。 |
| `labels`        | `map<string,string>`                                         | 必要字段。使用标签对服务注册表中的服务端点进行筛选。         |
| `trafficPolicy` | [`TrafficPolicy`](https://preliminary.istio.io/zh/docs/reference/config/istio.networking.v1alpha3/#trafficpolicy) | 应用到这一子集的流量策略。缺省情况下子集会继承 `DestinationRule` 级别的策略，这一字段的定义则会覆盖缺省的继承策略。 |

**示例**

下面是一条对 `productpage` 服务的流量目的地策略的配置。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  subsets:
  - name: v1
    labels:
      version: v1
```

该路由策略将所有对 `reviews` 服务的流量路由到 `v1` 的 subset。

### 6.2.5 ServiceEntry

Istio 服务网格内部会维护一个与平台无关的使用通用模型表示的服务注册表，当你的服务网格需要访问外部服务的时候，就需要使用 `ServiceEntry` 来添加服务注册。

### 6.2.6 EnvoyFilter

[`EnvoyFilter`](https://preliminary.istio.io/zh/docs/reference/config/istio.networking.v1alpha3/#envoyfilter) 描述了针对代理服务的过滤器，用来定制由 Istio Pilot 生成的代理配置。一定要谨慎使用此功能。错误的配置内容一旦完成传播，可能会令整个服务网格陷入瘫痪状态。这一配置是用于对 Istio 网络系统内部实现进行变更的，属于高级配置，用于扩展 Envoy 中的过滤器的。

### 6.2.7 Gateway

[Gateway](https://preliminary.istio.io/zh/docs/reference/config/istio.networking.v1alpha3/#gateway) 为 HTTP/TCP 流量配置了一个负载均衡，多数情况下在网格边缘进行操作，用于启用一个服务的入口（ingress）流量，相当于前端代理。与 Kubernetes 的 Ingress 不同，Istio `Gateway` 只配置四层到六层的功能（例如开放端口或者 TLS 配置），而 Kubernetes 的 Ingress 是七层的。将 `VirtualService` 绑定到 `Gateway` 上，用户就可以使用标准的 Istio 规则来控制进入的 HTTP 和 TCP 流量。

Gateway 设置了一个集群外部流量访问集群中的某些服务的入口，而这些流量究竟如何路由到那些服务上则需要通过配置 `VirtualServcie` 来绑定。下面仍然以 `productpage` 这个服务来说明。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: bookinfo-gateway
spec:
  selector:
    istio: ingressgateway # 使用默认的控制器
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo
spec:
  hosts:
  - "*"
  gateways:
  - bookinfo-gateway
  http:
  - match:
    - uri:
        exact: /productpage
    - uri:
        exact: /login
    - uri:
        exact: /logout
    - uri:
        prefix: /api/v1/products
    route:
    - destination:
        host: productpage
        port:
          number: 9080
```

上面的例子中 `bookinfo` 这个 `VirtualService` 中绑定到了 `bookinfo-gateway`。`bookinfo-gateway`使用了标签选择器选择对应的 Kubernetes pod，即下图中的 pod。

[![istio ingress gateway pod](https://www.servicemesher.com/istio-handbook/images/0069RVTdgy1fv7xh71h8fj31fn0dyq9g.jpg)](https://www.servicemesher.com/istio-handbook/images/0069RVTdgy1fv7xh71h8fj31fn0dyq9g.jpg)图片 - istio ingress gateway pod

我们再看下 `istio-ingressgateway` 的 YAML 安装配置。

```yaml
# Deployment 配置
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: istio-ingressgateway
  namespace: istio-system
  labels:
    app: ingressgateway
    chart: gateways-1.0.0
    release: RELEASE-NAME
    heritage: Tiller
    app: istio-ingressgateway
    istio: ingressgateway
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: istio-ingressgateway
        istio: ingressgateway
      annotations:
        sidecar.istio.io/inject: "false"
        scheduler.alpha.kubernetes.io/critical-pod: ""
    spec:
      serviceAccountName: istio-ingressgateway-service-account
      containers:
        - name: ingressgateway
          image: "gcr.io/istio-release/proxyv2:1.0.0" # 容器启动命令入口是 /usr/local/bin/pilot-agent，后面跟参数 proxy 就会启动一个 Envoy 进程
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
            - containerPort: 443
            - containerPort: 31400
            - containerPort: 15011
            - containerPort: 8060
            - containerPort: 15030
            - containerPort: 15031
          args:
          - proxy
          - router
          - -v
          - "2"
          - --discoveryRefreshDelay
          - '1s' #discoveryRefreshDelay
          - --drainDuration
          - '45s' #drainDuration
          - --parentShutdownDuration
          - '1m0s' #parentShutdownDuration
          - --connectTimeout
          - '10s' #connectTimeout
          - --serviceCluster
          - istio-ingressgateway
          - --zipkinAddress
          - zipkin:9411
          - --statsdUdpAddress
          - istio-statsd-prom-bridge:9125
          - --proxyAdminPort
          - "15000"
          - --controlPlaneAuthPolicy
          - NONE
          - --discoveryAddress
          - istio-pilot.istio-system:8080
          resources:
            requests:
              cpu: 10m           
          env:
          - name: POD_NAME
            valueFrom:
              fieldRef:
                apiVersion: v1
                fieldPath: metadata.name
          - name: POD_NAMESPACE
            valueFrom:
              fieldRef:
                apiVersion: v1
                fieldPath: metadata.namespace
          - name: INSTANCE_IP
            valueFrom:
              fieldRef:
                apiVersion: v1
                fieldPath: status.podIP
          - name: ISTIO_META_POD_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          volumeMounts:
            ...
# 服务配置
---
apiVersion: v1
kind: Service
metadata:
  name: istio-ingressgateway
  namespace: istio-system
  annotations:
  labels:
    chart: gateways-1.0.0
    release: RELEASE-NAME
    heritage: Tiller
    app: istio-ingressgateway
    istio: ingressgateway
spec:
  type: NodePort
  selector:
    app: istio-ingressgateway
    istio: ingressgateway
  ports:
    - name: http2 # 将 ingressgateway 的 80 端口映射到节点的 31380 端口以代理 HTTP 请求
      nodePort: 31380
      port: 80
      targetPort: 80
    - name: https
      nodePort: 31390
      port: 443
    - name: tcp
      nodePort: 31400
      port: 31400
    - name: tcp-pilot-grpc-tls
      port: 15011
      targetPort: 15011
    - name: tcp-citadel-grpc-tls
      port: 8060
      targetPort: 8060
    - name: http2-prometheus
      port: 15030
      targetPort: 15030
    - name: http2-grafana
      port: 15031
      targetPort: 15031
```

我们看到 `ingressgateway` 使用的是 `proxyv2` 镜像，该镜像容器的启动命令入口是 `/usr/local/bin/pilot-agent`，后面跟参数 proxy 就会启动一个 Envoy 进程，因此 Envoy 既作为 sidecar 也作为边缘代理，`egressgateway` 的情况也是类似，只不过它控制的是集群内部对外集群外部的请求。这正好验证了本文开头中所画的 Istio Pilot 架构图。请求 `/productpage`、`/login`、`/logout`、`/api/v1/products` 这些 URL 的流量转发给 `productpage` 服务的 9080 端口，而这些流量进入集群内又是经过 `ingressgateway` pod 代理的，通过访问 `ingressgateway` pod 所在的宿主机的 31380 端口进入集群内部的。



示例

我们以官方的 `bookinfo` 示例来解析流量管理配置。下图是 VirtualService 和 DestinationRule 的示意图，其中只显示了 `productpage` 和 `reviews` 服务。

[![VirtualSerivce 和 DestimationRule 示意图](https://www.servicemesher.com/istio-handbook/images/istio-virtualservice-and-destinationrule-illustration.png)](https://www.servicemesher.com/istio-handbook/images/istio-virtualservice-and-destinationrule-illustration.png)图片 - VirtualSerivce 和 DestimationRule 示意图

在前提条件中我部署了该示例，并列出了该示例中的所有 pod，现在我们使用 [istioctl](https://preliminary.istio.io/zh/docs/reference/commands/istioctl) 命令来启动查看 `productpage-v1-745ffc55b7-2l2lw` pod 中的流量配置。

**查看 pod 中 Envoy sidecar 的启动配置信息**

[Bootstrap](https://www.envoyproxy.io/docs/envoy/latest/api-v2/config/bootstrap/v2/bootstrap.proto.html#envoy-api-msg-config-bootstrap-v2-bootstrap) 消息是 Envoy 配置的根本来源，[Bootstrap](https://www.envoyproxy.io/docs/envoy/latest/api-v2/config/bootstrap/v2/bootstrap.proto.html#envoy-api-msg-config-bootstrap-v2-bootstrap) 消息的一个关键的概念是静态和动态资源的之间的区别。例如 [Listener](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/lds.proto.html#envoy-api-msg-listener) 或 [Cluster](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/cds.proto.html#envoy-api-msg-cluster) 这些资源既可以从 [static_resources](https://www.envoyproxy.io/docs/envoy/latest/api-v2/config/bootstrap/v2/bootstrap.proto.html#envoy-api-field-config-bootstrap-v2-bootstrap-static-resources) 静态的获得也可以从 [dynamic_resources](https://www.envoyproxy.io/docs/envoy/latest/api-v2/config/bootstrap/v2/bootstrap.proto.html#envoy-api-field-config-bootstrap-v2-bootstrap-dynamic-resources)中配置的 LDS 或 CDS 之类的 xDS 服务获取。关于 xDS 服务的详解请参考 [Envoy 中的 xDS REST 和 gRPC 协议详解](http://www.servicemesher.com/blog/envoy-xds-protocol/)。

```bash
$ istioctl proxy-config bootstrap productpage-v1-745ffc55b7-2l2lw -o json
{
    "bootstrap": {
        "node": {
            "id": "sidecar~172.33.78.10~productpage-v1-745ffc55b7-2l2lw.default~default.svc.cluster.local",
            "cluster": "productpage",
            "metadata": {
                    "INTERCEPTION_MODE": "REDIRECT",
                    "ISTIO_PROXY_SHA": "istio-proxy:6166ae7ebac7f630206b2fe4e6767516bf198313",
                    "ISTIO_PROXY_VERSION": "1.0.0",
                    "ISTIO_VERSION": "1.0.0",
                    "POD_NAME": "productpage-v1-745ffc55b7-2l2lw",
                    "istio": "sidecar"
                },
            "buildVersion": "0/1.8.0-dev//RELEASE"
        },
        "staticResources": { # Envoy 的静态配置，除非销毁后重设，否则不会改变，配置中会明确指定每个上游主机的已解析网络名称（ IP 地址、端口、unix 域套接字等）。
            "clusters": [
                {
                    "name": "xds-grpc",
                    "type": "STRICT_DNS",
                    "connectTimeout": "10.000s",
                    "hosts": [
                        {    # istio-pilot 的地址，指定控制平面地址，这个必须是通过静态的方式配置的
                            "socketAddress": {
                                "address": "istio-pilot.istio-system",
                                "portValue": 15010
                            }
                        }
                    ],
                    "circuitBreakers": { # 断路器配置
                        "thresholds": [
                            {
                                "maxConnections": 100000,
                                "maxPendingRequests": 100000,
                                "maxRequests": 100000
                            },
                            {
                                "priority": "HIGH",
                                "maxConnections": 100000,
                                "maxPendingRequests": 100000,
                                "maxRequests": 100000
                            }
                        ]
                    },
                    "http2ProtocolOptions": {

                    },
                    "upstreamConnectionOptions": { # 上游连接选项
                        "tcpKeepalive": {
                            "keepaliveTime": 300
                        }
                    }
                },
                { # zipkin 分布式追踪地址配置
                    "name": "zipkin",
                    "type": "STRICT_DNS",
                    "connectTimeout": "1.000s",
                    "hosts": [
                        {
                            "socketAddress": {
                                "address": "zipkin.istio-system",
                                "portValue": 9411
                            }
                        }
                    ]
                }
            ]
        }, # 以下是动态配置
        "dynamicResources": {
            "ldsConfig": { # Listener Discovery Service 配置，直接使用 ADS 配置，此处不用配置
                "ads": {

                }
            },
            "cdsConfig": { # Cluster Discovery Service 配置，直接使用 ADS 配置，此处不用配置
                "ads": {

                }
            },
            "adsConfig": { # Aggregated Discovery Service 配置，ADS 中集成了 LDS、RDS、CDS
                "apiType": "GRPC",
                "grpcServices": [
                    {
                        "envoyGrpc": {
                            "clusterName": "xds-grpc"
                        }
                    }
                ],
                "refreshDelay": "1.000s"
            }
        },
        "statsSinks": [ # metric 汇聚的地址
            {
                "name": "envoy.statsd",
                "config": {
                        "address": {
                                    "socket_address": {
                                                "address": "10.254.109.175",
                                                "port_value": 9125
                                            }
                                }
                    }
            }
        ],
        "statsConfig": {
            "useAllDefaultTags": false
        },
        "tracing": { # zipkin 地址
            "http": {
                "name": "envoy.zipkin",
                "config": {
                        "collector_cluster": "zipkin"
                    }
            }
        },
        "admin": {
            "accessLogPath": "/dev/stdout",
            "address": {
                "socketAddress": {
                    "address": "127.0.0.1",
                    "portValue": 15000
                }
            }
        }
    },
    "lastUpdated": "2018-09-04T03:38:45.645Z"
}
```

以上为初始信息。

创建一个名为 `reviews` 的 `VirtualService`。

```bash
$ cat<<EOF | istioctl create -f -
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
EOF
```

上面的 `VirtualService` 定义只是定义了访问 `reviews` 服务的流量要全部流向 `reviews`服务的 `v1`子集，至于哪些实例是 `v1` 子集，`VirtualService` 中并没有定义，这就需要再创建个 `DestinationRule`。

```bash
$ cat <<EOF | istioctl create -f -
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  subsets:
  - name: v1
    labels:
      version: v1
EOF
```

同时还可以为每个 `subset` 设置负载均衡规则。这里面也可以同时创建多个子集，例如同时创建3个 `subset` 分别对应3个版本的实例。

```yaml
$ cat <<EOF | istioctl create -f -
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
  - name: v3
    labels:
      version: v3
EOF
```

同时配置了三个 `subset` 当你需要切分流量时可以直接修改 `VirtualService` 中 `destination` 里的 `subset` 即可，还可以根据百分比拆分流量，配置超时和重试，进行错误注入等，详见[流量管理](https://istio.io/zh/docs/concepts/traffic-management/)

当然上面这个例子中只是简单的将流量全部导到某个 `VirtualService` 的 `subset` 中，还可以根据其他限定条件如 HTTP headers、pod 的 label、URL 等。

此时再查询 `productpage-v1-745ffc55b7-2l2lw` pod 的配置信息。

```bash
$ istioctl proxy-config clusters productpage-v1-8d69b45c-bcjqv|grep reviews
reviews.default.svc.cluster.local                           9080      -          outbound      EDS
reviews.default.svc.cluster.local                           9080      v1         outbound      EDS
reviews.default.svc.cluster.local                           9080      v2         outbound      EDS
reviews.default.svc.cluster.local                           9080      v3         outbound      EDS
```

可以看到 `reviews` 服务的 EDS 设置中包含了3个 `subset`，另外读者还可以自己运行 `istioctl proxy-config listeners` 和 `istioctl proxy-config route` 来查询 pod 的监听器和路由配置。



## 6.3 Istio 中的 Sidecar 的流量路由详解

本文以 Istio 官方的 [bookinfo 示例](https://preliminary.istio.io/zh/docs/examples/bookinfo)来讲解在进入 Pod 的流量被 iptables 转交给 Envoy sidecar 后，Envoy 是如何做路由转发的，详述了 Inbound 和 Outbound 处理过程。关于流量拦截的详细分析请参考[理解 Istio Service Mesh 中 Envoy 代理 Sidecar 注入及流量劫持](https://jimmysong.io/posts/envoy-sidecar-injection-in-istio-service-mesh-deep-dive/)。

下面是 Istio 官方提供的 bookinfo 的请求流程图，假设 bookinfo 应用的所有服务中没有配置 DestinationRule。

[![Bookinfo 示例](https://www.servicemesher.com/istio-handbook/images/006tNbRwgy1fvlwjd3302j31bo0ro0x5.jpg)](https://www.servicemesher.com/istio-handbook/images/006tNbRwgy1fvlwjd3302j31bo0ro0x5.jpg)图片 - Bookinfo 示例

下面是 Istio 自身组件与 Bookinfo 示例的连接关系图，我们可以看到所有的 HTTP 连接都在 9080 端口监听。

[![Bookinfo 示例与 Istio 组件连接关系图](https://www.servicemesher.com/istio-handbook/images/006tNbRwly1fyitp0jsghj31o70u0x6p.jpg)](https://www.servicemesher.com/istio-handbook/images/006tNbRwly1fyitp0jsghj31o70u0x6p.jpg)图片 - Bookinfo 示例与 Istio 组件连接关系图

可以在 [Google Drive](https://drive.google.com/open?id=19ed3_tkjf6RgGboxllMdt_Ytd5_cocib) 上下载原图。

### 6.3.1 Sidecar 注入及流量劫持步骤概述

下面是从 Sidecar 注入、Pod 启动到 Sidecar proxy 拦截流量及 Envoy 处理路由的步骤概览。

**1.** Kubernetes 通过 Admission Controller 自动注入，或者用户使用 `istioctl` 命令手动注入 sidecar 容器。

**2.** 应用 YAML 配置部署应用，此时 Kubernetes API server 接收到的服务创建配置文件中已经包含了 Init 容器及 sidecar proxy。

**3.** 在 sidecar proxy 容器和应用容器启动之前，首先运行 Init 容器，Init 容器用于设置 iptables（Istio 中默认的流量拦截方式，还可以使用 BPF、IPVS 等方式） 将进入 pod 的流量劫持到 Envoy sidecar proxy。所有 TCP 流量（Envoy 目前只支持 TCP 流量）将被 sidecar 劫持，其他协议的流量将按原来的目的地请求。

**4.** 启动 Pod 中的 Envoy sidecar proxy 和应用程序容器。这一步的过程请参考[通过管理接口获取完整配置](https://zhaohuabing.com/post/2018-09-25-istio-traffic-management-impl-intro/#通过管理接口获取完整配置)。

> **Sidecar proxy 与应用容器的启动顺序问题**
>
> 启动 sidecar proxy 和应用容器，究竟哪个容器先启动呢？正常情况是 Envoy Sidecar 和应用程序容器全部启动完成后再开始接收流量请求。但是我们无法预料哪个容器会先启动，那么容器启动顺序是否会对 Envoy 劫持流量有影响呢？答案是肯定的，不过分为以下两种情况。
>
> **情况1：应用容器先启动，而 sidecar proxy 仍未就绪**
>
> 这种情况下，流量被 iptables 转移到 15001 端口，而 Pod 中没有监听该端口，TCP 链接就无法建立，请求失败。
>
> **情况2：Sidecar 先启动，请求到达而应用程序仍未就绪**
>
> 这种情况下请求也肯定会失败，至于是在哪一步开始失败的，留给读者来思考。

**问题**：如果为 sidecar proxy 和应用程序容器添加[就绪和存活探针](https://jimmysong.io/kubernetes-handbook/guide/configure-liveness-readiness-probes.html)是否可以解决该问题呢？

**5.** 不论是进入还是从 Pod 发出的 TCP 请求都会被 iptables 劫持，inbound 流量被劫持后经 Inbound Handler 处理后转交给应用程序容器处理，outbound 流量被 iptables 劫持后转交给 Outbound Handler 处理，并确定转发的 upstream 和 Endpoint。

**6.** Sidecar proxy 请求 Pilot 使用 xDS 协议同步 Envoy 配置，其中包括 LDS、EDS、CDS 等，不过为了保证更新的顺序，Envoy 会直接使用 ADS 向 Pilot 请求配置更新。

### 6.3.2 Envoy 如何处理路由转发

下图展示的是 `productpage` 服务请求访问 `http://reviews.default.svc.cluster.local:9080/`，当流量进入 `reviews` 服务内部时，`reviews` 服务内部的 Envoy Sidecar 是如何做流量拦截和路由转发的。可以在 [Google Drive](https://drive.google.com/file/d/1n-h235tm8DnL_RqxTTA95rgGtrLkBsyr/view?usp=sharing) 上下载原图。

[![Envoy sidecar 流量劫持与路由转发示意图](https://www.servicemesher.com/istio-handbook/images/006tKfTcly1g13besag8zj31c70u0qkt.jpg)](https://www.servicemesher.com/istio-handbook/images/006tKfTcly1g13besag8zj31c70u0qkt.jpg)图片 - Envoy sidecar 流量劫持与路由转发示意图

第一步开始时，`productpage` Pod 中的 Envoy sidecar 已经通过 EDS 选择出了要请求的 `reviews` 服务的一个 Pod，知晓了其 IP 地址，发送 TCP 连接请求。

Istio 官网中的 [Envoy 配置深度解析](https://preliminary.istio.io/zh/help/ops/traffic-management/proxy-cmd/#envoy-配置深度解析)中是以发起 HTTP 请求的一方来详述 Envoy 做流量转发的过程，而本文中考虑的是接受 downstream 的流量的一方，它既要接收 downstream 发来的请求，自己还需要请求其他服务，例如 `reviews` 服务中的 Pod 还需要请求 `ratings` 服务。

`reviews` 服务有三个版本，每个版本有一个实例，三个版本中的 sidecar 工作步骤类似，下文只以 `reviews-v1-cb8655c75-b97zc` 这一个 Pod 中的 Sidecar 流量转发步骤来说明。

### 6.3.3 理解 Inbound Handler

Inbound handler 的作用是将 iptables 拦截到的 downstream 的流量转交给 localhost，与 Pod 内的应用程序容器建立连接。

查看下 `reviews-v1-cb8655c75-b97zc` pod 中的 Listener。

运行 `istioctl pc listener reviews-v1-cb8655c75-b97zc` 查看该 Pod 中的具有哪些 Listener。

```ini
ADDRESS            PORT      TYPE 
172.33.3.3         9080      HTTP <--- 接收所有 Inbound HTTP 流量，该地址即为当前 Pod 的 IP 地址
10.254.0.1         443       TCP  <--+
10.254.4.253       80        TCP     |
10.254.4.253       8080      TCP     |
10.254.109.182     443       TCP     |
10.254.22.50       15011     TCP     |
10.254.22.50       853       TCP     |
10.254.79.114      443       TCP     | 
10.254.143.179     15011     TCP     |
10.254.0.2         53        TCP     | 接收与 0.0.0.0_15001 监听器配对的 Outbound 非 HTTP 流量
10.254.22.50       443       TCP     |
10.254.16.64       42422     TCP     |
10.254.127.202     16686     TCP     |
10.254.22.50       31400     TCP     |
10.254.22.50       8060      TCP     |
10.254.169.13      14267     TCP     |
10.254.169.13      14268     TCP     |
10.254.32.134      8443      TCP     |
10.254.118.196     443       TCP  <--+
0.0.0.0            15004     HTTP <--+
0.0.0.0            8080      HTTP    |
0.0.0.0            15010     HTTP    | 
0.0.0.0            8088      HTTP    |
0.0.0.0            15031     HTTP    |
0.0.0.0            9090      HTTP    | 
0.0.0.0            9411      HTTP    | 接收与 0.0.0.0_15001 配对的 Outbound HTTP 流量
0.0.0.0            80        HTTP    |
0.0.0.0            15030     HTTP    |
0.0.0.0            9080      HTTP    |
0.0.0.0            9093      HTTP    |
0.0.0.0            3000      HTTP    |
0.0.0.0            8060      HTTP    |
0.0.0.0            9091      HTTP <--+    
0.0.0.0            15001     TCP  <--- 接收所有经 iptables 拦截的 Inbound 和 Outbound 流量并转交给虚拟监听器处理
```

当来自 `productpage` 的流量抵达 `reviews` Pod 的时候已经，downstream 必须明确知道 Pod 的 IP 地址为 `172.33.3.3` 所以才会访问该 Pod，所以该请求是 `172.33.3.3:9080`。

**virtual Listener**

从该 Pod 的 Listener 列表中可以看到，0.0.0.0:15001/TCP 的 Listener（其实际名字是 `virtual`）监听所有的 Inbound 流量，下面是该 Listener 的详细配置。

```json
{
    "name": "virtual",
    "address": {
        "socketAddress": {
            "address": "0.0.0.0",
            "portValue": 15001
        }
    },
    "filterChains": [
        {
            "filters": [
                {
                    "name": "envoy.tcp_proxy",
                    "config": {
                        "cluster": "BlackHoleCluster",
                        "stat_prefix": "BlackHoleCluster"
                    }
                }
            ]
        }
    ],
    "useOriginalDst": true
}
```

**UseOriginalDst**：从配置中可以看出 `useOriginalDst` 配置指定为 `true`，这是一个布尔值，缺省为 false，使用 iptables 重定向连接时，proxy 接收的端口可能与[原始目的地址](http://www.servicemesher.com/envoy/configuration/listener_filters/original_dst_filter.html)的端口不一样，如此处 proxy 接收的端口为 15001，而原始目的地端口为 9080。当此标志设置为 true 时，Listener 将连接重定向到与原始目的地址关联的 Listener，此处为 `172.33.3.3:9080`。如果没有与原始目的地址关联的 Listener，则连接由接收它的 Listener 处理，即该 `virtual` Listener，经过 `envoy.tcp_proxy` 过滤器处理转发给 `BlackHoleCluster`，这个 Cluster 的作用正如它的名字，当 Envoy 找不到匹配的虚拟监听器时，就会将请求发送给它，并返回 404。这个将于下文提到的 Listener 中设置 `bindToPort` 相呼应。

**注意**：该参数将被废弃，请使用[原始目的地址](http://www.servicemesher.com/envoy/configuration/listener_filters/original_dst_filter.html)的 Listener filter 替代。该参数的主要用途是：Envoy 通过监听 15001 端口将 iptables 拦截的流量经由其他 Listener 处理而不是直接转发出去，详情见 [Virtual Listener](https://zhaohuabing.com/post/2018-09-25-istio-traffic-management-impl-intro/#virtual-listener)。

**Listener 172.33.3.3_9080**

上文说到进入 Inbound handler 的流量被 `virtual` Listener 转移到 `172.33.3.3_9080` Listener，我们在查看下该 Listener 配置。

运行 `istioctl pc listener reviews-v1-cb8655c75-b97zc --address 172.33.3.3 --port 9080 -o json` 查看。

```json
[{
    "name": "172.33.3.3_9080",
    "address": {
        "socketAddress": {
            "address": "172.33.3.3",
            "portValue": 9080
        }
    },
    "filterChains": [
        {
            "filterChainMatch": {
                "transportProtocol": "raw_buffer"
            },
            "filters": [
                {
                    "name": "envoy.http_connection_manager",
                    "config": {
                        ... 
                        "route_config": {
                            "name": "inbound|9080||reviews.default.svc.cluster.local",
                            "validate_clusters": false,
                            "virtual_hosts": [
                                {
                                    "domains": [
                                        "*"
                                    ],
                                    "name": "inbound|http|9080",
                                    "routes": [
                                        {
                                            ...
                                            "route": {
                                                "cluster": "inbound|9080||reviews.default.svc.cluster.local",
                                                "max_grpc_timeout": "0.000s",
                                                "timeout": "0.000s"
                                            }
                                        }
                                    ]
                                }
                            ]
                        },
                        "use_remote_address": false,
                        ...
                    }
                }
            ]，
            "deprecatedV1": {
                "bindToPort": false
            }
        ...
        },
        {
            "filterChainMatch": {
                "transportProtocol": "tls"
            },
            "tlsContext": {...
            },
            "filters": [...
            ]
        }
    ],
...
}]
```

**bindToPort**：注意其中有一个 [`bindToPort`](https://www.envoyproxy.io/docs/envoy/v1.6.0/api-v1/listeners/listeners) 的配置，其值为 `false`，该配置的缺省值为 `true`，表示将 Listener 绑定到端口上，此处设置为 `false` 则该 Listener 只能处理其他 Listener 转移过来的流量，即上文所说的 `virtual` Listener，我们看其中的 filterChains.filters 中的 `envoy.http_connection_manager` 配置部分：

```json
"route_config": {
                            "name": "inbound|9080||reviews.default.svc.cluster.local",
                            "validate_clusters": false,
                            "virtual_hosts": [
                                {
                                    "domains": [
                                        "*"
                                    ],
                                    "name": "inbound|http|9080",
                                    "routes": [
                                        {
                                            ...
                                            "route": {
                                                "cluster": "inbound|9080||reviews.default.svc.cluster.local",
                                                "max_grpc_timeout": "0.000s",
                                                "timeout": "0.000s"
                                            }
                                        }
                                    ]
                                }
                            ]
                        }
```

该配置表示流量将转交给 Cluster `inbound|9080||reviews.default.svc.cluster.local` 处理。

**Cluster inbound|9080||reviews.default.svc.cluster.local**

运行 `istioctl pc cluster reviews-v1-cb8655c75-b97zc --fqdn reviews.default.svc.cluster.local --direction inbound -o json` 查看该 Cluster 的配置如下。

```json
[
    {
        "name": "inbound|9080||reviews.default.svc.cluster.local",
        "connectTimeout": "1.000s",
        "hosts": [
            {
                "socketAddress": {
                    "address": "127.0.0.1",
                    "portValue": 9080
                }
            }
        ],
        "circuitBreakers": {
            "thresholds": [
                {}
            ]
        }
    }
]
```

可以看到该 Cluster 的 Endpoint 直接对应的就是 localhost，再经过 iptables 转发流量就被应用程序容器消费了。

### 6.3.4 理解 Outbound Handler

因为 `reviews` 会向 `ratings` 服务发送 HTTP 请求，请求的地址是：`http://ratings.default.svc.cluster.local:9080/`，Outbound handler 的作用是将 iptables 拦截到的本地应用程序发出的流量，经由 Envoy 判断如何路由到 upstream。

应用程序容器发出的请求为 Outbound 流量，被 iptables 劫持后转移给 Envoy Outbound handler 处理，然后经过 `virtual` Listener、`0.0.0.0_9080` Listener，然后通过 Route 9080 找到 upstream 的 cluster，进而通过 EDS 找到 Endpoint 执行路由动作。这一部分可以参考 Istio 官网中的 [Envoy 深度配置解析](https://preliminary.istio.io/zh/help/ops/traffic-management/proxy-cmd/#envoy-配置深度解析)。

**Route 9080**

`reviews` 会请求 `ratings` 服务，运行 `istioctl proxy-config routes reviews-v1-cb8655c75-b97zc --name 9080 -o json` 查看 route 配置，因为 Envoy 会根据 HTTP header 中的 domains 来匹配 VirtualHost，所以下面只列举了 `ratings.default.svc.cluster.local:9080` 这一个 VirtualHost。

```json
[{
    "name": "ratings.default.svc.cluster.local:9080",
    "domains": [
        "ratings.default.svc.cluster.local",
        "ratings.default.svc.cluster.local:9080",
        "ratings",
        "ratings:9080",
        "ratings.default.svc.cluster",
        "ratings.default.svc.cluster:9080",
        "ratings.default.svc",
        "ratings.default.svc:9080",
        "ratings.default",
        "ratings.default:9080",
        "10.254.234.130",
        "10.254.234.130:9080"
    ],
    "routes": [
        {
            "match": {
                "prefix": "/"
            },
            "route": {
                "cluster": "outbound|9080||ratings.default.svc.cluster.local",
                "timeout": "0.000s",
                "maxGrpcTimeout": "0.000s"
            },
            "decorator": {
                "operation": "ratings.default.svc.cluster.local:9080/*"
            },
            "perFilterConfig": {...
            }
        }
    ]
},
..]
```

从该 Virtual Host 配置中可以看到将流量路由到 Cluster `outbound|9080||ratings.default.svc.cluster.local`。

**Endpoint outbound|9080||ratings.default.svc.cluster.local**

Istio 1.1 以前版本不支持使用 `istioctl` 命令直接查询 Cluster 的 Endpoint，可以使用查询 Pilot 的 debug 端点的方式折中。

```bash
kubectl exec reviews-v1-cb8655c75-b97zc -c istio-proxy curl http://istio-pilot.istio-system.svc.cluster.local:9093/debug/edsz > endpoints.json
```

`endpoints.json` 文件中包含了所有 Cluster 的 Endpoint 信息，我们只选取其中的 `outbound|9080||ratings.default.svc.cluster.local` Cluster 的结果如下。

```json
{
  "clusterName": "outbound|9080||ratings.default.svc.cluster.local",
  "endpoints": [
    {
      "locality": {

      },
      "lbEndpoints": [
        {
          "endpoint": {
            "address": {
              "socketAddress": {
                "address": "172.33.100.2",
                "portValue": 9080
              }
            }
          },
          "metadata": {
            "filterMetadata": {
              "istio": {
                  "uid": "kubernetes://ratings-v1-8558d4458d-ns6lk.default"
                }
            }
          }
        }
      ]
    }
  ]
}
```

Endpoint 可以是一个或多个，Envoy 将根据一定规则选择适当的 Endpoint 来路由。

**注**：Istio 1.1 将支持 `istioctl pc endpoint` 命令来查询 Endpoint。



## 6.4 熔断与异常检测在 Istio 中的应用

在微服务领域，各个服务需要在网络上执行大量的调用。而网络是很脆弱的，如果某个服务繁忙或者无法响应请求，将有可能引发集群的大规模级联故障，从而造成整个系统不可用，通常把这种现象称为服务雪崩效应。为了使服务有一定的冗余，以便在系统故障期间能够保持服务能力，我们可以使用熔断机制。

### 6.4.1 什么是熔断？

熔断（Circuit Breaking）这一概念来源于电子工程中的**断路器**（Circuit Breaker）。在互联网系统中，当下游服务因访问压力过大而响应变慢或失败，上游服务为了保护系统整体的可用性，可以暂时切断对下游服务的调用。这种牺牲局部，保全整体的措施就叫做**熔断**。

如果不采取熔断措施，我们的系统会怎样呢？我们来看一个栗子。

当前系统中有 A、B、C 三个服务，服务 A 是上游，服务 B 是中游，服务 C 是下游。它们的调用链如下：

[![img](https://www.servicemesher.com/istio-handbook/images/006tKfTcgy1g126lbd2g3j304r07jq31.jpg)](https://www.servicemesher.com/istio-handbook/images/006tKfTcgy1g126lbd2g3j304r07jq31.jpg)

一旦下游服务 C 因某些原因变得不可用，积压了大量请求，服务 B 的请求线程也随之阻塞。线程资源逐渐耗尽，使得服务 B 也变得不可用。紧接着，服务 A 也变为不可用，整个调用链路被拖垮。

[![img](https://www.servicemesher.com/istio-handbook/images/006tNc79gy1g1s550xmqej30sz0agdgw.jpg)](https://www.servicemesher.com/istio-handbook/images/006tNc79gy1g1s550xmqej30sz0agdgw.jpg)

像这种调用链路的连锁故障，就是上文所说的服务雪崩效应。

正所谓刮骨疗毒，壮士断腕。在这种时候，就需要我们的熔断机制来挽救整个系统。熔断机制的大体流程如下：

[![img](https://www.servicemesher.com/istio-handbook/images/006tKfTcgy1g126mv3s9fj30bf0fcgmd.jpg)](https://www.servicemesher.com/istio-handbook/images/006tKfTcgy1g126mv3s9fj30bf0fcgmd.jpg)

这里需要解释两点：

- 开启熔断：在固定时间窗口内，接口调用超时比率达到一个阈值，会开启熔断。进入熔断状态后，后续对该服务接口的调用不再经过网络，直接执行本地的默认方法，达到服务降级的效果。
- 熔断恢复：熔断不可能是永久的，当经过了规定时间之后，服务将从熔断状态恢复过来，再次接受调用方的远程调用。

### 4.4.2 Istio 中的熔断

Istio 是通过 Envoy Proxy 来实现熔断机制的，Envoy 强制在网络层面配置熔断策略，这样就不必为每个应用程序单独配置或重新编程。下面就通过一个示例来演示如何为 Istio 网格中的服务配置熔断的连接数、请求数和异常检测。

该示例的架构如图所示：

[![img](https://www.servicemesher.com/istio-handbook/images/006tKfTcgy1g126o9utc8j30k208eaax.jpg)](https://www.servicemesher.com/istio-handbook/images/006tKfTcgy1g126o9utc8j30k208eaax.jpg)

该示例由客户端和服务端组成，其中客户端是一个 Java HTTP 应用程序，被打包在镜像 `docker.io/ceposta/http-envoy-client-standalone:latest` 中，它用来模拟对后端服务 `httpbin` 发起 http 调用，所有的调用首先都会被 Envoy Proxy 拦截。

> 假设你的集群中已经部署了 Istio，没有启用 Sidecar 的自动注入，并且没有启用双向 TLS 身份验证。



#### 4.4.2.1 部署示例

部署 [httpbin](https://github.com/istio/istio/tree/master/samples/httpbin) 应用，该应用将会作为本示例的后端服务：

```bash
# 进入 istio 根目录
$ kubectl apply -f <(istioctl kube-inject -f samples/httpbin/httpbin.yaml)
```

创建一个 `DestinationRule`，针对 `httpbin` 服务设置熔断策略：

```yaml
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: httpbin
spec:
  host: httpbin
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 1
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
    outlierDetection:
      consecutiveErrors: 2
      interval: 1s
      baseEjectionTime: 3m
      maxEjectionPercent: 100
EOF
```

查看该策略在 Envoy 中对应的 `Cluster` 配置：

```bash
$ kubectl get pod -l app=httpbin

NAME                      READY     STATUS    RESTARTS   AGE
httpbin-d6d68fb97-cswzc   2/2       Running   0          2m
$ istioctl pc cluster httpbin-d6d68fb97-cswzc --fqdn httpbin.default.svc.cluster.local --direction outbound -o json

[
    {
        "name": "outbound|8000||httpbin.default.svc.cluster.local",
        "type": "EDS",
        "edsClusterConfig": {
            "edsConfig": {
                "ads": {}
            },
            "serviceName": "outbound|8000||httpbin.default.svc.cluster.local"
        },
        "connectTimeout": "1.000s",
        "maxRequestsPerConnection": 1,
        "circuitBreakers": {
            "thresholds": [
                {
                    "maxConnections": 1,
                    "maxPendingRequests": 1
                }
            ]
        },
        "outlierDetection": {
            "interval": "1.000s",
            "baseEjectionTime": "180.000s",
            "maxEjectionPercent": 100,
            "enforcingConsecutive5xx": 0,
            "consecutiveGatewayFailure": 2,
            "enforcingConsecutiveGatewayFailure": 100
        }
    }
]
```

上面的配置告诉我们：

- **maxConnections** : 限制对后端服务发起的 `HTTP/1.1` 连接数，如果超过了这个限制，就会开启熔断。
- **maxPendingRequests** : 限制待处理请求列表的长度， 如果超过了这个限制，就会开启熔断。
- **maxRequestsPerConnection** : 在任何给定时间内限制对后端服务发起的 `HTTP/2` 请求数，如果超过了这个限制，就会开启熔断。

下面分别对这几个参数做详细解释。

- **maxConnections** : 表示在任何给定时间内， Envoy 与上游集群（这里指的是 httpbin 服务）建立的最大连接数。该配置仅适用于 `HTTP/1.1` 协议，因为 `HTTP/2` 协议可以在同一个 TCP 连接中发送多个请求，而 `HTTP/1.1` 协议在同一个连接中只能处理一个请求。如果超过了这个限制（即断路器溢出），集群的[upstream_cx_overflow](https://www.envoyproxy.io/docs/envoy/latest/configuration/cluster_manager/cluster_stats#config-cluster-manager-cluster-stats) 计数器就会递增。
- **maxPendingRequests** : 表示待处理请求队列的长度。因为 `HTTP/2` 是通过单个连接并发处理多个请求的，因此该熔断策略仅在创建初始 `HTTP/2` 连接时有用，之后的请求将会在同一个 TCP 连接上多路复用。对于 `HTTP/1.1` 协议，只要没有足够的上游连接可用于立即分派请求，就会将请求添加到待处理请求队列中，因此该断路器将在该进程的生命周期内保持有效。如果该断路器溢出，集群的 [upstream_rq_pending_overflow](https://www.envoyproxy.io/docs/envoy/latest/configuration/cluster_manager/cluster_stats#config-cluster-manager-cluster-stats) 计数器就会递增。
- **maxRequestsPerConnection** : 表示在任何给定时间内，上游集群中所有主机（这里指的是 httpbin 服务）可以处理的最大请求数。实际上，这适用于仅 `HTTP/2` 集群，因为 `HTTP/1.1` 集群由最大连接数断路器控制。如果该断路器溢出，集群的 [upstream_rq_pending_overflow](https://www.envoyproxy.io/docs/envoy/latest/configuration/cluster_manager/cluster_stats#config-cluster-manager-cluster-stats) 计数器就会递增。

Istio DestinationRule 与 Envoy 的熔断参数对照表如下所示：

|      Envoy paramether       |    Envoy upon object     |     Istio parameter      | Istio upon ojbect |
| :-------------------------: | :----------------------: | :----------------------: | :---------------: |
|       max_connections       | cluster.circuit_breakers |      maxConnections      |    TCPSettings    |
|    max_pending_requests     | cluster.circuit_breakers | http1MaxPendingRequests  |   HTTPSettings    |
|        max_requests         | cluster.circuit_breakers |     http2MaxRequests     |   HTTPSettings    |
|         max_retries         | cluster.circuit_breakers |        maxRetries        |   HTTPSettings    |
|     connect_timeout_ms      |         cluster          |      connectTimeout      |    TCPSettings    |
| max_requests_per_connection |         cluster          | maxRequestsPerConnection |   HTTPSettings    |

#### 4.4.2.2 最大连接数

现在我们已经为 `httpbin` 服务设置了熔断策略，接下来创建一个 Java 客户端，用来向后端服务发送请求，观察是否会触发熔断策略。这个客户端可以控制连接数量、并发数、待处理请求队列，使用这一客户端，能够有效的触发前面在目标规则中设置的熔断策略。该客户端的 deployment yaml 内容如下：

```yaml
# httpbin-client-deploy.yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: httpbin-client-v1
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: httpbin-client
        version: v1
    spec:
      containers:
      - image: ceposta/http-envoy-client-standalone:latest
        imagePullPolicy: IfNotPresent
        name: httpbin-client
        command: ["/bin/sleep","infinity"]
```

这里我们会把给客户端也进行 Sidecar 的注入，以此保证 Istio 对网络交互的控制：

```bash
$ kubectl apply -f <(istioctl kube-inject -f httpbin-client-deploy.yaml)
```

下面来观察一下当客户端试图使用太多线程与上游集群建立并发连接时，Envoy 会如何应对。

在上面的熔断设置中指定了 `maxConnections: 1` 以及 `http1MaxPendingRequests: 1`。这意味着如果超过了一个连接同时发起请求，Istio 就会熔断，阻止后续的请求或连接。

先尝试通过单线程（`NUM_THREADS=1`）创建一个连接，并进行 5 次调用（默认值：`NUM_CALLS_PER_CLIENT=5`）：

```bash
$ CLIENT_POD=$(kubectl get pod | grep httpbin-client | awk '{ print $1 }')
$ kubectl exec -it $CLIENT_POD -c httpbin-client -- sh -c 'export URL_UNDER_TEST=http://httpbin:8000/get export NUM_THREADS=1 && java -jar http-client.jar'

using num threads: 1
Starting pool-1-thread-1 with numCalls=5 delayBetweenCalls=0 url=http://localhost:15001/get mixedRespTimes=false
pool-1-thread-1: successes=[5], failures=[0], duration=[545ms]
```

可以看到所有请求都通过了：

```bash
successes=[5]
```

我们可以查询 istio-proxy 的状态，获取更多相关信息：

```bash
$ kubectl exec -it $CLIENT_POD  -c istio-proxy  -- sh -c 'curl localhost:15000/stats' | grep httpbin

...
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_cx_http1_total: 5
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_cx_overflow: 0
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_total: 5
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_200: 5
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_2xx: 5
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_pending_overflow: 0
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_retry: 0
...
```

可以看出总共发送了 5 个 `HTTP/1.1` 连接，也就是 5 个请求，响应码均为 `200`。

下面尝试把线程数提高到 2：

```bash
$ kubectl exec -it $CLIENT_POD -c httpbin-client -- sh -c 'export URL_UNDER_TEST=http://httpbin:8000/get export NUM_THREADS=2 && java -jar http-client.jar'

using num threads: 2
Starting pool-1-thread-1 with numCalls=5 parallelSends=false delayBetweenCalls=0 url=http://httpbin:8000/get mixedRespTimes=false
Starting pool-1-thread-2 with numCalls=5 parallelSends=false delayBetweenCalls=0 url=http://httpbin:8000/get mixedRespTimes=false
pool-1-thread-1: successes=[3], failures=[2], duration=[96ms]
pool-1-thread-2: successes=[4], failures=[1], duration=[87ms]
```

再次查看 istio-proxy 的状态：

```bash
$ kubectl exec -it $CLIENT_POD  -c istio-proxy  -- sh -c 'curl localhost:15000/stats' | grep httpbin

...
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_cx_http1_total: 7
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_cx_overflow: 3
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_total: 10
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_200: 7
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_2xx: 7
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_503: 3
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_5xx: 3
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_pending_overflow: 3
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_retry: 0
...
```

总共发送了 10 个 `HTTP/1` 连接，只有 7 个 被允许通过，剩余请求被断路器拦截了。其中 `upstream_cx_overflow` 的值为 3，表明 `maxConnections` 断路器起作用了。`Istio-proxy` 允许一定的冗余，你可以将线程数提高到 3，熔断的效果会更明显。

#### 4.4.2.3 待处理请求队列

测试完 `maxConnections` 断路器之后，我们再来测试一下 `maxPendingRequests` 断路器。前面已经将 `maxPendingRequests` 的值设置为 1，现在按照预想，我们只需要模拟在单个 `HTTP/1.1` 连接中同时发送多个请求，就可以触发该断路器开启熔断。由于 `HTTP/1.1` 同一个连接只能处理一个请求，剩下的请求只能放到待处理请求队列中。通过限制待处理请求队列的长度，可以对恶意请求、DoS 和系统中的级联错误起到一定的缓解作用。

现在尝试通过单线程（`NUM_THREADS=1`）创建一个连接，并同时发送 20 个请求（`PARALLEL_SENDS=true`，`NUM_CALLS_PER_CLIENT=20`）：

```bash
$ kubectl exec -it $CLIENT_POD -c httpbin-client -- sh -c 'export URL_UNDER_TEST=http://httpbin:8000/get export NUM_THREADS=1 export PARALLEL_SENDS=true export NUM_CALLS_PER_CLIENT=20 && java -jar http-client.jar'

using num threads: 1
Starting pool-1-thread-1 with numCalls=20 parallelSends=true delayBetweenCalls=0 url=http://httpbin:8000/get mixedRespTimes=false
finished batch 0
finished batch 5
finished batch 10
finished batch 15
pool-1-thread-1: successes=[16], failures=[4], duration=[116ms]
```

查询 istio-proxy 的状态：

```bash
$ kubectl exec -it $CLIENT_POD  -c istio-proxy  -- sh -c 'curl localhost:15000/stats' | grep httpbin | grep pending

cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_pending_active: 0
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_pending_failure_eject: 0
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_pending_overflow: 4
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_pending_total: 16
```

`upstream_rq_pending_overflow` 的值是 `4`，说明有 `4` 次调用触发了 `maxPendingRequests` 断路器的熔断策略，被标记为熔断。

#### 4.4.2.4 如果服务完全崩溃怎么办？

现在我们知道 Envoy 的熔断策略对集群中压力过大的上游服务起到一定的保护作用，但还有一种极端的情况需要我们考虑，如果集群中的某些节点完全崩溃（或者即将完全崩溃）该怎么办？

为了专门应对这种情况，Envoy 中引入了**异常检测**的功能，通过周期性的异常检测来动态确定上游集群中的某些主机是否异常，如果发现异常，就将该主机从连接池中隔离出去。异常检测是**被动**健康检查的一种形式，Envoy 同时支持[主动健康检查](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/health_checking#arch-overview-health-checking)和被动健康检查，它们可以同时启用，联合决定上游主机的健康状况。

#### 4.4.2.5 异常检测的隔离算法

根据异常检测的类型，对主机的隔离可以连续执行（例如连续返回 `5xx` 状态码），也可以周期性执行（例如配置了周期性成功率检测）。隔离算法的工作流程如下：

1. 检测到了某个主机异常。
2. 如果到目前为止负载均衡池中还没有主机被隔离出去，Envoy 将会立即隔离该异常主机；如果已经有主机被隔离出去，就会检查当前隔离的主机数是否低于设定的阈值（通过 [outlier_detection.max_ejection_percent](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/cluster/outlier_detection.proto#envoy-api-field-cluster-outlierdetection-max-ejection-percent) 指定），如果当前被隔离的主机数量不超过该阈值，就将该主机隔离出去，否则不隔离。
3. 隔离不是永久的，会有一个时间限制。当主机被隔离后，该主机就会被标记为不健康，并且不会被加入到负载均衡池中，除非负载均衡处于[恐慌](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/load_balancing/panic_threshold#arch-overview-load-balancing-panic-threshold)模式。隔离时间等于 [outlier_detection.base_ejection_time_ms](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/cluster/outlier_detection.proto#envoy-api-field-cluster-outlierdetection-base-ejection-time) 的值乘以主机被隔离的次数。所以如果某个主机连续出现故障，会导致它被隔离的时间越来越长。
4. 经过了规定的隔离时间之后，被隔离的主机将会自动恢复过来，重新接受调用方的远程调用。通常异常检测会与主动健康检查一起用于全面的健康检查解决方案。

> 恐慌模式指的是：在这种情况下，代理服务器会无视负载均衡池的健康标记，重新向所有主机发送数据。这是一个非常棒的机制。在分布式系统中，必须了解到的一点是，有时候“理论上”的东西可能不是正常情况，最好能降低一点要求来防止扩大故障影响。另外一方面，可以对这一比例进行控制（缺省情况下超过 50% 的驱逐就会进入恐慌状态），可以提高，也可以禁止这一阈值。

#### 4.4.2.6 异常检测类型

Envoy 支持以下几种异常检测类型：

- **连续 5xx 响应**：如果上游主机连续返回一定数量的 `5xx` 响应，该主机就会被驱逐。注意，这里的 `5xx` 响应不仅包括返回的 `5xx` 状态码（包括 `500`），也包括 HTTP 路由返回的一个事件（如连接超时和连接错误）。隔离主机所需的 `5xx` 响应数量由 [outlier_detection.consecutive_5xx](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/cluster/outlier_detection.proto#envoy-api-field-cluster-outlierdetection-consecutive-5xx) 的值控制。
- **连续网关故障**：如果上游主机连续返回一定数量的 `"gateway errors"`（`502`，`503` 或 `504` 状态码，但不包括 `500`），该主机就会被驱逐。这里同样也包括 HTTP 路由返回的一个事件（如连接超时和连接错误）。隔离主机所需的连续网关故障数量由 [outlier_detection.consecutive_gateway_failure](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/cluster/outlier_detection.proto#envoy-api-field-cluster-outlierdetection-consecutive-gateway-failure) 的值控制。
- **调用成功率**：基于调用成功率的异常检测类型会聚合集群中每个主机的调用成功率，然后根据统计的数据以给定的周期来隔离主机。如果该主机的请求数量小于 [outlier_detection.success_rate_request_volume](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/cluster/outlier_detection.proto#envoy-api-field-cluster-outlierdetection-success-rate-request-volume) 指定的值，则不会为该主机计算调用成功率，因此聚合的统计数据中不会包括该主机的调用成功率。如果在给定的周期内具有最小所需请求量的主机数小于 [outlier_detection.success_rate_minimum_hosts](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/cluster/outlier_detection.proto#envoy-api-field-cluster-outlierdetection-success-rate-minimum-hosts) 指定的值，则不会对该集群执行调用成功率检测。

Istio DestinationRule 与 Envoy 的异常检测参数对照表如下所示：

|      Envoy paramether       |     Envoy upon object     |  Istio parameter   | Istio upon ojbect |
| :-------------------------: | :-----------------------: | :----------------: | :---------------: |
| consecutive_gateway_failure | cluster.outlier_detection | consecutiveErrors  | outlierDetection  |
|          interval           | cluster.outlier_detection |      interval      | outlierDetection  |
|      baseEjectionTime       | cluster.outlier_detection |  baseEjectionTime  | outlierDetection  |
|     maxEjectionPercent      | cluster.outlier_detection | maxEjectionPercent | outlierDetection  |

Envoy 中还有一些其他参数在 Istio 中暂时是不支持的，具体参考 Envoy 官方文档 [Outlier detection](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/cluster/outlier_detection.proto#cluster-outlierdetection)。

现在我们回头再来看一下本文最初创建的 DestinationRule 中关于异常检测的配置：

```yaml
    outlierDetection:
      consecutiveErrors: 2
      interval: 1s
      baseEjectionTime: 3m
      maxEjectionPercent: 100
```

该配置表示每秒钟扫描一次上游主机，连续失败 `1` 次返回 5xx 错误码的所有主机会被移出连接池 3 分钟。

#### 4.4.2.7 异常检测示例

下面我们通过调用一个 URL 来指定 httpbin 服务返回 `502` 状态码，以此来触发连续网关故障异常检测。总共发起 3 次调用，因为 DestinationRule 中的配置要求 Envoy 的异常检测机制必须检测到两个连续的网关故障才会将 httpbin 服务移除负载均衡池。

```bash
$ kubectl exec -it $CLIENT_POD -c httpbin-client -- sh -c 'export URL_UNDER_TEST=http://httpbin:8000/status/502 export NUM_CALLS_PER_CLIENT=3 && java -jar http-client.jar'

using num threads: 1
Starting pool-1-thread-1 with numCalls=3 parallelSends=false delayBetweenCalls=0 url=http://httpbin:8000/status/502 mixedRespTimes=false
pool-1-thread-1: successes=[0], failures=[3], duration=[99ms]
```

查看 istio-proxy 的状态：

```bash
$ kubectl exec -it $CLIENT_POD  -c istio-proxy  -- sh -c 'curl localhost:15000/stats' | grep httpbin | grep outlier

cluster.outbound|8000||httpbin.default.svc.cluster.local.outlier_detection.ejections_active: 1
cluster.outbound|8000||httpbin.default.svc.cluster.local.outlier_detection.ejections_consecutive_5xx: 0
cluster.outbound|8000||httpbin.default.svc.cluster.local.outlier_detection.ejections_detected_consecutive_5xx: 0
cluster.outbound|8000||httpbin.default.svc.cluster.local.outlier_detection.ejections_detected_consecutive_gateway_failure: 1
cluster.outbound|8000||httpbin.default.svc.cluster.local.outlier_detection.ejections_detected_success_rate: 0
cluster.outbound|8000||httpbin.default.svc.cluster.local.outlier_detection.ejections_enforced_consecutive_5xx: 0
cluster.outbound|8000||httpbin.default.svc.cluster.local.outlier_detection.ejections_enforced_consecutive_gateway_failure: 1
cluster.outbound|8000||httpbin.default.svc.cluster.local.outlier_detection.ejections_enforced_success_rate: 0
cluster.outbound|8000||httpbin.default.svc.cluster.local.outlier_detection.ejections_enforced_total: 1
cluster.outbound|8000||httpbin.default.svc.cluster.local.outlier_detection.ejections_overflow: 0
cluster.outbound|8000||httpbin.default.svc.cluster.local.outlier_detection.ejections_success_rate: 0
cluster.outbound|8000||httpbin.default.svc.cluster.local.outlier_detection.ejections_total: 0
```

确实检测到了连续网关故障，`consecutive_gateway_failure` 的值为 1。但是我通过查看 EDS，发现 Envoy 并没有将 httpbin 服务移出负载均衡池：

```bash
$ export PILOT_SVC_IP=$(kubectl -n istio-system get svc istio-pilot -o go-template='{{.spec.clusterIP}}')
$ curl -s http://$PILOT_SVC_IP:8080/debug/edsz|grep "outbound|8000||httpbin.default.svc.cluster.local" -B 1 -A 15
{
  "clusterName": "outbound|8000||httpbin.default.svc.cluster.local",
  "endpoints": [
    {
      "locality": {

      },
      "lbEndpoints": [
        {
          "endpoint": {
            "address": {
              "socketAddress": {
                "address": "172.30.104.27",
                "portValue": 8000
              }
            }
          },
```

回去重新看了一下 Envoy 的配置，`enforcingConsecutiveGatewayFailure` 的值为 100，理论上 outlierDetection 应该是生效的。但这里我疏忽了 Envoy 的恐慌模式，Envoy 默认的恐慌阈值是 `50%`，而 httpbin 应用只有一个 upstream，所以没有被移除。当然这只是个人猜测，暂时也没时间验证，大家可以自己验证一下。





# 十二 为服务网格选择入口网关

## 12.1 为服务网格选择入口网关

在启用了Istio服务网格的Kubernetes集群中，缺省情况下只能在集群内部访问网格中的服务，要如何才能从外部网络访问这些服务呢？ Kubernetes和Istio提供了NodePort，LoadBalancer，Kubernetes Ingress，Istio Gateway等多种外部流量入口的方式，面对这么多种方式，我们在产品部署中应该如何选择？

本文将对Kubernetes和Istio对外提供服务的各种方式进行详细介绍和对比分析，并根据分析结果提出一个可用于产品部署的解决方案。

> 说明：阅读本文要求读者了解Kubernetes和Istio的基本概念，包括Pod、Service、NodePort、LoadBalancer、Ingress、Gateway、VirtualService等。如对这些概念不熟悉，可以在阅读过程中参考文后的相关链接。

​	

## 12.2 内部服务间的通信

首先，我们来回顾一下Kubernetes集群内部各个服务之间相互访问的方法。

### 12.2.1 Cluster IP

Kubernetes以Pod作为应用部署的最小单位。kubernetes会根据Pod的声明对其进行调度，包括创建、销毁、迁移、水平伸缩等，因此Pod 的IP地址不是固定的，不方便直接采用Pod IP对服务进行访问。

为解决该问题，Kubernetes提供了Service资源，Service对提供同一个服务的多个Pod进行聚合。一个Service提供一个虚拟的Cluster IP，后端对应一个或者多个提供服务的Pod。在集群中访问该Service时，采用Cluster IP即可，Kube-proxy负责将发送到Cluster IP的请求转发到后端的Pod上。

Kube-proxy是一个运行在每个节点上的go应用程序，支持三种工作模式：

#### 12.2.1.1 userspace 模式

该模式下kube-proxy会为每一个Service创建一个监听端口。发向Cluster IP的请求被Iptables规则重定向到Kube-proxy监听的端口上，Kube-proxy根据LB算法选择一个提供服务的Pod并和其建立链接，以将请求转发到Pod上。
该模式下，Kube-proxy充当了一个四层Load balancer的角色。由于kube-proxy运行在userspace中，在进行转发处理时会增加两次内核和用户空间之间的数据拷贝，效率较另外两种模式低一些；好处是当后端的Pod不可用时，kube-proxy可以重试其他Pod。

[![Kube-proxy userspace模式](https://www.servicemesher.com/istio-handbook/images/6ce41a46gy1g1l4lmw4z7j20m80cj0tq.jpg)](https://www.servicemesher.com/istio-handbook/images/6ce41a46gy1g1l4lmw4z7j20m80cj0tq.jpg)图片 - Kube-proxy userspace模式

图片来自：[Kubernetes官网文档](https://kubernetes.io/docs/concepts/services-networking/service/#virtual-ips-and-service-proxies/)

#### 12.2.1.2 iptables 模式

为了避免增加内核和用户空间的数据拷贝操作，提高转发效率，Kube-proxy提供了iptables模式。在该模式下，Kube-proxy为service后端的每个Pod创建对应的iptables规则，直接将发向Cluster IP的请求重定向到一个Pod IP。
该模式下Kube-proxy不承担四层代理的角色，只负责创建iptables规则。该模式的优点是较userspace模式效率更高，但不能提供灵活的LB策略，当后端Pod不可用时也无法进行重试。

[![Kube-proxy iptables模式](https://www.servicemesher.com/istio-handbook/images/6ce41a46gy1g1l4n2vx1tj20ol0h0dh3.jpg)](https://www.servicemesher.com/istio-handbook/images/6ce41a46gy1g1l4n2vx1tj20ol0h0dh3.jpg)图片 - Kube-proxy iptables模式

图片来自：[Kubernetes官网文档](https://kubernetes.io/docs/concepts/services-networking/service/#virtual-ips-and-service-proxies/)

#### 12.2.1.3 ipvs 模式

该模式和iptables类似，kube-proxy监控Pod的变化并创建相应的ipvs rules。ipvs也是在kernel模式下通过netfilter实现的，但采用了hash table来存储规则，因此在规则较多的情况下，Ipvs相对iptables转发效率更高。除此以外，ipvs支持更多的LB算法。如果要设置kube-proxy为ipvs模式，必须在操作系统中安装IPVS内核模块。

[![Kube-proxy ipvs模式](https://www.servicemesher.com/istio-handbook/images/6ce41a46gy1g1l4nvyl1vj20nj0g83zi.jpg)](https://www.servicemesher.com/istio-handbook/images/6ce41a46gy1g1l4nvyl1vj20nj0g83zi.jpg)图片 - Kube-proxy ipvs模式

图片来自：[Kubernetes官网文档](https://kubernetes.io/docs/concepts/services-networking/service/#virtual-ips-and-service-proxies/)

### 12.2.2 Istio Sidecar Proxy

Cluster IP解决了服务之间相互访问的问题，但从上面Kube-proxy的三种模式可以看到，Cluster IP的方式只提供了服务发现和基本的LB功能。如果要为服务间的通信应用灵活的路由规则以及提供Metrics collection，distributed tracing等服务管控功能,就必须得依靠Istio提供的服务网格能力了。

在Kubernetes中部署Istio后，Istio通过iptables和Sidecar Proxy接管服务之间的通信，服务间的相互通信不再通过Kube-proxy，而是通过Istio的Sidecar Proxy进行。请求流程是这样的：Client发起的请求被iptables重定向到Sidecar Proxy，Sidecar Proxy根据从控制面获取的服务发现信息和路由规则，选择一个后端的Server Pod创建链接，代理并转发Client的请求。

Istio Sidecar Proxy和Kube-proxy的userspace模式的工作机制类似，都是通过在用户空间的一个代理来实现客户端请求的转发和后端多个Pod之间的负载均衡。两者的不同点是：Kube-Proxy工作在四层，而Sidecar Proxy则是一个七层代理，可以针对HTTP，GRPS等应用层的语义进行处理和转发，因此功能更为强大，可以配合控制面实现更为灵活的路由规则和服务管控功能。

[![Istio Sidecar Proxy](https://www.servicemesher.com/istio-handbook/images/6ce41a46ly1g1kur74j27j20ho0bujsm.jpg)](https://www.servicemesher.com/istio-handbook/images/6ce41a46ly1g1kur74j27j20ho0bujsm.jpg)图片 - Istio Sidecar Proxy

## 12.3 如何从外部网络访问

Kubernetes的Pod IP和Cluster IP都只能在集群内部访问，而我们通常需要从外部网络上访问集群中的某些服务，Kubernetes提供了下述几种方式来为集群提供外部流量入口。

### 12.3.1 NodePort

NodePort在集群中的主机节点上为Service提供一个代理端口，以允许从主机网络上对Service进行访问。Kubernetes官网文档只介绍了NodePort的功能，并未对其实现原理进行解释。下面我们通过实验来分析NodePort的实现机制。

www.katacoda.com 这个网站提供了一个交互式的Kubernetes playground，注册即可免费实验kubernetes的相关功能，下面我们就使用Katacoda来分析Nodeport的实现原理。

在浏览器中输入这个网址：https://www.katacoda.com/courses/kubernetes/networking-introduction， 打开后会提供了一个实验用的Kubernetes集群，并可以通过网元模拟Terminal连接到集群的Master节点。

执行下面的命令创建一个nodeport类型的service。

```bash
kubectl apply -f nodeport.yaml
```

查看创建的service，可以看到kubernetes创建了一个名为webapp-nodeport-svc的service，并为该service在主机节点上创建了30080这个Nodeport。

```bash
master $ kubectl get svc
NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes             ClusterIP   10.96.0.1       <none>        443/TCP        36m
webapp1-nodeport-svc   NodePort    10.103.188.73   <none>        80:30080/TCP   3m
```

webapp-nodeport-svc后端对应两个Pod，其Pod的IP分别为10.32.0.3和10.32.0.5。

```bash
master $ kubectl get pod -o wide
NAME                                           READY     STATUS    RESTARTS   AGE       IPNODE      NOMINATED NODE
webapp1-nodeport-deployment-785989576b-cjc5b   1/1       Running   0          2m        10.32.0.3
webapp1-nodeport-deployment-785989576b-tpfqr   1/1       Running   0          2m        10.32.0.5
```

通过netstat命令可以看到Kube-proxy在主机网络上创建了30080监听端口，用于接收从主机网络进入的外部流量。

```bash
master $ netstat -lnp|grep 30080
tcp6       0      0 :::30080                :::*                    LISTEN      7427/kube-proxy
```

下面是Kube-proxy创建的相关iptables规则以及对应的说明。可以看到Kube-proxy为Nodeport创建了相应的IPtable规则，将发向30080这个主机端口上的流量重定向到了后端的两个Pod IP上。

```bash
iptables-save > iptables-dump
# Generated by iptables-save v1.6.0 on Thu Mar 28 07:33:57 2019
*nat
# Nodeport规则链
:KUBE-NODEPORTS - [0:0]
# Service规则链
:KUBE-SERVICES - [0:0]
# Nodeport和Service共用的规则链
:KUBE-SVC-J2DWGRZTH4C2LPA4 - [0:0]
:KUBE-SEP-4CGFRVESQ3AECDE7 - [0:0]
:KUBE-SEP-YLXG4RMKAICGY2B3 - [0:0]

# 将host上30080端口的外部tcp流量转到KUBE-SVC-J2DWGRZTH4C2LPA4链
-A KUBE-NODEPORTS -p tcp -m comment --comment "default/webapp1-nodeport-svc:" -m tcp --dport 30080 -j KUBE-SVC-J2DWGRZTH4C2LPA4

#将发送到Cluster IP 10.103.188.73的内部流量转到KUBE-SVC-J2DWGRZTH4C2LPA4链
KUBE-SERVICES -d 10.103.188.73/32 -p tcp -m comment --comment "default/webapp1-nodeport-svc: cluster IP" -m tcp --dport 80 -j KUBE-SVC-J2DWGRZTH4C2LPA4

#将发送到webapp1-nodeport-svc的流量转交到第一个Pod（10.32.0.3）相关的规则链上，比例为50%
-A KUBE-SVC-J2DWGRZTH4C2LPA4 -m comment --comment "default/webapp1-nodeport-svc:" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-YLXG4RMKAICGY2B3
#将发送到webapp1-nodeport-svc的流量转交到第二个Pod（10.32.0.5）相关的规则链上
-A KUBE-SVC-J2DWGRZTH4C2LPA4 -m comment --comment "default/webapp1-nodeport-svc:" -j KUBE-SEP-4CGFRVESQ3AECDE7

#将请求重定向到Pod 10.32.0.3
-A KUBE-SEP-YLXG4RMKAICGY2B3 -p tcp -m comment --comment "default/webapp1-nodeport-svc:" -m tcp -j DNAT --to-destination 10.32.0.3:80
#将请求重定向到Pod 10.32.0.5
-A KUBE-SEP-4CGFRVESQ3AECDE7 -p tcp -m comment --comment "default/webapp1-nodeport-svc:" -m tcp -j DNAT --to-destination 10.32.0.5:80
```

从上面的实验可以看到，通过将一个Service定义为NodePort类型，Kubernetes会通过集群中node上的Kube-proxy为该Service在主机网络上创建一个监听端口。Kube-proxy并不会直接接收该主机端口进入的流量，而是会创建相应的Iptables规则，并通过Iptables将从该端口收到的流量直接转发到后端的Pod中。

NodePort的流量转发机制和Cluster IP的iptables模式类似，唯一不同之处是在主机网络上开了一个“NodePort”来接受外部流量。从上面的规则也可以看出，在创建Nodeport时，Kube-proxy也会同时为Service创建Cluster IP相关的iptables规则。

> 备注：除采用iptables进行流量转发，NodePort应该也可以提供userspace模式以及ipvs模式，这里未就这两种模式进行实验验证。

从分析得知，在NodePort模式下，集群内外部的通讯如下图所示：

[![NodePort](https://www.servicemesher.com/istio-handbook/images/6ce41a46ly1g1kur7ink1j20dx0bcabj.jpg)](https://www.servicemesher.com/istio-handbook/images/6ce41a46ly1g1kur7ink1j20dx0bcabj.jpg)图片 - NodePort

### 12.3.2 LoadBalancer

NodePort提供了一种从外部网络访问Kubernetes集群内部Service的方法，但该方法存在下面一些限制，导致这种方式主要适用于程序开发，不适合用于产品部署。

- Kubernetes cluster host的IP必须是一个well-known IP，即客户端必须知道该IP。但Cluster中的host是被作为资源池看待的，可以增加删除，每个host的IP一般也是动态分配的，因此并不能认为host IP对客户端而言是well-known IP。
- 客户端访问某一个固定的host IP的方式存在单点故障。假如一台host宕机了，kubernetes cluster会把应用 reload到另一节点上，但客户端就无法通过该host的nodeport访问应用了。
- 通过一个主机节点作为网络入口，在网络流量较大时存在性能瓶颈。

为了解决这些问题，Kubernetes提供了LoadBalancer。通过将Service定义为LoadBalancer类型，Kubernetes在主机节点的NodePort前提供了一个四层的负载均衡器。该四层负载均衡器负责将外部网络流量分发到后面的多个节点的NodePort端口上。

下图展示了Kubernetes如何通过LoadBalancer方式对外提供流量入口，图中LoadBalancer后面接入了两个主机节点上的NodePort，后端部署了三个Pod提供服务。根据集群的规模，可以在LoadBalancer后面可以接入更多的主机节点，以进行负荷分担。

[![NodeBalancer](https://www.servicemesher.com/istio-handbook/images/6ce41a46ly1g1kur7aa0qj20qv0hl3zr.jpg)](https://www.servicemesher.com/istio-handbook/images/6ce41a46ly1g1kur7aa0qj20qv0hl3zr.jpg)图片 - NodeBalancer

> 备注：LoadBalancer类型需要云服务提供商的支持，Service中的定义只是在Kubernetes配置文件中提出了一个要求，即为该Service创建Load Balancer，至于如何创建则是由Google Cloud或Amazon Cloud等云服务商提供的，创建的Load Balancer的过程不在Kubernetes Cluster的管理范围中。
>
> 目前WS, Azure, CloudStack, GCE 和 OpenStack 等主流的公有云和私有云提供商都可以为Kubernetes提供Load Balancer。一般来说，公有云提供商还会为Load Balancer提供一个External IP，以提供Internet接入。如果你的产品没有使用云提供商，而是自建Kubernetes Cluster，则需要自己提供LoadBalancer。

### 12.3.3Ingress

LoadBalancer类型的Service提供的是四层负载均衡器，当只需要向外暴露一个服务的时候，采用这种方式是没有问题的。但当一个应用需要对外提供多个服务时，采用该方式则要求为每一个四层服务（IP+Port）都创建一个外部load balancer。

一般来说，同一个应用的多个服务/资源会放在同一个域名下，在这种情况下，创建多个Load balancer是完全没有必要的，反而带来了额外的开销和管理成本。另外直接将服务暴露给外部用户也会导致了前端和后端的耦合，影响了后端架构的灵活性，如果以后由于业务需求对服务进行调整会直接影响到客户端。

在这种情况下，我们可以通过使用Kubernetes Ingress来统一网络入口。Kubernetes Ingress声明了一个应用层（OSI七层）的负载均衡器，可以根据HTTP请求的内容将来自同一个TCP端口的请求分发到不同的Kubernetes Service，其功能包括：

#### 12.3.3.1 按HTTP请求的URL进行路由

同一个TCP端口进来的流量可以根据URL路由到Cluster中的不同服务，如下图所示：

[![按HTTP请求的ULR进行路由](https://www.servicemesher.com/istio-handbook/images/6ce41a46ly1g1kur85xbfj20fz0bp0t4.jpg)](https://www.servicemesher.com/istio-handbook/images/6ce41a46ly1g1kur85xbfj20fz0bp0t4.jpg)图片 - 按HTTP请求的ULR进行路由

#### 12.3.3.2 按HTTP请求的Host进行路由

同一个IP进来的流量可以根据HTTP请求的Host路由到Cluster中的不同服务，如下图所示：

[![按HTTP请求的Host进行路由](https://www.servicemesher.com/istio-handbook/images/6ce41a46ly1g1kur7zut9j20fw0caaaf.jpg)](https://www.servicemesher.com/istio-handbook/images/6ce41a46ly1g1kur7zut9j20fw0caaaf.jpg)图片 - 按HTTP请求的Host进行路由

Ingress 规则定义了对七层网关的要求，包括URL分发规则，基于不同域名的虚拟主机，SSL证书等。Kubernetes使用Ingress Controller 来监控Ingress规则，并通过一个七层网关来实现这些要求，一般可以使用Nginx，HAProxy，Envoy等。

虽然Ingress Controller通过七层网关为后端的多个Service提供了统一的入口，但由于其部署在集群中，因此并不能直接对外提供服务。实际上Ingress需要配合NodePort和LoadBalancer才能提供对外的流量入口，如下图所示：

[![采用Ingress, NodePortal和LoadBalancer提供外部流量入口的拓扑结构](https://www.servicemesher.com/istio-handbook/images/6ce41a46ly1g1kur7vshrj20lw0gpaao.jpg)](https://www.servicemesher.com/istio-handbook/images/6ce41a46ly1g1kur7vshrj20lw0gpaao.jpg)图片 - 采用Ingress, NodePortal和LoadBalancer提供外部流量入口的拓扑结构

上图描述了如何采用Ingress配合NodePort和Load Balancer为集群提供外部流量入口，从该拓扑图中可以看到该架构的伸缩性非常好，在NodePort，Ingress，Pod等不同的接入层面都可以对系统进行水平扩展，以应对不同的外部流量要求。

上图只展示了逻辑架构，下面的图展示了具体的实现原理：

[![采用Ingress, NodePortal和LoadBalancer提供外部流量入口的实现原理](https://www.servicemesher.com/istio-handbook/images/6ce41a46ly1g1kur7w5yoj20es0lpwfn.jpg)](https://www.servicemesher.com/istio-handbook/images/6ce41a46ly1g1kur7w5yoj20es0lpwfn.jpg)图片 - 采用Ingress, NodePortal和LoadBalancer提供外部流量入口的实现原理

流量从外部网络到达Pod的完整路径如下：

1. 外部请求先通过四层Load Balancer进入内部网络
2. Load Balancer将流量分发到后端多个主机节点上的NodePort (userspace转发)
3. 请求从NodePort进入到Ingress Controller (iptabes规则，Ingress Controller本身是一个NodePort类型的Service)
4. Ingress Controller根据Ingress rule进行七层分发，根据HTTP的URL和Host将请求分发给不同的Service (userspace转发)
5. Service将请求最终导入到后端提供服务的Pod中 (iptabes规则)

从前面的介绍可以看到，K8S Ingress提供了一个基础的七层网关功能的抽象定义，其作用是对外提供一个七层服务的统一入口，并根据URL/HOST将请求路由到集群内部不同的服务上。

## 12.4 如何为服务网格选择入口网关？

在Istio服务网格中，通过为每个Service部署一个sidecar代理，Istio接管了Service之间的请求流量。控制面可以对网格中的所有sidecar代理进行统一配置，实现了对网格内部流量的路由控制，从而可以实现灰度发布，流量镜像，故障注入等服务管控功能。但是，Istio并没有为入口网关提供一个较为完善的解决方案。

### 12.4.1 K8s Ingress

在0.8版本以前，Istio缺省采用K8s Ingress来作为Service Mesh的流量入口。K8s Ingress统一了应用的流量入口，但存在两个问题：

- K8s Ingress是独立在Istio体系之外的，需要单独采用Ingress rule进行配置，导致系统入口和内部存在两套互相独立的路由规则配置，运维和管理较为复杂。
- K8s Ingress rule的功能较弱，不能在入口处实现和网格内部类似的路由规则，也不具备网格sidecar的其它能力，导致难以从整体上为应用系统实现灰度发布、分布式跟踪等服务管控功能。

[![采用Kubernetes Ingress作为服务网格的流量入口](https://www.servicemesher.com/istio-handbook/images/6ce41a46ly1g1kur7amu9j20oy0bdwf0.jpg)](https://www.servicemesher.com/istio-handbook/images/6ce41a46ly1g1kur7amu9j20oy0bdwf0.jpg)图片 - 采用Kubernetes Ingress作为服务网格的流量入口

### 12.4.2 Istio Gateway

Istio社区意识到了Ingress和Mesh内部配置割裂的问题，因此从0.8版本开始，社区采用了 Gateway 资源代替K8s Ingress来表示流量入口。

Istio Gateway资源本身只能配置L4-L6的功能，例如暴露的端口，TLS设置等；但Gateway可以和绑定一个VirtualService，在VirtualService 中可以配置七层路由规则，这些七层路由规则包括根据按照服务版本对请求进行导流，故障注入，HTTP重定向，HTTP重写等所有Mesh内部支持的路由规则。

Gateway和VirtualService用于表示Istio Ingress的配置模型，Istio Ingress的缺省实现则采用了和Sidecar相同的Envoy proxy。

通过该方式，Istio控制面用一致的配置模型同时控制了入口网关和内部的sidecar代理。这些配置包括路由规则，策略检查、Telemetry收集以及其他服务管控功能。

[![采用 Istio Ingress Gateway作为服务网格的流量入口](https://www.servicemesher.com/istio-handbook/images/6ce41a46ly1g1kur6wqsjj20kh0cbaax.jpg)](https://www.servicemesher.com/istio-handbook/images/6ce41a46ly1g1kur6wqsjj20kh0cbaax.jpg)图片 - 采用 Istio Ingress Gateway作为服务网格的流量入口

### 12.4.3 应用对API Gateway的需求

采用Gateway和VirtualService实现的Istio Ingress Gateway提供了网络入口处的基础通信功能，包括可靠的通信和灵活的路由规则。但对于一个服务化应用来说，网络入口除了基础的通讯功能之外，还有一些其他的应用层功能需求，例如：

- 第三方系统对API的访问控制
- 用户对系统的访问控制
- 修改请求/返回数据
- 服务API的生命周期管理
- 服务访问的SLA、限流及计费
- ….

[![Kubernetes ingress, Istio gateway and API gateway的功能对比](https://www.servicemesher.com/istio-handbook/images/6ce41a46ly1g1kv0ys0ndj20m80azdiw.jpg)](https://www.servicemesher.com/istio-handbook/images/6ce41a46ly1g1kv0ys0ndj20m80azdiw.jpg)图片 - Kubernetes ingress, Istio gateway and API gateway的功能对比

API Gateway需求中很大一部分需要根据不同的应用系统进行定制，目前看来暂时不大可能被纳入K8s Ingress或者Istio Gateway的规范之中。为了满足这些需求，涌现出了各类不同的k8s Ingress Controller以及Istio Ingress Gateway实现，包括Ambassador ，Kong, Traefik,Solo等。

这些网关产品在实现在提供基础的K8s Ingress能力的同时，提供了强大的API Gateway功能，但由于缺少统一的标准，这些扩展实现之间相互之间并不兼容。而且遗憾的是，目前这些Ingress controller都还没有正式提供和Istio 控制面集成的能力。

> 备注：
>
> - Ambassador将对Istio路由规则的支持纳入了Roadmap https://www.getambassador.io/user-guide/with-istio/
> - Istio声称支持Istio-Based Route Rule Discovery (尚处于实验阶段) https://gloo.solo.io/introduction/architecture/

### 12.4.4 采用API Gateway + Sidecar Proxy作为服务网格的流量入口

在目前难以找到一个同时具备API Gateway和Istio Ingress能力的网关的情况下，一个可行的方案是使用API Gateway和Sidecar Proxy一起为服务网格提供外部流量入口。

由于API Gateway已经具备七层网关的功能，Mesh Ingress中的Sidecar只需要提供VirtualService资源的路由能力，并不需要提供Gateway资源的网关能力，因此采用Sidecar Proxy即可。网络入口处的Sidecar Proxy和网格内部应用Pod中Sidecar Proxy的唯一一点区别是：该Sidecar只接管API Gateway向Mesh内部的流量，并不接管外部流向API Gateway的流量；而应用Pod中的Sidecar需要接管进入应用的所有流量。

[![采用API Gateway + Sidecar Proxy为服务网格提供流量入口](https://www.servicemesher.com/istio-handbook/images/6ce41a46ly1g1kur7v8ktj20nt0c0ab2.jpg)](https://www.servicemesher.com/istio-handbook/images/6ce41a46ly1g1kur7v8ktj20nt0c0ab2.jpg)图片 - 采用API Gateway + Sidecar Proxy为服务网格提供流量入口

> 备注：在实际部署时，API Gateway前端需要采用NodePort和LoadBalancer提供外部流量入口。为了突出主题，对上图进行了简化，没有画出NodePort和LoadBalancer。

采用API Gateway和Sidecar Proxy一起作为服务网格的流量入口，既能够通过对网关进行定制开发满足产品对API网关的各种需求，又可以在网络入口处利用服务网格提供的灵活的路由能力和分布式跟踪，策略等管控功能，是服务网格产品入口网关的一个理想方案。

性能方面的考虑：从上图可以看到，采用该方案后，外部请求的处理流程在入口处增加了Sidecar Proxy这一跳，因此该方式会带来少量的性能损失，但该损失是完全可以接受的。

对于请求时延而言，在服务网格中，一个外部请求本来就要经过较多的代理和应用进程的处理，在Ingress处增加一个代理对整体的时延影响基本忽略不计，而且对于绝大多数应用来说，网络转发所占的时间比例本来就很小，99%的耗时都在业务逻辑。如果系统对于增加的该时延非常敏感，则建议重新考虑该系统是否需要采用微服务架构和服务网格。

对于吞吐量而言，如果入口处的网络吞吐量存在瓶颈，则可以通过对API Gateway + Sidecar Proxy组成的Ingress整体进行水平扩展，来对入口流量进行负荷分担，以提高网格入口的网络吞吐量。



