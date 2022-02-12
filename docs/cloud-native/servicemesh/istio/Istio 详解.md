# Istio 详解

# 一 Gateway

## 1.1 简介

使用[网关](https://istio.io/latest/zh/docs/reference/config/networking/gateway/#Gateway)为网格来管理入站和出站流量，可以让您指定要进入或离开网格的流量。网关配置被用于运行在网格边界的独立 Envoy 代理，而不是服务工作负载的 sidecar 代理。

Istio 的网关资源可以配置 4-6 层的负载均衡属性，如对外暴露的端口、TLS 设置等。作为替代应用层流量路由（L7）到相同的 API 资源，您绑定了一个常规的 Istio [虚拟服务](https://istio.io/latest/zh/docs/concepts/traffic-management/#virtual-services)到网关。这让您可以像管理网格中其他数据平面的流量一样去管理网关流量。

网关主要用于管理进入的流量，但您也可以配置出口网关。出口网关让您为离开网格的流量配置一个专用的出口节点，这可以限制哪些服务可以或应该访问外部网络，或者启用[出口流量安全控制](https://istio.io/latest/zh/blog/2019/egress-traffic-control-in-istio-part-1/)为您的网格添加安全性。您也可以使用网关配置一个纯粹的内部代理。

Istio 提供了一些预先配置好的网关代理部署（`istio-ingressgateway` 和 `istio-egressgateway`）

## 1.2 配置详解

* Gateway

| Field      | Type                  | Description                                                  | Required |
| ---------- | --------------------- | ------------------------------------------------------------ | -------- |
| `servers`  | `Server[]`            | A list of server specifications.                             | Yes      |
| `selector` | `map<string, string>` | One or more labels that indicate a specific set of pods/VMs on which this gateway configuration should be applied. The scope of label search is restricted to the configuration namespace in which the the resource is present. In other words, the Gateway resource must reside in the same namespace as the gateway workload instance. | Yes      |

* Port

| Field      | Type     | Description                                                  | Required |
| ---------- | -------- | ------------------------------------------------------------ | -------- |
| `number`   | `uint32` | A valid non-negative integer port number.                    | Yes      |
| `protocol` | `string` | The protocol exposed on the port. MUST BE one of HTTP\|HTTPS\|GRPC\|HTTP2\|MONGO\|TCP\|TLS. TLS implies the connection will be routed based on the SNI header to the destination without terminating the TLS connection. | Yes      |
| `name`     | `string` | Label assigned to the port.                                  | No       |

* Server

| Field             | Type         | Description                                                  | Required |
| ----------------- | ------------ | ------------------------------------------------------------ | -------- |
| `port`            | `Port`       | The Port on which the proxy should listen for incoming connections. | Yes      |
| `hosts`           | `string[]`   | One or more hosts exposed by this gateway. While typically applicable to HTTP services, it can also be used for TCP services using TLS with SNI. A host is specified as a `dnsName` with an optional `namespace/` prefix. The `dnsName` should be specified using FQDN format, optionally including a wildcard character in the left-most component (e.g., `prod/*.example.com`). Set the `dnsName` to `*` to select all `VirtualService` hosts from the specified namespace (e.g.,`prod/*`).The `namespace` can be set to `*` or `.`, representing any or the current namespace, respectively. For example, `*/foo.example.com` selects the service from any available namespace while `./foo.example.com` only selects the service from the namespace of the sidecar. The default, if no `namespace/` is specified, is `*/`, that is, select services from any namespace. Any associated `DestinationRule` in the selected namespace will also be used.A `VirtualService` must be bound to the gateway and must have one or more hosts that match the hosts specified in a server. The match could be an exact match or a suffix match with the server’s hosts. For example, if the server’s hosts specifies `*.example.com`, a `VirtualService` with hosts `dev.example.com` or `prod.example.com` will match. However, a `VirtualService` with host `example.com` or `newexample.com` will not match.NOTE: Only virtual services exported to the gateway’s namespace (e.g., `exportTo` value of `*`) can be referenced. Private configurations (e.g., `exportTo` set to `.`) will not be available. Refer to the `exportTo` setting in `VirtualService`, `DestinationRule`, and `ServiceEntry` configurations for details. | Yes      |
| `tls`             | `TLSOptions` | Set of TLS related options that govern the server’s behavior. Use these options to control if all http requests should be redirected to https, and the TLS modes to use. | No       |
| `defaultEndpoint` | `string`     | The loopback IP endpoint or Unix domain socket to which traffic should be forwarded to by default. Format should be `127.0.0.1:PORT` or `unix:///path/to/socket` or `unix://@foobar` (Linux abstract namespace). | No       |

* Server.TLSOptions

| Field                   | Type          | Description                                                  | Required |
| ----------------------- | ------------- | ------------------------------------------------------------ | -------- |
| `httpsRedirect`         | `bool`        | If set to true, the load balancer will send a 301 redirect for all http connections, asking the clients to use HTTPS. | No       |
| `mode`                  | `TLSmode`     | Optional: Indicates whether connections to this port should be secured using TLS. The value of this field determines how TLS is enforced. | No       |
| `serverCertificate`     | `string`      | REQUIRED if mode is `SIMPLE` or `MUTUAL`. The path to the file holding the server-side TLS certificate to use. | No       |
| `privateKey`            | `string`      | REQUIRED if mode is `SIMPLE` or `MUTUAL`. The path to the file holding the server’s private key. | No       |
| `caCertificates`        | `string`      | REQUIRED if mode is `MUTUAL`. The path to a file containing certificate authority certificates to use in verifying a presented client side certificate. | No       |
| `credentialName`        | `string`      | The credentialName stands for a unique identifier that can be used to identify the serverCertificate and the privateKey. The credentialName appended with suffix “-cacert” is used to identify the CaCertificates associated with this server. Gateway workloads capable of fetching credentials from a remote credential store such as Kubernetes secrets, will be configured to retrieve the serverCertificate and the privateKey using credentialName, instead of using the file system paths specified above. If using mutual TLS, gateway workload instances will retrieve the CaCertificates using credentialName-cacert. The semantics of the name are platform dependent. In Kubernetes, the default Istio supplied credential server expects the credentialName to match the name of the Kubernetes secret that holds the server certificate, the private key, and the CA certificate (if using mutual TLS). Set the `ISTIO_META_USER_SDS` metadata variable in the gateway’s proxy to enable the dynamic credential fetching feature. | No       |
| `subjectAltNames`       | `string[]`    | A list of alternate names to verify the subject identity in the certificate presented by the client. | No       |
| `verifyCertificateSpki` | `string[]`    | An optional list of base64-encoded SHA-256 hashes of the SKPIs of authorized client certificates. Note: When both verify*certificate*hash and verify*certificate*spki are specified, a hash matching either value will result in the certificate being accepted. | No       |
| `verifyCertificateHash` | `string[]`    | An optional list of hex-encoded SHA-256 hashes of the authorized client certificates. Both simple and colon separated formats are acceptable. Note: When both verify*certificate*hash and verify*certificate*spki are specified, a hash matching either value will result in the certificate being accepted. | No       |
| `minProtocolVersion`    | `TLSProtocol` | Optional: Minimum TLS protocol version.                      | No       |
| `maxProtocolVersion`    | `TLSProtocol` | Optional: Maximum TLS protocol version.                      | No       |
| `cipherSuites`          | `string[]`    | Optional: If specified, only support the specified cipher list. Otherwise default to the default cipher list supported by Envoy. | No       |

* Server.TLSOptions.TLSProtocol

| Name       | Description                                   |
| ---------- | --------------------------------------------- |
| `TLS_AUTO` | Automatically choose the optimal TLS version. |
| `TLSV1_0`  | TLS version 1.0                               |
| `TLSV1_1`  | TLS version 1.1                               |
| `TLSV1_2`  | TLS version 1.2                               |
| `TLSV1_3`  | TLS version 1.3                               |

* Server.TLSOptions.TLSmode

| Name               | Description                                                  |
| ------------------ | ------------------------------------------------------------ |
| `PASSTHROUGH`      | The SNI string presented by the client will be used as the match criterion in a VirtualService TLS route to determine the destination service from the service registry. |
| `SIMPLE`           | Secure connections with standard TLS semantics.              |
| `MUTUAL`           | Secure connections to the downstream using mutual TLS by presenting server certificates for authentication. |
| `AUTO_PASSTHROUGH` | Similar to the passthrough mode, except servers with this TLS mode do not require an associated VirtualService to map from the SNI value to service in the registry. The destination details such as the service/subset/port are encoded in the SNI value. The proxy will forward to the upstream (Envoy) cluster (a group of endpoints) specified by the SNI value. This server is typically used to provide connectivity between services in disparate L3 networks that otherwise do not have direct connectivity between their respective endpoints. Use of this mode assumes that both the source and the destination are using Istio mTLS to secure traffic. |
| `ISTIO_MUTUAL`     | Secure connections from the downstream using mutual TLS by presenting server certificates for authentication. Compared to Mutual mode, this mode uses certificates, representing gateway workload identity, generated automatically by Istio for mTLS authentication. When this mode is used, all other fields in `TLSOptions` should be empty. |

## 1.3 示例

gateway中的selector 指定类型为ingressgateway或是egressgateway，当然可以制定将gateway部署在k8s的一个ns中。

* 简单实例

使用hosts字段来列举虚拟服务的主机，类似于web服务器的虚拟主机--即用户指定的目标活是路由规则设定的目标，这是客户端想服务器发送请求时使用的一个或多个地址。

虚拟服务主机名称可以是IP地址、DNS名称，或者依赖于平台的一个简称（例如k8s中的svc的短名称），隐式或显式地执行一个完全限定域名（FQDN），hosts字段需要为浏览器请求访问的hosts，如果为IP访问可以临时指定为`* `

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: bookinfo-gateway
spec:
  selector:
    istio: ingressgateway # use istio default controller
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
```

* TLS证书

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

# 二 VirtualService

## 2.1 简介

虚拟服务在增强 Istio 流量管理的灵活性和有效性方面发挥着重要作用，可以配置超时，重试，黑白名单，故障注入，基于百分比的流量分发，金丝雀发布等。

## 2.2 配置详解

https://istio.io/zh/docs/reference/config/networking/virtual-service/

## 2.3 示例

virtualservice 对应一个svc，vs是在svc上进行来app/version的区别，区分开来subnet，                                                                     

一个svc对应一个endpoints，但是一个endpoints利用app/version分开了多个subnet，也就是deploy，也就是不同版本的pod

服务间通信还是使用的默认K8s的svc

* url分发

可以根据流量端口、header字段、URI等内容上设置匹配条件。

```yaml
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

* 请求超时

超时设置式微了确保服务不会因等待答复而无限期刮起，在可预测的时间范围内调用成功或失败，http请求默认超时15秒。

```yaml
# 请求超时
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

* 重试

重试指如果初始调用失败，envoy代理尝试连接服务的最大次数，避免调用不会因临时过载的服务或网络问题问永久失败，重试可以提供服务可用性和应用程序性能，重试之间的间隔(25ms+)是可变的，并由istio自动确认，从而防止被调用服务被请求淹没，默认情况下如果不配置重试策略，不会进行重试。

细化重试，重试3次来连接到服务子集，每个重试都有2秒的超时。

```yaml
# 重试

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

* 故障注入

在配置的网络包括故障恢复策略后，可以使用istio的故障注入机制来队整个应用系统测试故障恢复能力。故障注入是将一种错误引入系统以确保系统能够承载并从错误条件中恢复的测试方法，使用故障注入特别有用，能确保故障恢复策略不至于不兼容或太严格导致服务不可用。

故障注入分为两类：

延迟：延迟是时间故障，它们模拟增加的网络延迟或一个超载的上游服务。

终止：终止是奔溃失败，它们模拟上游服务失败，终止通常以HTTP错误码或TCP连接失败形式出现。

```yaml
# 故障注入之延迟
spec:
  hosts:
  - ratings
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    fault:
      delay:
        percentage:
          value: 100.0
        fixedDelay: 7s
    route:
    - destination:
        host: ratings
        subset: v1    
        
# 故障注入之丢弃
spec:
  hosts:
  - ratings
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    fault:
      abort:
        percentage:
          value: 100.0
        httpStatus: 500
    route:
    - destination:
        host: ratings
        subset: v1 
        

# 基于百分比的流量分发，常用于金丝雀发布
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

* 流量镜像

可以看到，该配置指定了镜像流量需要发送的目标服务地址为`serviceB`。`mirror.subset`配置项配置一个`Service B`服务的服务子集名称 ，指定了要将镜像流量镜像到`v2`版本的`Service B`服务子集中去。`mirror_percent`配置将`100%`的真实流量进行镜像发送。所以下面的配置整体表示当流量到来时，将请求转发到`v1`版本的`service B`服务子集中，再以镜像的方式发送到`v2`版本的`service B`服务上一份，并将真实流量全部镜像。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: serviceB
spec:
  hosts:
  - istio.gateway.xxxx.tech
  gateways:
  - ingressgateway.istio-system.svc.cluster.local
  http:
  - match:
    - uri:
        prefix: /serviceB
    rewrite:
      uri: /
    route:
    - destination:
        host: serviceB
        subset: v1
    mirror:
      host: serviceB
      subset: v2
    mirror_percent: 100
```



# 三 DestinationRule

## 3.1 简介



## 3.2 配置详解

虚拟服务为将流量如何路由到给定目标地址，然后根据目标规则来对流量进行配置，可以使用目标规则来指定命名的服务子集，例如按照版本为所给定的服务实例进行分组，然后可以在虚拟服务的路由规则中使用这些服务子集来控制服务不同实例的流量，例如选择负载均衡模型，TLS安全模式活熔断设置。

默认情况istio使用轮训负载均衡策略，

* 随机：请求以随机方式转发到后端一组实例。
* 权重：请求根据指定百分比转到实例。
* 最少请求；请求被转到最少被访问的实例。

## 3.3 示例

dr是作用在一个svc，一个endpoints后，根据label的app/version标签将一个svc后的多个deploy也就是pod进行分组为subset，然后可以设置负载均衡策略，TLS安全和断路器等。其中host字段为 k8s的svc。

![image-20200611153855483](/Users/xuel/Library/Application Support/typora-user-images/image-20200611153855483.png)

https://istio.io/zh/docs/reference/config/networking/destination-rule/

* 负载均衡

其中对于subsets上的v1和v3子集为随机，v2位轮询

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

* 断路器

限制并发数位100

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





# 四 ServiceEntry

将调用外部得一个服务，注册再改istio服务网格内，可以用istio得服务治理，例如重试，超时，故障注入，

apiVersion: networking.istio.io/v1alpha3 kind: ServiceEntry metadata:   name: svc-entry spec:   hosts:   - ext-svc.example.com   ports:   - number: 443     name: https     protocol: HTTPS   location: MESH_EXTERNAL   resolution: DNS 



# 注意事项

* deployment需要用app/version两个标签来表识
* 服务间调用使用k8s原生svc
* istio控制管理的对象为svc，没有svc的deployment是无法被istio发现并操作的
* 一个gateway关联多个svc，一个svc关联一个endpoints，一个endpoints有多个deploy，也就是多个pods