# istio之流量管理

# 一 概述

Istio的流量路由规则使您可以轻松控制服务之间的流量和API调用。Istio简化了诸如断路器，超时和重试之类的服务级别属性的配置，并使其易于设置重要任务（如A / B测试，canary部署和基于百分比的流量拆分的分段部署）。它还提供了开箱即用的故障恢复功能，有助于使您的应用程序更强大，以防止相关服务或网络的故障。

Istio的流量管理模型依赖于 随服务一起部署的Envoy代理。网状服务发送和接收的所有流量（数据平面流量）都通过Envoy进行代理，从而可以轻松地引导和控制网状网络周围的流量，而无需对服务进行任何更改。

如果您对本指南中描述的功能的工作方式的细节感兴趣，可以在本文档结尾的“ [架构”](https://istio.io/docs/concepts/traffic-management/#architecture)部分中找到有关Istio流量管理架构的更多信息。本指南的其余部分介绍了Istio的流量管理功能。

为了在网状网络内引导流量，Istio需要知道所有端点在哪里以及它们属于哪些服务。为了填充自己的 服务注册表，Istio连接到服务发现系统。例如，如果您在Kubernetes群集上安装了Istio，则Istio会自动检测该群集中的服务和端点。

使用此服务注册表，Envoy代理可以将流量定向到相关服务。大多数基于微服务的应用程序每个服务工作负载都有多个实例来处理服务流量，有时也称为负载平衡池。默认情况下，Envoy代理使用循环模型在每个服务的负载平衡池中分配流量，该请求将请求依次发送到每个池成员，并在每个服务实例收到请求后返回池的顶部。

虽然Istio的基本服务发现和负载平衡为您提供了一个有效的服务网格，但它远非Istio可以做的。在许多情况下，您可能希望对网状流量发生的情况进行更细粒度的控制。您可能希望将特定百分比的流量定向到服务的新版本，作为A / B测试的一部分，或者将不同的负载平衡策略应用于特定服务实例子集的流量。您可能还想将特殊规则应用于进出网格的流量，或将网格的外部依赖项添加到服务注册表。您可以使用Istio的流量管理API向Istio添加自己的流量配置，以完成所有这些工作。

与其他Istio配置一样，使用Kubernetes自定义资源定义（CRD）指定API ，您可以使用YAML对其进行配置，如示例所示。

# 二 Virtual services

[虚拟服务](https://istio.io/docs/reference/config/networking/v1alpha3/virtual-service/#VirtualService)以及[目标规则](https://istio.io/docs/concepts/traffic-management/#destination-rules)是Istio流量路由功能的关键构建块。虚拟服务使您可以在Istio和您的平台提供的基本连接和发现的基础上，配置如何将请求路由到Istio服务网格中的服务。每个虚拟服务由一组路由规则组成，这些路由规则按顺序进行评估，从而使Istio将对虚拟服务的每个给定请求匹配到网格内特定的实际目的地。您的网格可能需要多个虚拟服务，也可能不需要多个虚拟服务，具体取决于您的用例。

## 2.1 为什么使用Virtual services

虚拟服务在使Istio的流量管理变得灵活和强大方面起着关键作用。他们通过将客户端从真正实现它们的目标工作负载中发送请求的位置之间去耦合，来实现这一点。虚拟服务还提供了一种丰富的方法来指定用于将流量发送到那些工作负载的不同流量路由规则。

为什么这么有用？没有虚拟服务，Envoy使用循环负载平衡在所有服务实例之间分配流量，如简介中所述。您可以通过了解工作负载来改善此行为。例如，某些代表不同的版本。这在A / B测试中很有用，您可能需要基于不同服务版本之间的百分比来配置流量路由，或将流量从内部用户引导到一组特定的实例。

使用虚拟服务，可以为一个或多个主机名指定流行为。您可以在虚拟服务中使用路由规则，该规则告诉Envoy如何将虚拟服务的流量发送到适当的目的地。路由目的地可以是相同服务的版本，也可以是完全不同的服务的版本。

典型的用例是将流量发送到指定为服务子集的服务的不同版本。客户端将请求发送到虚拟服务主机，就好像它是单个实体一样，然后Envoy根据虚拟服务规则将流量路由到不同版本：例如，“ 20％的呼叫转到新版本”或“呼叫这些用户转到版本2”。例如，这使您可以创建一个“金丝雀”部署，在其中逐步增加发送到新服务版本的流量百分比。流量路由与实例部署完全分开，这意味着可以完全根据流量负载来扩展和缩小实现新服务版本的实例的数量，而无需完全引用流量路由。相比之下，像Kubernetes这样的容器编排平台仅支持基于实例扩展的流量分配，实例扩展很快变得复杂。您可以在下面阅读有关虚拟服务如何帮助金丝雀部署的更多信息。[使用Istio的Canary部署](https://istio.io/blog/2017/0.1-canary/)。

虚拟服务还使您：

- 通过单个虚拟服务处理多个应用程序服务。例如，如果网格使用Kubernetes，则可以配置虚拟服务以处理特定名称空间中的所有服务。将单个虚拟服务映射到多个“真实”服务在帮助将单片应用程序转换为由不同的微服务构建的复合服务时特别有用，而无需该服务的使用者适应过渡。您的路由规则可以指定“对`monolith.com`go的这些URI的调用 `microservice A`”，依此类推。您可以在[下面的示例之一中](https://istio.io/docs/concepts/traffic-management/#more-about-routing-rules)查看其工作原理。
- 结合[网关](https://istio.io/docs/concepts/traffic-management/#gateways)配置流量规则， 以控制入口和出口流量。

在某些情况下，您还需要配置目标规则以使用这些功能，因为这些是您指定服务子集的位置。在单独的对象中指定服务子集和其他特定于目标的策略，使您可以在虚拟服务之间清晰地重用它们。您可以在下一部分中找到有关目标规则的更多信息。

## 2.2 虚拟服务示例

以下虚拟服务根据请求是否来自特定用户将请求路由到服务的不同版本。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v3
```

### 2.2.1 主机字段

该`hosts`字段列出了虚拟服务的主机-换句话说，这些路由规则适用于用户可寻址的目的地。这是客户端向服务发送请求时使用的地址。

```yaml
hosts:
- reviews
```

虚拟服务主机名可以是IP地址，DNS名称，也可以是短名称（例如Kubernetes服务短名称），具体取决于平台，后者可以隐式或显式地解析为完全限定的域名（FQDN）。您还可以使用通配符（“ *”）前缀，从而为所有匹配的服务创建一套路由规则。虚拟服务主机实际上不一定是Istio服务注册表的一部分，它们只是虚拟目的地。这样，您就可以为网格内没有可路由条目的虚拟主机的流量建模。

### 2.2.2 路由规则

本`http`节包含虚拟服务的路由规则，描述了路由发送到主机字段中指定的目标的HTTP / 1.1，HTTP2和gRPC流量的匹配条件和操作（您还可以使用`tcp`和 `tls`部分来配置[TCP的](https://istio.io/docs/reference/config/networking/v1alpha3/virtual-service/#TCPRoute)路由规则 和未终止的 [TLS](https://istio.io/docs/reference/config/networking/v1alpha3/virtual-service/#TLSRoute) 流量）。路由规则由您希望流量流向的目的地以及零个或多个匹配条件组成，具体取决于您的用例。

* 匹配条件

示例中的第一个路由规则具有条件，因此从该`match`字段开始 。在这种情况下，你想要这个路由适用于从用户的“杰森”的所有要求，让你用的`headers`，`end-user`和`exact`字段选择适当的请求。

```yaml
- match:
   - headers:
       end-user:
         exact: jason
```

* 目的地

路线部分的`destination`字段指定符合此条件的流量的实际目的地。与虚拟服务的主机不同，目的地的主机必须是Istio的服务注册表中存在的真实目的地，否则Envoy将不知道将流量发送到哪里。这可以是带有代理的网格服务，也可以是使用服务条目添加的非网格服务。在这种情况下，我们在Kubernetes上运行，主机名是Kubernetes服务名：

```yaml
route:
- destination:
    host: reviews
    subset: v2
```

请注意，在本页面的该示例和其他示例中，为简单起见，我们使用Kubernetes短名称作为目标主机。评估此规则后，Istio将基于虚拟服务的名称空间添加一个域后缀，该名称后缀包含路由规则以获取主机的标准名称。在我们的示例中使用短名称还意味着您可以在任意喜欢的名称空间中复制并尝试使用它们。

⚠️：仅当目标主机和虚拟服务实际上在同一Kubernetes命名空间中时，才使用这样的短名称。因为使用Kubernetes短名称可能会导致配置错误，所以我们建议您在生产环境中指定标准主机名。

Destination部分还指定要让符合此规则条件的请求转到此Kubernetes服务的哪个子集，在本例中为名为v2的子集。您将在下面有关[目标规则](https://istio.io/docs/concepts/traffic-management/#destination-rules)的部分中看到如何定义服务子集 。

### 2.2.3 路由规则优先级

路由规则**从上到下按顺序评估**，其中虚拟服务定义中的第一个规则具有最高优先级。在这种情况下，您希望不符合第一个路由规则的所有内容都转到第二个规则中指定的默认目标。因此，第二条规则没有匹配条件，仅将流量定向到v3子集。

```yaml
- route:
  - destination:
      host: reviews
      subset: v3
```

我们建议在每个虚拟服务中提供默认的“无条件”或基于权重的规则（如下所述）作为每个虚拟服务中的最后一个规则，以确保到虚拟服务的流量始终具有至少一条匹配的路由。

## 2.3 路由规则更多信息

如上所述，路由规则是将流量的特定子集路由到特定目的地的强大工具。您可以在流量端口，标头字段，URI等上设置匹配条件。例如，此虚拟服务使用户可以将流量发送到两个单独的服务（评级和评论），就像他们是较大虚拟服务的一部分一样。`http://bookinfo.com/.`虚拟服务规则根据请求URI匹配流量，并将请求定向到适当的服务。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo
spec:
  hosts:
    - bookinfo.com
  http:
  - match:
    - uri:
        prefix: /reviews
    route:
    - destination:
        host: reviews
  - match:
    - uri:
        prefix: /ratings
    route:
    - destination:
        host: ratings
...

  http:
  - match:
      sourceLabels:
        app: reviews
    route:
...

```

对于某些匹配条件，您还可以选择使用精确值，前缀或正则表达式来选择它们。

您可以将多个匹配条件添加`match`到与您的条件相同的块，或者将多个匹配条件添加到与您的条件相同的规则。对于任何给定的虚拟服务，您还可以具有多个路由规则。这样，您就可以在单个虚拟服务中根据自己的喜好使路由条件变得复杂或简单。匹配条件字段及其可能值的完整列表可以在[`HTTPMatchRequest`参考资料中](https://istio.io/docs/reference/config/networking/v1alpha3/virtual-service/#HTTPMatchRequest)找到 。

除了使用匹配条件外，您还可以按百分比“权重”分配流量。这对于A / B测试和Canary推出很有用：

⚠️：匹配url规则可以用于区分不同的服务，按百分比“权重”可以区分形同服务的多个版本

```yaml
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 75
    - destination:
        host: reviews
        subset: v2
      weight: 25

```

您还可以使用路由规则对流量执行一些操作，例如：

- 附加或删除标题。
- 重写URL。
- 为该目的地的呼叫设置[重试策略](https://istio.io/docs/concepts/traffic-management/#retries)。

要了解有关可用操作的更多信息，请参见 [`HTTPRoute`参考资料](https://istio.io/docs/reference/config/networking/v1alpha3/virtual-service/#HTTPRoute)。

# 三 Destination rules

与[虚拟服务一起](https://istio.io/docs/concepts/traffic-management/#virtual-services)， [目的地规则](https://istio.io/docs/reference/config/networking/v1alpha3/destination-rule/#DestinationRule) 是Istio流量路由功能的关键部分。你能想到的虚拟服务，如何您将自己的流量**到**你指定的目的地，然后用目的地规则来配置发生了什么流量**为**那个目的地。在评估了虚拟服务路由规则之后，将应用目标规则，因此它们将应用于流量的“真实”目标。

特别是，您使用目标规则来指定命名的服务子集，例如按版本对所有给定服务的实例进行分组。然后，您可以在虚拟服务的路由规则中使用这些服务子集，以控制流向服务的不同实例的流量。

目标规则还允许您在调用整个目标服务或特定服务子集时自定义Envoy的流量策略，例如首选的负载平衡模型，TLS安全模式或断路器设置。您可以在“ [目标规则”参考中](https://istio.io/docs/reference/config/networking/v1alpha3/destination-rule/)看到目标规则选项的完整列表 。

## 3.1 负载均衡器选项

默认情况下，Istio使用循环负载平衡策略，其中实例池中的每个服务实例依次获得一个请求。Istio还支持以下模型，您可以在目标规则中为对特定服务或服务子集的请求指定这些模型。

- 随机：将请求随机转发到池中的实例。
- 加权：根据特定百分比将请求转发到池中的实例。
- 最少请求：将请求转发到请求数量最少的实例。

有关每个选项的更多信息，请参阅 [Envoy负载平衡文档](https://www.envoyproxy.io/docs/envoy/v1.5.0/intro/arch_overview/load_balancing)。

## 3.2 目标规则示例

以下示例目标规则`my-svc`使用不同的负载平衡策略为目标服务配置了三个不同的子集：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: my-destination-rule
spec:
  host: my-svc
  trafficPolicy:
    loadBalancer:
      simple: RANDOM
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
    trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN
  - name: v3
    labels:
      version: v3
```



每个子集都是基于一个或多个定义的`labels`，在Kubernetes中，这些子集是附加到对象（例如Pod）的键/值对。这些标签将应用于Kubernetes服务的部署中`metadata`以标识不同的版本。

除了定义子集外，此目标规则还具有针对此目标中所有子集的默认流量策略，以及针对该子集覆盖该子集的特定于子集的策略。在该`subsets` 字段上方定义的默认策略为`v1`和`v3`子集设置了一个简单的随机负载均衡器。在该 `v2`策略中，在相应子集的字段中指定了循环负载均衡器。

# 四 Gateways

您可以使用[网关](https://istio.io/docs/reference/config/networking/v1alpha3/gateway/#Gateway)来管理网格的入站和出站流量，让您指定要输入或离开网格的流量。网关配置适用于在网格边缘运行的独立Envoy代理，而不是与服务工作负载一起运行的sidecar Envoy代理。

与其他控制进入系统的流量的机制（例如Kubernetes Ingress API）不同，Istio网关使您可以充分利用Istio流量路由的功能和灵活性。您可以执行此操作，因为Istio的网关资源仅允许您配置4-6层负载平衡属性，例如要公开的端口，TLS设置等。然后，您没有将应用程序层流量路由（L7）添加到相同的API资源，而是将常规Istio [虚拟服务](https://istio.io/docs/concepts/traffic-management/#virtual-services)绑定到网关。这样，您就可以像管理Istio网格中的任何其他数据平面流量一样，基本上管理网关流量。

网关主要用于管理入口流量，但是您也可以配置出口网关。出口网关使您可以为离开网状网络的流量配置专用的出口节点，从而限制可以或应该访问外部网络的服务，或者启用 [对出口流量的安全控制](https://istio.io/blog/2019/egress-traffic-control-in-istio-part-1/) 以增加网状网络的安全性。您还可以使用网关来配置纯内部代理。

Istio提供了一些可以预配置的网关代理部署（`istio-ingressgateway`和`istio-egressgateway`），您可以使用-如果您使用我们的[演示安装](https://istio.io/docs/setup/install/kubernetes/)，则两者都将部署，而只有入口网关将使用我们的[默认配置文件或sds配置文件进行](https://istio.io/docs/setup/additional-setup/config-profiles/)部署 [。](https://istio.io/docs/setup/additional-setup/config-profiles/)您可以将自己的网关配置应用于这些部署，也可以部署和配置自己的网关代理。

## 4.1 网关示例

以下示例显示了用于外部HTTPS入口流量的可能网关配置：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: ext-host-gwy
spec:
  selector:
    app: my-gateway-controller
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - ext-host.example.com
    tls:
      mode: SIMPLE
      serverCertificate: /tmp/tls.crt
      privateKey: /tmp/tls.key
```

此网关配置允许HTTPS流量`ext-host.example.com`进入端口443上的网格，但不为该流量指定任何路由。

要指定路由并使网关按预期工作，您还必须将网关绑定到虚拟服务。您可以使用虚拟服务的`gateways`字段来执行此操作 ，如以下示例所示：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: virtual-svc
spec:
  hosts:
  - ext-host.example.com
  gateways:
    - ext-host-gwy
```

然后，您可以使用外部流量的路由规则配置虚拟服务。

# 五 Service entries

您使用 [服务条目](https://istio.io/docs/reference/config/networking/v1alpha3/service-entry/#ServiceEntry)将[条目](https://istio.io/docs/reference/config/networking/v1alpha3/service-entry/#ServiceEntry)添加到Istio内部维护的服务注册表中。添加服务条目后，Envoy代理可以将流量发送到服务，就好像它是网格中的服务一样。通过配置服务条目，您可以管理在网格外部运行的服务的流量，包括以下任务：

- 重定向和转发外部目标的流量，例如从网络使用的API或旧基础结构中服务的流量。
- 为外部目标定义[重试](https://istio.io/docs/concepts/traffic-management/#retries)，[超时](https://istio.io/docs/concepts/traffic-management/#timeouts)和 [故障注入](https://istio.io/docs/concepts/traffic-management/#fault-injection)策略。
- 将在虚拟机（VM）中运行的服务添加到网格以 [扩展网格](https://istio.io/docs/examples/mesh-expansion/single-network/#running-services-on-a-mesh-expansion-machine)。
- 在逻辑[上将](https://istio.io/docs/setup/install/multicluster/gateways/#configure-the-example-services)来自其他群集的服务添加到网格，以 在Kubernetes上配置 [多群集Istio网格](https://istio.io/docs/setup/install/multicluster/gateways/#configure-the-example-services)。

您不需要为要使用网格服务的每个外部服务添加服务条目。默认情况下，Istio将Envoy代理配置为将请求传递给未知服务。但是，您不能使用Istio功能来控制流向未在网格中注册的目标的流量。

## 5.1 服务输入示例

以下示例mesh-external服务条目将`ext-resource` 外部依赖项添加到Istio的服务注册表中：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: svc-entry
spec:
  hosts:
  - ext-svc.example.com
  ports:
  - number: 443
    name: https
    protocol: HTTPS
  location: MESH_EXTERNAL
  resolution: DNS
```



您可以使用该`hosts`字段指定外部资源。您可以完全限定它，也可以使用带通配符前缀的域名。

您可以配置虚拟服务和目标规则，以更精细的方式控制到服务条目的流量，就像为网格中的其他任何服务配置流量一样。例如，以下目标规则将流量路由配置为使用双向TLS来保护与`ext-svc.example.com`我们使用服务条目配置的外部服务的连接 ：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: ext-res-dr
spec:
  host: ext-svc.example.com
  trafficPolicy:
    tls:
      mode: MUTUAL
      clientCertificate: /etc/certs/myclientcert.pem
      privateKey: /etc/certs/client_private_key.pem
      caCertificates: /etc/certs/rootcacerts.pem
```



有关 更多可能的配置选项，请参阅 [服务条目参考](https://istio.io/docs/reference/config/networking/v1alpha3/service-entry)。

# 六 Sidecars

默认情况下，Istio将每个Envoy代理配置为在其相关工作负载的所有端口上接受流量，并在转发流量时到达网格中的每个工作负载。您可以使用[sidecar](https://istio.io/docs/reference/config/networking/v1alpha3/sidecar/#Sidecar)配置执行以下操作：

- 微调Envoy代理接受的端口和协议集。
- 限制Envoy代理可以访问的服务集。

在大型应用程序中，您可能希望像这样限制Sidecar的可达性，在该应用程序中，将每个代理配置为可访问网格中的所有其他服务可能会由于内存使用率较高而潜在地影响网格性能。

您可以指定希望将sidecar配置应用于特定名称空间中的所有工作负载，或使用来选择特定工作负载 `workloadSelector`。例如，以下sidecar配置将`bookinfo`名称空间中的所有服务配置为仅访问在相同名称空间和Istio控制平面（当前需要使用Istio的策略和遥测功能）中运行的服务：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Sidecar
metadata:
  name: default
  namespace: bookinfo
spec:
  egress:
  - hosts:
    - "./*"
    - "istio-system/*"
```

有关 更多详细信息，请参见[Sidecar参考](https://istio.io/docs/reference/config/networking/v1alpha3/sidecar/)。



# 七 Network resilience and testing

除了帮助您引导网络中的流量外，Istio还提供了可以在运行时动态配置的可选故障恢复和故障注入功能。使用这些功能可以帮助您的应用程序可靠地运行，确保服务网格可以容忍出现故障的节点，并防止局部故障级联到其他节点。

## 7.1 Timeouts

超时是Envoy代理应等待来自给定服务的答复的时间，以确保服务不会无限期地等待答复，并确保呼叫在可预测的时间内成功或失败。HTTP请求的默认超时为15秒，这意味着如果服务在15秒内未响应，则调用将失败。

对于某些应用程序和服务，Istio的默认超时可能不合适。例如，超时时间过长可能导致等待失败服务的回复导致过多的延迟，而超时时间过短可能导致在等待涉及多个服务的操作返回时呼叫不必要地失败。为了查找和使用最佳超时设置，Istio使您可以使用[虚拟服务](https://istio.io/docs/concepts/traffic-management/#virtual-services)轻松地基于[虚拟服务](https://istio.io/docs/concepts/traffic-management/#virtual-services)按服务动态调整超时，而无需编辑服务代码。这是一个虚拟服务，它为评级服务的v1子集的调用指定了10秒的超时时间：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - route:
    - destination:
        host: ratings
        subset: v1
    timeout: 10s
```

## 7.2 Retries

重试设置指定如果初始调用失败，Envoy代理尝试连接到服务的最大次数。重试可以确保不会因暂时性问题（例如服务或网络暂时过载）而导致呼叫永久失败，从而提高服务可用性和应用程序性能。重试之间的间隔（25ms +）是可变的，并由Istio自动确定，以防止被调用的服务被请求所淹没。默认情况下，Envoy代理在第一次失败后不尝试重新连接到服务。

像超时一样，Istio的默认重试行为在延迟（对失败的服务进行过多的重试会降低速度）或可用性方面可能不适合您的应用程序需求。与超时一样，您也可以在[虚拟](https://istio.io/docs/concepts/traffic-management/#virtual-services)服务中基于每个服务调整重试设置，而无需触摸服务代码。您还可以通过添加每次重试超时，指定您要等待每次重试尝试成功连接到服务的时间，来进一步优化重试行为。下面的示例在一次初始呼叫失败后最多配置3次重试以连接到该服务子集，每个重试都有2秒的超时时间。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - route:
    - destination:
        host: ratings
        subset: v1
    retries:
      attempts: 3
      perTryTimeout: 2s
```

## 7.3 Circuit breakers

断路器是Istio提供的另一个有用的机制，可用于创建基于弹性微服务的应用程序。在断路器中，可以设置对服务内单个主机的呼叫限制，例如并发连接数或对该主机的呼叫失败次数。一旦达到该限制，断路器将“跳闸”并停止与该主机的进一步连接。使用断路器模式可实现快速故障，而不是使客户端尝试连接到过载或故障主机。

由于电路中断适用于负载平衡池中的“真实”网格目标，因此您可以在[目标规则中](https://istio.io/docs/concepts/traffic-management/#destination-rules)配置电路断路器阈值 ，并将设置应用于服务中的每个主机。以下示例`reviews`将v1子集的服务工作负载的并发连接数限制为100：

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
    trafficPolicy:
      connectionPool:
        tcp:
          maxConnections: 100
```



您可以在[Circuit Breaking中](https://istio.io/docs/tasks/traffic-management/circuit-breaking/)找到有关创建断路器的更多信息 。

## 7.4 Fault injection

配置完网络（包括故障恢复策略）后，可以使用Istio的故障注入机制来测试整个应用程序的故障恢复能力。故障注入是一种将错误引入系统的测试方法，以确保它可以承受并从错误情况中恢复。使用故障注入对确保您的故障恢复策略不是不兼容或限制过于严格可能特别有用，这可能会导致关键服务不可用。

与其他在网络层引入错误（例如延迟数据包或杀死Pod）的机制不同，Istio'允许您在应用程序层注入故障。这使您可以注入更多相关的故障，例如HTTP错误代码，以获得更多相关的结果。

您可以注入两种类型的故障，这两种故障都是使用[虚拟服务](https://istio.io/docs/concepts/traffic-management/#virtual-services)配置的 ：

- 延迟：延迟是计时失败。它们模仿增加的网络延迟或上游服务过载。
- 中止：中止是崩溃失败。它们模仿上游服务中的故障。中止通常以HTTP错误代码或TCP连接失败的形式表现出来。

例如，此虚拟服务为服务的每1000个请求中的1个引入5秒的延迟`ratings`。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - fault:
      delay:
        percentage:
          value: 0.1
        fixedDelay: 5s
    route:
    - destination:
        host: ratings
        subset: v1
```



有关如何配置延迟和中止的详细说明，请参阅 [故障注入](https://istio.io/docs/tasks/traffic-management/fault-injection/)。

## 7.5 处理您的应用程序

Istio故障恢复功能对应用程序是完全透明的。应用程序不知道Envoy Sidecar代理在返回响应之前是否正在处理被调用服务的故障。这意味着，如果您还在应用程序代码中设置故障恢复策略，则需要记住两者都独立工作，因此可能会发生冲突。例如，假设您有两个超时，一个在虚拟服务中配置，另一个在应用程序中配置。应用程序为服务的API调用设置了2秒超时。但是，您在虚拟服务中配置了3秒超时和1次重试。在这种情况下，应用程序的超时首先开始，因此您的Envoy超时和重试尝试无效。

尽管Istio故障恢复功能提高了网格中服务的可靠性和可用性，但应用程序必须处理故障或错误并采取适当的后备操作。例如，当负载平衡池中的所有实例都失败时，Envoy将返回一个`HTTP 503` 代码。应用程序必须实现处理`HTTP 503`错误代码所需的所有后备逻辑。







# 八 bookinfo示例

* Service

```
[root@vm_0_2_centos ~]# kubectl get svc -n istio-system istio-ingressgateway -o yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"labels":{"app":"istio-ingressgateway","chart":"gateways","heritage":"Tiller","istio":"ingressgateway","release":"istio"},"name":"istio-ingressgateway","namespace":"istio-system"},"spec":{"ports":[{"name":"status-port","port":15020,"targetPort":15020},{"name":"http2","nodePort":31380,"port":80,"targetPort":80},{"name":"https","nodePort":31390,"port":443},{"name":"tcp","nodePort":31400,"port":31400},{"name":"https-kiali","port":15029,"targetPort":15029},{"name":"https-prometheus","port":15030,"targetPort":15030},{"name":"https-grafana","port":15031,"targetPort":15031},{"name":"https-tracing","port":15032,"targetPort":15032},{"name":"tls","port":15443,"targetPort":15443}],"selector":{"app":"istio-ingressgateway","istio":"ingressgateway","release":"istio"},"type":"LoadBalancer"}}
  creationTimestamp: "2019-08-05T08:29:28Z"
  labels:
    app: istio-ingressgateway
    chart: gateways
    heritage: Tiller
    istio: ingressgateway
    release: istio
  name: istio-ingressgateway
  namespace: istio-system
  resourceVersion: "6812"
  selfLink: /api/v1/namespaces/istio-system/services/istio-ingressgateway
  uid: 6f6b6e05-6345-477f-ac16-5aedfef4e95e
spec:
  clusterIP: 10.108.162.3
  externalTrafficPolicy: Cluster
  ports:
  - name: status-port
    nodePort: 31931
    port: 15020
    protocol: TCP
    targetPort: 15020
  - name: http2
    nodePort: 31380
    port: 80
    protocol: TCP
    targetPort: 80
  - name: https
    nodePort: 31390
    port: 443
    protocol: TCP
    targetPort: 443
  - name: tcp
    nodePort: 31400
    port: 31400
    protocol: TCP
    targetPort: 31400
  - name: https-kiali
    nodePort: 32205
    port: 15029
    protocol: TCP
    targetPort: 15029
  - name: https-prometheus
    nodePort: 31540
    port: 15030
    protocol: TCP
    targetPort: 15030
  - name: https-grafana
    nodePort: 30722
    port: 15031
    protocol: TCP
    targetPort: 15031
  - name: https-tracing
    nodePort: 32366
    port: 15032
    protocol: TCP
    targetPort: 15032
  - name: tls
    nodePort: 31319
    port: 15443
    protocol: TCP
    targetPort: 15443
  selector:
    app: istio-ingressgateway
    istio: ingressgateway
    release: istio
  sessionAffinity: None
  type: NodePort
```

* bookinfo-gateway

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"networking.istio.io/v1alpha3","kind":"Gateway","metadata":{"annotations":{},"name":"bookinfo-gateway","namespace":"default"},"spec":{"selector":{"istio":"ingressgateway"},"servers":[{"hosts":["*"],"port":{"name":"http","number":80,"protocol":"HTTP"}}]}}
  creationTimestamp: null
  name: bookinfo-gateway
  namespace: default
  resourceVersion: "6469"
spec:
  selector:
    istio: ingressgateway
  servers:
  - hosts:
    - '*'
    port:
      name: http
      number: 80
      protocol: HTTP
```

* gateway

```yaml
[root@vm_0_2_centos ~]# istioctl get gateway
Command "get" is deprecated, Use `kubectl get` instead (see https://kubernetes.io/docs/tasks/tools/install-kubectl)
GATEWAY NAME       HOSTS                 NAMESPACE   AGE
bookinfo-gateway   *                     default     73d
httpbin-gateway    *                     default     72d
mygateway          httpbin.example.com   default     70d
tcp-echo-gateway   *                     default     72d
[root@vm_0_2_centos ~]# istioctl  get gateway mygateway -o yaml
Command "get" is deprecated, Use `kubectl get` instead (see https://kubernetes.io/docs/tasks/tools/install-kubectl)
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"networking.istio.io/v1alpha3","kind":"Gateway","metadata":{"annotations":{},"name":"mygateway","namespace":"default"},"spec":{"selector":{"istio":"ingressgateway"},"servers":[{"hosts":["httpbin.example.com"],"port":{"name":"https","number":443,"protocol":"HTTPS"},"tls":{"credentialName":"httpbin-credential","mode":"SIMPLE"}}]}}
  creationTimestamp: null
  name: mygateway
  namespace: default
  resourceVersion: "223848"
spec:
  selector:
    istio: ingressgateway
  servers:
  - hosts:
    - httpbin.example.com
    port:
      name: https
      number: 443
      protocol: HTTPS
    tls:
      credentialName: httpbin-credential
      mode: SIMPLE
```

* virtualservice

```yaml
[root@vm_0_2_centos ~]# istioctl get virtualservice
Command "get" is deprecated, Use `kubectl get` instead (see https://kubernetes.io/docs/tasks/tools/install-kubectl)
VIRTUAL-SERVICE NAME   GATEWAYS           HOSTS                 #HTTP     #TCP      NAMESPACE   AGE
bookinfo               bookinfo-gateway   *                         1        0      default     73d
details                                   details                   1        0      default     72d
httpbin                mygateway          httpbin.example.com       1        0      default     72d
productpage                               productpage               1        0      default     72d
ratings                                   ratings                   1        0      default     72d
reviews                                   reviews                   1        0      default     72d
tcp-echo               tcp-echo-gateway   *                         0        1      default     72d
[root@vm_0_2_centos ~]# istioctl get virtualservice bookinfo -o yaml
Command "get" is deprecated, Use `kubectl get` instead (see https://kubernetes.io/docs/tasks/tools/install-kubectl)
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"networking.istio.io/v1alpha3","kind":"VirtualService","metadata":{"annotations":{},"name":"bookinfo","namespace":"default"},"spec":{"gateways":["bookinfo-gateway"],"hosts":["*"],"http":[{"match":[{"uri":{"exact":"/productpage"}},{"uri":{"prefix":"/static"}},{"uri":{"exact":"/login"}},{"uri":{"exact":"/logout"}},{"uri":{"prefix":"/api/v1/products"}}],"route":[{"destination":{"host":"productpage","port":{"number":9080}}}]}]}}
  creationTimestamp: null
  name: bookinfo
  namespace: default
  resourceVersion: "6470"
spec:
  gateways:
  - bookinfo-gateway
  hosts:
  - '*'
  http:
  - match:
    - uri:
        exact: /productpage
    - uri:
        prefix: /static
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





# 九 http-bin服务

```yaml
[root@vm_0_2_centos ~]# kubectl get gateway
NAME               AGE
bookinfo-gateway   73d
httpbin-gateway    72d
mygateway          71d
tcp-echo-gateway   72d


[root@vm_0_2_centos ~]# kubectl get gateway httpbin-gateway -oyaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"networking.istio.io/v1alpha3","kind":"Gateway","metadata":{"annotations":{},"name":"httpbin-gateway","namespace":"default"},"spec":{"selector":{"istio":"ingressgateway"},"servers":[{"hosts":["*"],"port":{"name":"http","number":80,"protocol":"HTTP"}}]}}
  creationTimestamp: "2019-08-06T09:22:19Z"
  generation: 2
  name: httpbin-gateway
  namespace: default
  resourceVersion: "115949"
  selfLink: /apis/networking.istio.io/v1alpha3/namespaces/default/gateways/httpbin-gateway
  uid: 64be1356-6eba-430b-bfb1-defc46bfa04c
spec:
  selector:
    istio: ingressgateway
  servers:
  - hosts:
    - '*'
    port:
      name: http
      number: 80
      protocol: HTTP
      
[root@vm_0_2_centos ~]# kubectl get virtualservice
NAME          GATEWAYS             HOSTS                   AGE
bookinfo      [bookinfo-gateway]   [*]                     73d
details                            [details]               73d
httpbin       [mygateway]          [httpbin.example.com]   72d
productpage                        [productpage]           73d
ratings                            [ratings]               73d
reviews                            [reviews]               73d
tcp-echo      [tcp-echo-gateway]   [*]                     72d


[root@vm_0_2_centos ~]# kubectl get virtualservice httpbin -oyaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"networking.istio.io/v1alpha3","kind":"VirtualService","metadata":{"annotations":{},"name":"httpbin","namespace":"default"},"spec":{"gateways":["mygateway"],"hosts":["httpbin.example.com"],"http":[{"match":[{"uri":{"prefix":"/status"}},{"uri":{"prefix":"/delay"}}],"route":[{"destination":{"host":"httpbin","port":{"number":8000}}}]}]}}
  creationTimestamp: "2019-08-06T09:24:50Z"
  generation: 3
  name: httpbin
  namespace: default
  resourceVersion: "224298"
  selfLink: /apis/networking.istio.io/v1alpha3/namespaces/default/virtualservices/httpbin
  uid: 23a2bde9-8536-4aae-8d82-876f52555f89
spec:
  gateways:
  - mygateway     # 在此关联gateway
  hosts:
  - httpbin.example.com
  http:
  - match:
    - uri:
        prefix: /status
    - uri:
        prefix: /delay
    route:
    - destination:
        host: httpbin
        port:
          number: 8000
          
          

```

# 十 tcp-echo服务

```yaml
[root@vm_0_2_centos ~]# kubectl get gw
NAME               AGE
bookinfo-gateway   73d
httpbin-gateway    72d
mygateway          71d
tcp-echo-gateway   72d


[root@vm_0_2_centos ~]# kubectl get gw tcp-echo-gateway -oyaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"networking.istio.io/v1alpha3","kind":"Gateway","metadata":{"annotations":{},"name":"tcp-echo-gateway","namespace":"default"},"spec":{"selector":{"istio":"ingressgateway"},"servers":[{"hosts":["*"],"port":{"name":"tcp","number":31400,"protocol":"TCP"}}]}}
  creationTimestamp: "2019-08-06T06:01:54Z"
  generation: 1
  name: tcp-echo-gateway
  namespace: default
  resourceVersion: "100417"
  selfLink: /apis/networking.istio.io/v1alpha3/namespaces/default/gateways/tcp-echo-gateway
  uid: e04b084c-e65a-4abe-826b-c30a124f8213
spec:
  selector:
    istio: ingressgateway
  servers:
  - hosts:
    - '*'
    port:
      name: tcp
      number: 31400
      protocol: TCP
      
[root@vm_0_2_centos ~]# kubectl get vs
NAME          GATEWAYS             HOSTS                   AGE
bookinfo      [bookinfo-gateway]   [*]                     73d
details                            [details]               73d
httpbin       [mygateway]          [httpbin.example.com]   72d
productpage                        [productpage]           73d
ratings                            [ratings]               73d
reviews                            [reviews]               73d
tcp-echo      [tcp-echo-gateway]   [*]                     72d

[root@vm_0_2_centos ~]# kubectl get vs tcp-echo -oyaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"networking.istio.io/v1alpha3","kind":"VirtualService","metadata":{"annotations":{},"name":"tcp-echo","namespace":"default"},"spec":{"gateways":["tcp-echo-gateway"],"hosts":["*"],"tcp":[{"match":[{"port":31400}],"route":[{"destination":{"host":"tcp-echo","port":{"number":9000},"subset":"v1"},"weight":80},{"destination":{"host":"tcp-echo","port":{"number":9000},"subset":"v2"},"weight":20}]}]}}
  creationTimestamp: "2019-08-06T06:01:54Z"
  generation: 2
  name: tcp-echo
  namespace: default
  resourceVersion: "112275"
  selfLink: /apis/networking.istio.io/v1alpha3/namespaces/default/virtualservices/tcp-echo
  uid: c03f8d6b-0967-45cf-8cf5-465d08634318
spec:
  gateways:
  - tcp-echo-gateway
  hosts:
  - '*'
  tcp:
  - match:
    - port: 31400
    route:
    - destination:
        host: tcp-echo
        port:
          number: 9000
        subset: v1
      weight: 80
    - destination:
        host: tcp-echo
        port:
          number: 9000
        subset: v2
      weight: 20
      
[root@vm_0_2_centos ~]# kubectl get dr
NAME                   HOST          AGE
details                details       73d
productpage            productpage   73d
ratings                ratings       73d
reviews                reviews       73d
tcp-echo-destination   tcp-echo      72d


[root@vm_0_2_centos ~]# kubectl get dr tcp-echo-destination -oyaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"networking.istio.io/v1alpha3","kind":"DestinationRule","metadata":{"annotations":{},"name":"tcp-echo-destination","namespace":"default"},"spec":{"host":"tcp-echo","subsets":[{"labels":{"version":"v1"},"name":"v1"},{"labels":{"version":"v2"},"name":"v2"}]}}
  creationTimestamp: "2019-08-06T06:01:54Z"
  generation: 1
  name: tcp-echo-destination
  namespace: default
  resourceVersion: "100418"
  selfLink: /apis/networking.istio.io/v1alpha3/namespaces/default/destinationrules/tcp-echo-destination
  uid: 930406ec-acf6-4130-85e1-ebe1a0a1e179
spec:
  host: tcp-echo
  subsets:
  - labels:
      version: v1
    name: v1
  - labels:
      version: v2
    name: v2
```







# 参考链接

* https://zhaohuabing.com/istio-practice/
* https://skyao.io/learning-istio/introduction/information.html
* https://github.com/servicemesher/istio-knowledge-map









