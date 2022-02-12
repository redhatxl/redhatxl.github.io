# Istio之XDS协议解析

# 一 xDS是什么

xds是一类发现服务的总称，包含LDS，RDS，CDS，EDS以及SDS，Envoy通过xDS API可以动态获取Listen（监听器），Route（路由），Cluster（集群），Endpoint（集群成员）以及Secret（密钥）配置。

# 二 LDS

Listener发现服务，Listener监听器控制Envoy启动端口监听（目前只支持TCP协议），并配置L3/L4层过滤器，当网络链接到达后，配置好的网络过滤器堆栈开始处理后续事件，	通过通过的监听器体系结构用于执行大多数不同的代理任务（限流，客户端认证，http链接管理，tcp代理等）	

# 三 RDS

route discovery service，用于http链接管理过滤器动态获取路由配置，路由配置保护http头部修改（增加，删除http头部犍值），virtual host（虚拟主机），以及virtual host定义的路由条目。

# 四 CDS

Cluster发现服务,用于动态获取Cluster信息。Envoy cluster管理器管理着所有的上游cluster。鉴于上游cluster或者主机可用于任何代理转发任务, 所以上游cluster-般从Listener或Route中抽象出来。

# 五 EDS

Endpoint发现服务。在Envoy术语中, Cluster成员就叫Endpoint ,对于每个Cluster , Envoy通过EDS API动态获取Endpoint。EDS作为首选的服务发现的原因有两点:

* 与通过DNS解析的负载均衡器进行路由相比, Envoy能明确的知道每个上游主机的信息,因而可以做出更加智能的负载均衡决策。

* Endpoint配置包含负载均衡权重、可用域等附加主机属性,这些属性可用于服务网格负载均衡,统计收集等过程中。

  五 EDS













