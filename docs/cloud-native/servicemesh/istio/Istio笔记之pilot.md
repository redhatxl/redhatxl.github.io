# Istio笔记之pilot

# 一 基本概念

[Pilot](https://istio.io/zh/docs/concepts/traffic-management/#pilot-和-envoy) 为 Envoy sidecar 提供服务发现功能，为智能路由（例如 A/B 测试、金丝雀部署等）和弹性（超时、重试、熔断器等）提供流量管理功能。它将控制流量行为的高级路由规则转换为特定于 Envoy 的配置，并在运行时将它们传播到 sidecar。

Pilot 将平台特定的服务发现机制抽象化并将其合成为符合 [Envoy 数据平面 API](https://github.com/envoyproxy/data-plane-api) 的任何 sidecar 都可以使用的标准格式。这种松散耦合使得 Istio 能够在多种环境下运行（例如，Kubernetes、Consul、Nomad），同时保持用于流量管理的相同操作界面。

# 二 核心功能

* 服务发现
* 配置管理

# 三 服务发现流程

![image-20191019114253981](/Users/xuel/Library/Application Support/typora-user-images/image-20191019114253981.png)

服务注册表：Pilot从平台获取服务发现数据，并提供统一的发现接口

服务发现：Envoy实现服务发现，动态更新负载均衡池。服务请求时使用对应的负载均衡策略将请求路由到对应的后端。

服务发现流程：service A访问service B出方向流量经过Proxy，proxy去请求Pilot servies拿到service b的服务地址以及负载均衡策略，然后发起请求，service b的入流量被service b伤的proxy拦截可以执行一系列的服务治理操作。proxy采用envoy和服务在一起，请求流量通过proxy， 但是service a访问其他service 是感知不到proxy



# 四 Pilot服务发现机制

* k8s的服务发现

我们都知道k8s其实是具备服务发现机制，svca访问svcb通过serviceip，然后请求到达node的kube-proxy上，kube-proxy又从kube-APIServer上获取集群其他service的后端backend示例，实现服务发现

![image-20191019120932485](/Users/xuel/Library/Application Support/typora-user-images/image-20191019120932485.png)共一个ns下可以直接短域名访问，跨ns，需要带上ns，service.namespace.svc.cluster.local

![image-20191019121108096](/Users/xuel/Library/Application Support/typora-user-images/image-20191019121108096.png)

* Adapter机制，Pilot其中只有服务发现的定义，没有服务发现的实现，提供了适应与各种平台及其他服务发现组件的适配器 ，服务发现的实现位adapter对接的平台或这其他服务发现的后端。

![image-20191019115305607](/Users/xuel/Library/Application Support/typora-user-images/image-20191019115305607.png)

* pilot基于kubernetes的服务发现，istio虽然具备可扩展性，能够结合其他服务发现组件，但是结合用户场景，其支持最好的还是kuberenets，在新版本istio会移除eureka的支持，在此为们着重研究pilot在kubernetes服务发现机制

![image-20191019115529297](/Users/xuel/Library/Application Support/typora-user-images/image-20191019115529297.png)

1. pilot实现若干服务发现的接口定义
2. pilot的controller list/watch kubernetesAPIserver上service/endpoint等资源对象并转换pilot的标准格式
3. envoy从pilot获取xDS，动态更新
4. 当要服务访问时，envoy在处理outbound请求时，根据配置的LB策略，选择一个服务实例发起访问。

# 五 kubernetes与istio服务模型对比

![image-20191019120056605](/Users/xuel/Library/Application Support/typora-user-images/image-20191019120056605.png)

istio的service就是k8s的service，istio对应的服务模型instance对应于k8s中的endpoint



![image-20191019120114996](/Users/xuel/Library/Application Support/typora-user-images/image-20191019120114996.png)

istio的version对应于k8s中的deployment

# 六 istio应用架构在k8s上的实现

控制层面：pilot去list/watch k8s的apiservice，用户可以通过crd来通过istioctl或kubectl执行命令，获取etcd中的资源

数据层面：envoy拦截pod中的请求从pilot获取配置进行服务治理操作

![image-20191019121751509](/Users/xuel/Library/Application Support/typora-user-images/image-20191019121751509.png)

# 七 Istio配置管理

![image-20191019122200403](/Users/xuel/Library/Application Support/typora-user-images/image-20191019122200403.png)

1. 配置：管理员通过Pilot配置服务治理规则
2. 下发：Envoy从Pilot获取规则
3. 执行：在流量访问的时候执行治理规则，在SvcA服务发起方的envoy中执行

# 八 Istio服务治理规则

## 8.1 VirtualService

服务访问路由控制，满足特定条件的请求路由到哪里，过程中治理，包括重写，重试，故障注入等

![image-20191019122905176](/Users/xuel/Library/Application Support/typora-user-images/image-20191019122905176.png)

## 8.2 DestinationRule

目标服务的策略，包括目标服务的负载均衡，链接池管理

![image-20191019123632241](/Users/xuel/Library/Application Support/typora-user-images/image-20191019123632241.png)

## 8.3 ServiceEntry 和gateway

![image-20191019124312607](/Users/xuel/Library/Application Support/typora-user-images/image-20191019124312607.png)

# 九 Istio配置规则维护和下发流程

![image-20191019124635230](/Users/xuel/Library/Application Support/typora-user-images/image-20191019124635230.png)

用户通过istioctl kubectl或者调用api操作crd来将治理规则下方到k8s中，存储在etcd中，pilot的configcontrolle一直去list/watch k8s的apiserver，Pilot的discoveryserver来从配置中拿到配置转换成envoy识别的规则，然后在envoy进行服务治理

# 十 Istio服务治理能力执行位置

![image-20191019125321507](/Users/xuel/Library/Application Support/typora-user-images/image-20191019125321507.png)w