# istio笔记

# 一 基础概念

## 1.1 istio是什么

Istio 允许您连接、保护、控制和观测微服务，在较高的层次上，Istio 有助于降低这些部署的复杂性，并减轻开发团队的压力。它是一个完全开源的服务网格，可以透明地分层到现有的分布式应用程序上。它也是一个平台，包括允许它集成到任何日志记录平台、遥测或策略系统的 API。Istio 的多样化功能集使您能够成功高效地运行分布式微服务架构，并提供保护、连接和监控微服务的统一方法。

## 1.2 什么是service mesh

在从单体应用程序向分布式微服务架构的转型过程中，开发人员和运维人员面临诸多挑战，使用 Istio 可以解决这些问题。

服务网格（Service Mesh）这个术语通常用于描述构成这些应用程序的微服务网络以及应用之间的交互。随着规模和复杂性的增长，服务网格越来越难以理解和管理。它的需求包括服务发现、负载均衡、故障恢复、指标收集和监控以及通常更加复杂的运维需求，例如 A/B 测试、金丝雀发布、限流、访问控制和端到端认证等。

Istio 提供了一个完整的解决方案，通过为整个服务网格提供行为洞察和操作控制来满足微服务应用程序的多样化需求。

## 1.3 为什么使用istio

Istio 提供一种简单的方式来为已部署的服务建立网络，该网络具有负载均衡、服务间认证、监控等功能，只需要对服务的代码进行[一点](https://istio.io/zh/docs/tasks/telemetry/distributed-tracing/overview/#trace-context-propagation)或不需要做任何改动。想要让服务支持 Istio，只需要在您的环境中部署一个特殊的 sidecar 代理，使用 Istio 控制平面功能配置和管理代理，拦截微服务之间的所有网络通信：

- HTTP、gRPC、WebSocket 和 TCP 流量的自动负载均衡。
- 通过丰富的路由规则、重试、故障转移和故障注入，可以对流量行为进行细粒度控制。
- 可插入的策略层和配置 API，支持访问控制、速率限制和配额。
- 对出入集群入口和出口中所有流量的自动度量指标、日志记录和追踪。
- 通过强大的基于身份的验证和授权，在集群中实现安全的服务间通信。

Istio 旨在实现可扩展性，满足各种部署需求。

# 二 核心功能

## 2.1 流量管理

通过简单的规则配置和流量路由，您可以控制服务之间的流量和 API 调用。Istio 简化了断路器、超时和重试等服务级别属性的配置，并且可以轻松设置 A/B测试、金丝雀部署和基于百分比的流量分割的分阶段部署等重要任务。

通过更好地了解您的流量和开箱即用的故障恢复功能，您可以在问题出现之前先发现问题，使调用更可靠，并且使您的网络更加强大——无论您面临什么条件。

## 2.2 安全

Istio 的安全功能使开发人员可以专注于应用程序级别的安全性。Istio 提供底层安全通信信道，并大规模管理服务通信的认证、授权和加密。使用Istio，服务通信在默认情况下是安全的，它允许您跨多种协议和运行时一致地实施策略——所有这些都很少或根本不需要应用程序更改。

虽然 Istio 与平台无关，但将其与 Kubernetes（或基础架构）网络策略结合使用，其优势会更大，包括在网络和应用层保护 pod 间或服务间通信的能力。

## 2.3 可观察行

Istio 强大的追踪、监控和日志记录可让您深入了解服务网格部署。通过 Istio 的监控功能，可以真正了解服务性能如何影响上游和下游的功能，而其自定义仪表板可以提供对所有服务性能的可视性，并让您了解该性能如何影响您的其他进程。

Istio 的 Mixer 组件负责策略控制和遥测收集。它提供后端抽象和中介，将 Istio 的其余部分与各个基础架构后端的实现细节隔离开来，并为运维提供对网格和基础架构后端之间所有交互的细粒度控制。

所有这些功能可以让您可以更有效地设置、监控和实施服务上的 SLO。当然，最重要的是，您可以快速有效地检测和修复问题。

## 2.4 平台支持

stio 是独立于平台的，旨在运行在各种环境中，包括跨云、内部部署、Kubernetes、Mesos 等。您可以在 Kubernetes 上部署 Istio 或具有 Consul 的 Nomad 上部署。Istio 目前支持：

- 在 Kubernetes 上部署的服务
- 使用 Consul 注册的服务
- 在虚拟机上部署的服务

## 2.5 集成和定制

策略执行组件可以扩展和定制，以便与现有的 ACL、日志、监控、配额、审计等方案集成。

# 三 架构

Istio 服务网格逻辑上分为**数据平面**和**控制平面**。

- **数据平面**由一组以 sidecar 方式部署的智能代理（[Envoy](https://www.envoyproxy.io/)）组成。这些代理可以调节和控制微服务及 [Mixer](https://istio.io/zh/docs/concepts/policies-and-telemetry/) 之间所有的网络通信。
- **控制平面**负责管理和配置代理来路由流量。此外控制平面配置 Mixer 以实施策略和收集遥测数据。

## 3.1 架构图

![image-20190803233405428](/Users/xuel/Library/Application Support/typora-user-images/image-20190803233405428.png)

## 3.2 组件

### 3.2.1 Envoy

Istio 使用 [Envoy](https://www.envoyproxy.io/) 代理的扩展版本，Envoy 是以 C++ 开发的高性能代理，用于调解服务网格中所有服务的所有入站和出站流量。Envoy 的许多内置功能被 Istio 发扬光大，例如：

- 动态服务发现
- 负载均衡
- TLS 终止
- HTTP/2 & gRPC 代理
- 熔断器
- 健康检查、基于百分比流量拆分的灰度发布
- 故障注入
- 丰富的度量指标

Envoy 被部署为 **sidecar**，和对应服务在同一个 Kubernetes pod 中。这允许 Istio 将大量关于流量行为的信号作为[属性](https://istio.io/zh/docs/concepts/policies-and-telemetry/#属性)提取出来，而这些属性又可以在 [Mixer](https://istio.io/zh/docs/concepts/policies-and-telemetry/) 中用于执行策略决策，并发送给监控系统，以提供整个网格行为的信息。

Sidecar 代理模型还可以将 Istio 的功能添加到现有部署中，而无需重新构建或重写代码。可以阅读更多来了解为什么我们在[设计目标](https://istio.io/zh/docs/concepts/what-is-istio/#设计目标)中选择这种方式。

### 3.2.2 Mixer

[Mixer](https://istio.io/zh/docs/concepts/policies-and-telemetry/) 是一个独立于平台的组件，负责在服务网格上执行访问控制和使用策略，并从 Envoy 代理和其他服务收集遥测数据。代理提取请求级[属性](https://istio.io/zh/docs/concepts/policies-and-telemetry/#属性)，发送到 Mixer 进行评估。有关属性提取和策略评估的更多信息，请参见 [Mixer 配置](https://istio.io/zh/docs/concepts/policies-and-telemetry/#配置模型)。

Mixer 中包括一个灵活的插件模型，使其能够接入到各种主机环境和基础设施后端，从这些细节中抽象出 Envoy 代理和 Istio 管理的服务。

### 3.2.3 Pilot

[Pilot](https://istio.io/zh/docs/concepts/traffic-management/#pilot-和-envoy) 为 Envoy sidecar 提供服务发现功能，为智能路由（例如 A/B 测试、金丝雀部署等）和弹性（超时、重试、熔断器等）提供流量管理功能。它将控制流量行为的高级路由规则转换为特定于 Envoy 的配置，并在运行时将它们传播到 sidecar。

Pilot 将平台特定的服务发现机制抽象化并将其合成为符合 [Envoy 数据平面 API](https://github.com/envoyproxy/data-plane-api) 的任何 sidecar 都可以使用的标准格式。这种松散耦合使得 Istio 能够在多种环境下运行（例如，Kubernetes、Consul、Nomad），同时保持用于流量管理的相同操作界面。

### 3.2.4 Citadel

[Citadel](https://istio.io/zh/docs/concepts/security/) 通过内置身份和凭证管理赋能强大的服务间和最终用户身份验证。可用于升级服务网格中未加密的流量，并为运维人员提供基于服务标识而不是网络控制的强制执行策略的能力。从 0.5 版本开始，Istio 支持[基于角色的访问控制](https://istio.io/zh/docs/concepts/security/#认证)，以控制谁可以访问您的服务，而不是基于不稳定的三层或四层网络标识。

### 3.2.5 Galley

Galley 代表其他的 Istio 控制平面组件，用来验证用户编写的 Istio API 配置。随着时间的推移，Galley 将接管 Istio 获取配置、 处理和分配组件的顶级责任。它将负责将其他的 Istio 组件与从底层平台（例如 Kubernetes）获取用户配置的细节中隔离开来。

## 3.3 设计目标

- **最大化透明度**：若想 Istio 被采纳，应该让运维和开发人员只需付出很少的代价就可以从中受益。为此，Istio 将自身自动注入到服务间所有的网络路径中。Istio 使用 sidecar 代理来捕获流量，并且在尽可能的地方自动编程网络层，以路由流量通过这些代理，而无需对已部署的应用程序代码进行任何改动。在 Kubernetes中，代理被注入到 pod 中，通过编写 iptables 规则来捕获流量。注入 sidecar 代理到 pod 中并且修改路由规则后，Istio 就能够调解所有流量。这个原则也适用于性能。当将 Istio 应用于部署时，运维人员可以发现，为提供这些功能而增加的资源开销是很小的。所有组件和 API 在设计时都必须考虑性能和规模。
- **可扩展性**：随着运维人员和开发人员越来越依赖 Istio 提供的功能，系统必然和他们的需求一起成长。虽然我们期望继续自己添加新功能，但是我们预计最大的需求是扩展策略系统，集成其他策略和控制来源，并将网格行为信号传播到其他系统进行分析。策略运行时支持标准扩展机制以便插入到其他服务中。此外，它允许扩展词汇表，以允许基于网格生成的新信号来执行策略。
- **可移植性**：使用 Istio 的生态系统将在很多维度上有差异。Istio 必须能够以最少的代价运行在任何云或预置环境中。将基于 Istio 的服务移植到新环境应该是轻而易举的，而使用 Istio 将一个服务同时部署到多个环境中也是可行的（例如，在多个云上进行冗余部署）。
- **策略一致性**：在服务间的 API 调用中，策略的应用使得可以对网格间行为进行全面的控制，但对于无需在 API 级别表达的资源来说，对资源应用策略也同样重要。例如，将配额应用到 ML 训练任务消耗的 CPU 数量上，比将配额应用到启动这个工作的调用上更为有用。因此，策略系统作为独特的服务来维护，具有自己的 API，而不是将其放到代理/sidecar 中，这容许服务根据需要直接与其集成。



# 四 安装部署

## 4.1 katacoda 部署istio

```shell
# 安装cli工具
curl -L https://git.io/getLatestIstio | ISTIO_VERSION=1.0.0 sh -
export PATH="$PATH:/root/istio-1.0.0/bin"
cd /root/istio-1.0.0

# 配置istio crd
kubectl apply -f install/kubernetes/helm/istio/templates/crds.yaml -n istio-system

# 配置TLS
kubectl apply -f install/kubernetes/istio-demo-auth.yaml

# 检测配置
kubectl get pods -n istio-system

# 配置Katacoda Service
kubectl apply -f /root/katacoda.yaml

# 安装配置实例
kubectl apply -f <(istioctl kube-inject -f samples/bookinfo/platform/kube/bookinfo.yaml)

# 部署网关
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml

# 应用默认规则
kubectl apply -f samples/bookinfo/networking/destination-rule-all-mtls.yaml



# 查看虚拟服务
istioctl get virtualservices

```

![image-20190804095128570](/Users/xuel/Library/Application Support/typora-user-images/image-20190804095128570.png)





## 4.2 minikube 中体验istio

安装脚本

```shell
curl -o /etc/yum.repos.d/docker-ce.repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
cat >>/etc/yum.repos.d/kuberetes.repo<<EOF
[kuberneres]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
enabled=1
EOF
yum install docker-ce kubelet kubectl kubeadm -y
systemctl start docker
systemctl enable docker
systemctl enable kubelet
# 下载minikube
curl -Lo minikube http://kubernetes.oss-cn-hangzhou.aliyuncs.com/minikube/releases/v1.2.0/minikube-linux-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/

# 启动--vm-driver=none 选项来在本机运行 Kubernetes 组件
minikube start --vm-driver=none --registry-mirror=https://registry.docker-cn.com
minikube dashboard
```

### 4.2.1 minikube安装

注意：服务器至少2颗CPU,由于强的原因，国内建议使用阿里的源，在linux服务器不启动virtualbox，则需要在宿主机安装docker和kubelet

- 设置 docker 源

```bash
curl -o /etc/yum.repos.d/docker-ce.repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

- 设置 k8s 源

```bash
cat >>/etc/yum.repos.d/kuberetes.repo<<EOF
[kuberneres]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
enabled=1
EOF
```

- 安装 docker-ce 和 kubelet kubectl

```bash
yum install docker-ce kubelet kubectl kubeadm -y
systemctl start docker
systemctl enable docker
systemctl enable kubelet
# 下载minikube
curl -Lo minikube http://kubernetes.oss-cn-hangzhou.aliyuncs.com/minikube/releases/v1.2.0/minikube-linux-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/

# 启动--vm-driver=none 选项来在本机运行 Kubernetes 组件
minikube start --vm-driver=none --registry-mirror=https://registry.docker-cn.com

sudo mv /root/.kube /root/.minikube $HOME
sudo chown -R $USER $HOME/.kube $HOME/.minikube
# 启动dashboard
minikube dashboard
```

![image-20190805155430847](/Users/xuel/Library/Application Support/typora-user-images/image-20190805155430847.png)

### 4.2.2 istio 安装

* 下载istio包

下载 Istio 发布包,

```shell
curl -L https://git.io/getLatestIstio | ISTIO_VERSION=1.2.2 sh -
cd istio-1.2.2
# 将istioctl 添加入PATH
cp bin/istioctl /usr/bin/
export PATH=$PWD/bin:$PATH

# 加载istioctl mainfest	
```

* 安装

```shell
# 安装CRD
for i in install/kubernetes/helm/istio-init/files/crd*yaml; do kubectl apply -f $i; done

# 宽松模式mutual TLS
kubectl apply -f install/kubernetes/istio-demo.yaml



# 确认结果
kubectl get svc -n istio-system
# 查看istio-ingressgateway 如果没有lb，则为一种pending状态，可以将其修改为NodePort模式

[root@VM_0_2_centos istio-1.2.2]# kubectl get service -n istio-system
NAME                     TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                                                                                                                                      AGE
grafana                  ClusterIP   10.98.106.233    <none>        3000/TCP                                                                                                                                     29m
istio-citadel            ClusterIP   10.105.117.162   <none>        8060/TCP,15014/TCP                                                                                                                           29m
istio-egressgateway      ClusterIP   10.96.101.47     <none>        80/TCP,443/TCP,15443/TCP                                                                                                                     29m
istio-galley             ClusterIP   10.97.227.4      <none>        443/TCP,15014/TCP,9901/TCP                                                                                                                   29m
istio-ingressgateway     NodePort    10.108.162.3     <none>        15020:31931/TCP,80:31380/TCP,443:31390/TCP,31400:31400/TCP,15029:32205/TCP,15030:31540/TCP,15031:30722/TCP,15032:32366/TCP,15443:31319/TCP   29m
istio-pilot              ClusterIP   10.111.150.44    <none>        15010/TCP,15011/TCP,8080/TCP,15014/TCP                                                                                                       29m
istio-policy             ClusterIP   10.101.26.145    <none>        9091/TCP,15004/TCP,15014/TCP                                                                                                                 29m
istio-sidecar-injector   ClusterIP   10.110.244.53    <none>        443/TCP                                                                                                                                      29m
istio-telemetry          ClusterIP   10.100.238.134   <none>        9091/TCP,15004/TCP,15014/TCP,42422/TCP                                                                                                       29m
jaeger-agent             ClusterIP   None             <none>        5775/UDP,6831/UDP,6832/UDP                                                                                                                   29m
jaeger-collector         ClusterIP   10.99.246.30     <none>        14267/TCP,14268/TCP                                                                                                                          29m
jaeger-query             ClusterIP   10.109.225.169   <none>        16686/TCP                                                                                                                                    29m
kiali                    ClusterIP   10.106.46.153    <none>        20001/TCP                                                                                                                                    29m
prometheus               ClusterIP   10.101.148.126   <none>        9090/TCP                                                                                                                                     29m
tracing                  ClusterIP   10.109.118.137   <none>        80/TCP                                                                                                                                       29m
zipkin                   ClusterIP   10.109.133.46    <none>        9411/TCP                                                                                                                                     29m


# 确认必要的 Kubernetes Pod 都已经创建并且其 STATUS 的值是 Running：

[root@VM_0_2_centos istio-1.2.2]# kubectl get pods -n istio-system
NAME                                      READY   STATUS      RESTARTS   AGE
grafana-6575997f54-zlq9d                  1/1     Running     0          10m
istio-citadel-894d98c85-hqd8m             1/1     Running     0          10m
istio-cleanup-secrets-1.2.2-6qdt8         0/1     Completed   0          10m
istio-egressgateway-9b7866bf5-26q2n       1/1     Running     0          10m
istio-galley-5b984f89b-95c4c              1/1     Running     0          10m
istio-grafana-post-install-1.2.2-lv7xg    0/1     Completed   0          10m
istio-ingressgateway-75ddf64567-jlhf8     1/1     Running     0          10m
istio-pilot-5d77c559d4-4c5tg              2/2     Running     0          10m
istio-policy-86478df5d4-qd79g             2/2     Running     5          10m
istio-security-post-install-1.2.2-lpvmc   0/1     Completed   0          10m
istio-sidecar-injector-7b98dd6bcc-kc622   1/1     Running     0          10m
istio-telemetry-786747687f-h9vs6          2/2     Running     5          10m
istio-tracing-555cf644d-z8pt2             1/1     Running     0          10m
kiali-6cd6f9dfb5-5ncxr                    1/1     Running     0          10m
prometheus-7d7b9f7844-wfdcl               1/1     Running     0          10m
```

如果你的集群在一个没有外部负载均衡器支持的环境中运行（例如 Minikube），`istio-ingressgateway` 的 `EXTERNAL-IP` 会是 `<pending>`。要访问这个网关，只能通过服务的 `NodePort` 或者使用端口转发来进行访问。



### 4.2.3 Bookinfo demo体验

* #### 实验概述

部署一个样例应用，它由四个单独的微服务构成，用来演示多种 Istio 特性。这个应用模仿在线书店的一个分类，显示一本书的信息。页面上会显示一本书的描述，书籍的细节（ISBN、页数等），以及关于这本书的一些评论。

Bookinfo 应用分为四个单独的微服务：

- `productpage` ：`productpage` 微服务会调用 `details` 和 `reviews` 两个微服务，用来生成页面。
- `details` ：这个微服务包含了书籍的信息。
- `reviews` ：这个微服务包含了书籍相关的评论。它还会调用 `ratings` 微服务。
- `ratings` ：`ratings` 微服务中包含了由书籍评价组成的评级信息。

`reviews` 微服务有 3 个版本：

- v1 版本不会调用 `ratings` 服务。
- v2 版本会调用 `ratings` 服务，并使用 1 到 5 个黑色星形图标来显示评分信息。
- v3 版本会调用 `ratings` 服务，并使用 1 到 5 个红色星形图标来显示评分信息。

* #### 架构拓扑

![image-20190805161944923](/Users/xuel/Library/Application Support/typora-user-images/image-20190805161944923.png)



Bookinfo 是一个异构应用，几个微服务是由不同的语言编写的。这些服务对 Istio 并无依赖，但是构成了一个有代表性的服务网格的例子：它由多个服务、多个语言构成，并且 `reviews` 服务具有多个版本。

* #### 部署应用

要在 Istio 中运行这一应用，无需对应用自身做出任何改变。我们只要简单的在 Istio 环境中对服务进行配置和运行，具体一点说就是把 Envoy sidecar 注入到每个服务之中。这个过程所需的具体命令和配置方法由运行时环境决定，而部署结果较为一致，如下图所示：

![image-20190805162157638](/Users/xuel/Library/Application Support/typora-user-images/image-20190805162157638.png)



所有的微服务都和 Envoy sidecar 集成在一起，被集成服务所有的出入流量都被 sidecar 所劫持，这样就为外部控制准备了所需的 Hook，然后就可以利用 Istio 控制平面为应用提供服务路由、遥测数据收集以及策略实施等功能。



* #### 部署bookinfo

```shell
# 在使用 kubectl apply 进行应用部署的时候，如果目标命名空间已经打上了标签 istio-injection=enabled，Istio sidecar injector 会自动把 Envoy 容器注入到你的应用 Pod 之中。
kubectl label namespace default istio-injection=enabled
```

如果目标命名空间中没有打上 `istio-injection` 标签， 可以使用 [`istioctl kube-inject`](https://istio.io/zh/docs/reference/commands/istioctl/#istioctl-kube-inject) 命令，在部署之前手工把 Envoy 容器注入到应用 Pod 之中：`istioctl kube-inject -f <your-app-spec>.yaml | kubectl apply -f -`

#### 部署bookinfo：

```shell
[root@VM_0_2_centos istio-1.2.2]# kubectl apply -f <(istioctl kube-inject -f samples/bookinfo/platform/kube/bookinfo.yaml)
service/details created
serviceaccount/bookinfo-details created
deployment.apps/details-v1 created
service/ratings created
serviceaccount/bookinfo-ratings created
deployment.apps/ratings-v1 created
service/reviews created
serviceaccount/bookinfo-reviews created
deployment.apps/reviews-v1 created
deployment.apps/reviews-v2 created
deployment.apps/reviews-v3 created
service/productpage created
serviceaccount/bookinfo-productpage created
deployment.apps/productpage-v1 created
```

上面的命令会启动全部的四个服务，其中也包括了 `reviews` 服务的三个版本（`v1`、`v2` 以及 `v3`）

#### 验证bookinfo服务是否启动完成

```shell
# 查看pods和svc
[root@VM_0_2_centos istio-1.2.2]# kubectl get pods
NAME                             READY   STATUS    RESTARTS   AGE
details-v1-677976f7dc-m7mgw      2/2     Running   0          4m32s
productpage-v1-f76db69c5-vqkjv   2/2     Running   0          4m32s
ratings-v1-7f66cd4c87-8b4q6      2/2     Running   0          4m33s
reviews-v1-77bcd5d4f5-nggcv      2/2     Running   0          4m33s
reviews-v2-7cdb7475fb-hz8km      2/2     Running   0          4m33s
reviews-v3-8496dbbbbf-rzwb6      2/2     Running   0          4m33s
[root@VM_0_2_centos istio-1.2.2]# kubectl get svc
NAME          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
details       ClusterIP   10.102.51.14     <none>        9080/TCP   4m46s
kubernetes    ClusterIP   10.96.0.1        <none>        443/TCP    57m
productpage   ClusterIP   10.102.140.244   <none>        9080/TCP   4m46s
ratings       ClusterIP   10.96.187.95     <none>        9080/TCP   4m46s
reviews       ClusterIP   10.98.206.234    <none>        9080/TCP   4m46s


# 检查一下pod，看istio的sidecar自动注入是否成功
kubectl describe pods details-v1-677976f7dc-m7mgw

# 要确认 Bookinfo 应用程序正在运行，请通过某个 pod 中的 curl 命令向其发送请求，例如来自 ratings：
[root@VM_0_2_centos istio-1.2.2]# kubectl exec -it $(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') -c ratings -- curl productpage:9080/productpage | grep -o "<title>.*</title>"
<title>Simple Bookstore App</title>
```

#### 确定 Ingress 的 IP 和端口

现在 Bookinfo 服务启动并运行中，你需要使应用程序可以从外部访问 Kubernetes 集群，例如使用浏览器。一个 [Istio Gateway](https://istio.io/zh/docs/concepts/traffic-management/#gateway)应用到了目标中。

* 为应用程序定义入口网关：

```shell
[root@VM_0_2_centos istio-1.2.2]# kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
gateway.networking.istio.io/bookinfo-gateway created
virtualservice.networking.istio.io/bookinfo created
```

* 确认网关创建完成：

```shell
[root@VM_0_2_centos istio-1.2.2]# kubectl get gateway
NAME               AGE
bookinfo-gateway   28s
[root@VM_0_2_centos istio-1.2.2]# kubectl get virtualservice
NAME       GATEWAYS             HOSTS   AGE
bookinfo   [bookinfo-gateway]   [*]     37s
```

* 根据[文档](https://istio.io/zh/docs/tasks/traffic-management/ingress/#使用外部负载均衡器时确定-ip-和端口)设置访问网关的 `INGRESS_HOST` 和 `INGRESS_PORT` 变量。确认并设置。

启动 [httpbin](https://github.com/istio/istio/tree/release-1.2/samples/httpbin) 样本，该样本将用作要在外部公开的目标服务。

```shell
[root@VM_0_2_centos istio-1.2.2]# kubectl apply -f <(istioctl kube-inject -f samples/httpbin/httpbin.yaml)
service/httpbin created
deployment.extensions/httpbin created
```

确定入口 IP 和端口

```shell
[root@VM_0_2_centos istio-1.2.2]# kubectl get svc istio-ingressgateway -n istio-system
NAME                   TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)                                                                                                                                      AGE
istio-ingressgateway   NodePort   10.108.162.3   <none>        15020:31931/TCP,80:31380/TCP,443:31390/TCP,31400:31400/TCP,15029:32205/TCP,15030:31540/TCP,15031:30722/TCP,15032:32366/TCP,15443:31319/TCP   36m
```

如果 `EXTERNAL-IP` 有值（IP 地址或主机名），则说明您的环境具有可用于 Ingress 网关的外部负载均衡器。如果 `EXTERNAL-IP` 值是 `<none>`（或一直是 `<pending>` ），则说明可能您的环境并没有为 Ingress 网关提供外部负载均衡器的功能。在这种情况下，您可以使用 Service 的 [node port](https://kubernetes.io/docs/concepts/services-networking/service/#nodeport) 方式访问网关。

#### 确定使用 Node Port 时的 ingress IP 和端口

```shell
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')
export INGRESS_HOST=$(minikube ip)
```

* 设置 `GATEWAY_URL`：

```shell
[root@VM_0_2_centos istio-1.2.2]# export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
[root@VM_0_2_centos istio-1.2.2]# echo $GATEWAY_URL
172.16.0.2:31380

# 确认应用
[root@VM_0_2_centos istio-1.2.2]# curl -s http://${GATEWAY_URL}/productpage | grep -o "<title>.*</title>"
<title>Simple Bookstore App</title>
```

#### istio 功能测试(流量管理)

由于demo环境为腾讯云CVM，使用公网IP进行访问

##### ![image-20190805172532752](/Users/xuel/Library/Application Support/typora-user-images/image-20190805172532752.png)![image-20190805172543950](/Users/xuel/Library/Application Support/typora-user-images/image-20190805172543950.png)![image-20190805172553721](/Users/xuel/Library/Application Support/typora-user-images/image-20190805172553721.png)请求路由

###### 应用缺省目标规则

在使用 Istio 控制 Bookinfo 版本路由之前，你需要在目标规则中定义好可用的版本，命名为 *subsets* 。

运行以下命令为 Bookinfo 服务创建的默认的目标规则：

如果不需要启用双向TLS，请执行以下命令：

```shell
[root@VM_0_2_centos istio-1.2.2]# kubectl apply -f samples/bookinfo/networking/destination-rule-all.yaml
destinationrule.networking.istio.io/productpage created
destinationrule.networking.istio.io/reviews created
destinationrule.networking.istio.io/ratings created
destinationrule.networking.istio.io/details created

# 查看目标规则
[root@VM_0_2_centos istio-1.2.2]# kubectl get destinationrules -o yaml
```

##### 

Istio [Bookinfo](https://istio.io/zh/docs/examples/bookinfo/) 示例包含四个独立的微服务，每个微服务都有多个版本。 其中一个微服务 `reviews` 的三个不同版本已经部署并同时运行。 为了说明这导致的问题，在浏览器中访问 Bookinfo 应用程序的 `/productpage` 并刷新几次。 您会注意到，有时书评的输出包含星级评分，有时则不包含。 这是因为没有明确的默认服务版本路由，Istio 将以循环方式请求路由到所有可用版本。

此任务的最初目标是应用将所有流量路由到微服务的 `v1` 版本的规则。稍后， 您将应用规则根据 HTTP 请求 header 的值路由流量。

应用 virtual service

```shell
[root@VM_0_2_centos istio-1.2.2]# kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml
virtualservice.networking.istio.io/productpage created
virtualservice.networking.istio.io/reviews created
virtualservice.networking.istio.io/ratings created
virtualservice.networking.istio.io/details created

[root@VM_0_2_centos istio-1.2.2]# kubectl get virtualservices
NAME          GATEWAYS             HOSTS           AGE
bookinfo      [bookinfo-gateway]   [*]             54m
details                            [details]       44s
productpage                        [productpage]   44s
ratings                            [ratings]       44s
reviews                            [reviews]       44s

[root@VM_0_2_centos istio-1.2.2]# kubectl get destinationrules
NAME          HOST          AGE
details       details       21m
productpage   productpage   21m
ratings       ratings       21m
reviews       reviews       21m
```

请注意，无论您刷新多少次，页面的评论部分都不会显示评级星标。这是因为您将 Istio 配置为将评论服务的所有流量路由到版本 `reviews:v1`，并且此版本的服务不访问星级评分服务。

![image-20190805175037187](/Users/xuel/Library/Application Support/typora-user-images/image-20190805175037187.png)

测试将流量路由到下一个服务

###### 基于用户身份的路由

接下来，您将更改路由配置，以便将来自特定用户的所有流量路由到特定服务版本。在这种情况下，来自名为 Jason 的用户的所有流量将被路由到服务 `reviews:v2`。

请注意，Istio 对用户身份没有任何特殊的内置机制。这个例子的基础在于， `productpage` 服务在所有针对 `reviews` 服务的调用请求中 都加自定义的 HTTP header，从而达到在流量中对最终用户身份识别的这一效果。

```shell
#运行以下命令以启用基于用户的路由
[root@VM_0_2_centos istio-1.2.2]# kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml
virtualservice.networking.istio.io/reviews configured
[root@VM_0_2_centos istio-1.2.2]# kubectl get virtualservice reviews -o yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"networking.istio.io/v1alpha3","kind":"VirtualService","metadata":{"annotations":{},"name":"reviews","namespace":"default"},"spec":{"hosts":["reviews"],"http":[{"match":[{"headers":{"end-user":{"exact":"jason"}}}],"route":[{"destination":{"host":"reviews","subset":"v2"}}]},{"route":[{"destination":{"host":"reviews","subset":"v1"}}]}]}}
  creationTimestamp: "2019-08-05T09:48:04Z"
  generation: 2
  name: reviews
  namespace: default
  resourceVersion: "10861"
  selfLink: /apis/networking.istio.io/v1alpha3/namespaces/default/virtualservices/reviews
  uid: f7c51bdf-0023-4520-ba1b-4d38ad65212e
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
        subset: v1
```

在 Bookinfo 应用程序的 `/productpage` 上，以用户 `jason` 身份登录。

![image-20190805175449249](/Users/xuel/Library/Application Support/typora-user-images/image-20190805175449249.png)

以其他用户身份登录（选择您想要的任何名称），刷新浏览器。现在星星消失了。这是因为除了 Jason 之外，所有用户的流量都被路由到 `reviews:v1`。

- `productpage` → `reviews:v2` → `ratings` (`jason` 用户)
- `productpage` → `reviews:v1` (其他用户)

###### 原理

在此任务中，您首先使用 Istio 将 100% 的请求流量都路由到了 Bookinfo 服务的 v1 版本。 然后再设置了一条路由规则，在 `productpage` 服务中添加了路由规则，这一规则根据请求的 `end-user` 自定义 header 内容，选择性地将特定的流量路由到了 `reviews` 服务的 `v2` 版本。

请注意，为了利用 Istio 的 L7 路由功能，Kubernetes 中的服务（如本任务中使用的 Bookinfo 服务）必须遵守某些特定限制。 参考 [sidecar 注入文档](https://istio.io/zh/docs/setup/kubernetes/additional-setup/requirements)了解详情。

在[流量转移](https://istio.io/zh/docs/tasks/traffic-management/traffic-shifting)任务中，您将按照在此处学习的相同基本模式来配置路由规则，以逐步将流量从服务的一个版本发送到另一个版本。

###### 清理

如果不操作后续实验，可以进行清理

```shell
kubectl delete -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```

##### 故障注入

###### 使用 HTTP 延迟进行故障注入

为了测试微服务应用程序 Bookinfo 的弹性，我们将为用户 `jason` 在 `reviews:v2` 和 `ratings` 服务之间注入一个 7 秒的延迟。 这个测试将会发现故意引入 Bookinfo 应用程序中的错误。

由于 `reviews:v2` 服务对其 ratings 服务的调用具有 10 秒的硬编码连接超时，比我们设置的 7s 延迟要大，因此我们期望端到端流程是正常的（没有任何错误）。

```shell
[root@VM_0_2_centos istio-1.2.2]# kubectl apply -f samples/bookinfo/networking/virtual-service-ratings-test-delay.yaml
virtualservice.networking.istio.io/ratings configured
[root@VM_0_2_centos istio-1.2.2]# kubectl get virtualservices
NAME          GATEWAYS             HOSTS           AGE
bookinfo      [bookinfo-gateway]   [*]             70m
details                            [details]       17m
productpage                        [productpage]   17m
ratings                            [ratings]       17m
reviews                            [reviews]       17m
[root@VM_0_2_centos istio-1.2.2]# kubectl get virtualservices ratings -o yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"networking.istio.io/v1alpha3","kind":"VirtualService","metadata":{"annotations":{},"name":"ratings","namespace":"default"},"spec":{"hosts":["ratings"],"http":[{"fault":{"delay":{"fixedDelay":"7s","percentage":{"value":100}}},"match":[{"headers":{"end-user":{"exact":"jason"}}}],"route":[{"destination":{"host":"ratings","subset":"v1"}}]},{"route":[{"destination":{"host":"ratings","subset":"v1"}}]}]}}
  creationTimestamp: "2019-08-05T09:48:04Z"
  generation: 2
  name: ratings
  namespace: default
  resourceVersion: "11757"
  selfLink: /apis/networking.istio.io/v1alpha3/namespaces/default/virtualservices/ratings
  uid: 212f39f7-9797-4ec0-99d9-f4ed7df7beb6
spec:
  hosts:
  - ratings
  http:
  - fault:
      delay:
        fixedDelay: 7s
        percentage:
          value: 100
    match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: ratings
        subset: v1
  - route:
    - destination:
        host: ratings
        subset: v1
```

使用用户 `jason` 登陆到 `/productpage` 界面。你期望 Bookinfo 主页在大约 7 秒钟加载完成并且没有错误。但是，出现了一个问题，Reviews 部分显示了错误消息：

![image-20190805180735189](/Users/xuel/Library/Application Support/typora-user-images/image-20190805180735189.png)

###### 原理

你发现了一个 bug。在微服务中有硬编码超时，导致 `reviews` 服务失败。

在 `productpage` 和 `reviews` 服务之间超时时间是 6s - 编码 3s + 1 次重试总共 6s ，`reviews` 和 `ratings` 服务之间的硬编码连接超时为 10s 。由于我们引入的延时，`/productpage` 提前超时并引发错误。

这些类型的错误可能发生在典型的企业应用程序中，其中不同的团队独立地开发不同的微服务。Istio 的故障注入规则可帮助您识别此类异常，而不会影响最终用户。

###### 错误修复

你通常会解决这样的问题：

1. 要么增加 `/productpage` 的超时或者减少 `reviews` 到 `ratings` 服务的超时
2. 终止并重启微服务
3. 确认 `productpage` 正常响应并且没有任何错误。

但是，我们已经在 `reviews` 服务的 v3 版中运行此修复程序，因此我们可以通过将所有流量迁移到 `reviews:v3` 来解决问题， 如[流量转移](https://istio.io/zh/docs/tasks/traffic-management/traffic-shifting/)中所述任务。

###### 使用 HTTP abort 进行故障注入

测试微服务弹性的另一种方法是引入 HTTP abort 故障。在这个任务中，在 `ratings` 微服务中引入 HTTP abort ，测试用户为 `jason` 。

在这个案例中，我们希望页面能够立即加载，同时显示 `Ratings service is currently unavailable` 这样的消息。

```shell
[root@VM_0_2_centos istio-1.2.2]# kubectl apply -f samples/bookinfo/networking/virtual-service-ratings-test-abort.yaml
virtualservice.networking.istio.io/ratings configured
[root@VM_0_2_centos istio-1.2.2]# kubectl get virtualservice ratings -o yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"networking.istio.io/v1alpha3","kind":"VirtualService","metadata":{"annotations":{},"name":"ratings","namespace":"default"},"spec":{"hosts":["ratings"],"http":[{"fault":{"abort":{"httpStatus":500,"percentage":{"value":100}}},"match":[{"headers":{"end-user":{"exact":"jason"}}}],"route":[{"destination":{"host":"ratings","subset":"v1"}}]},{"route":[{"destination":{"host":"ratings","subset":"v1"}}]}]}}
  creationTimestamp: "2019-08-05T09:48:04Z"
  generation: 3
  name: ratings
  namespace: default
  resourceVersion: "12538"
  selfLink: /apis/networking.istio.io/v1alpha3/namespaces/default/virtualservices/ratings
  uid: 212f39f7-9797-4ec0-99d9-f4ed7df7beb6
spec:
  hosts:
  - ratings
  http:
  - fault:
      abort:
        httpStatus: 500
        percentage:
          value: 100
    match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: ratings
        subset: v1
  - route:
    - destination:
        host: ratings
        subset: v1
```

使用用户 `jason` 登陆到 `/productpage` 界面。

![image-20190805181756521](/Users/xuel/Library/Application Support/typora-user-images/image-20190805181756521.png)

###### 清理

```shell
kubectl delete -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```



##### 流量迁移

一个常见的用例是将流量从一个版本的微服务逐渐迁移到另一个版本。在 Istio 中，您可以通过配置一系列规则来实现此目标，这些规则将一定百分比的流量路由到一个或另一个服务。在此任务中，您将 50％ 的流量发送到 `reviews:v1`，另外 50％ 的流量发送到 `reviews:v3`。然后将 100％ 的流量发送到 `reviews:v3` 来完成迁移。

###### 应用基于权重的路由

1. 将所有流量路由到 `v1` 版本的各个微服务。

```shell
[root@VM_0_2_centos istio-1.2.2]# kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml
virtualservice.networking.istio.io/productpage unchanged
virtualservice.networking.istio.io/reviews configured
virtualservice.networking.istio.io/ratings configured
virtualservice.networking.istio.io/details unchanged
```

请注意，不管刷新多少次，页面的评论部分都不会显示评级星号。这是因为 Istio 被配置为将 reviews 服务的的所有流量都路由到了 `reviews:v1` 版本， 而该版本的服务不会访问带星级的 ratings 服务。

2. 把 50% 的流量从 `reviews:v1` 转移到 `reviews:v3`：

```shell
[root@VM_0_2_centos istio-1.2.2]# kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-50-v3.yaml
virtualservice.networking.istio.io/reviews configured
[root@VM_0_2_centos istio-1.2.2]# kubectl get virtualservice reviews -o yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"networking.istio.io/v1alpha3","kind":"VirtualService","metadata":{"annotations":{},"name":"reviews","namespace":"default"},"spec":{"hosts":["reviews"],"http":[{"route":[{"destination":{"host":"reviews","subset":"v1"},"weight":50},{"destination":{"host":"reviews","subset":"v3"},"weight":50}]}]}}
  creationTimestamp: "2019-08-05T09:48:04Z"
  generation: 4
  name: reviews
  namespace: default
  resourceVersion: "13220"
  selfLink: /apis/networking.istio.io/v1alpha3/namespaces/default/virtualservices/reviews
  uid: f7c51bdf-0023-4520-ba1b-4d38ad65212e
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 50
    - destination:
        host: reviews
        subset: v3
      weight: 50
```

刷新浏览器中的 `/productpage` 页面，大约有 50% 的几率会看到页面中出带红色星级的评价内容。这是因为 `v3` 版本的 `reviews`访问了带星级评级的 `ratings` 服务，但 `v1` 版本却没有。

3. 如果您认为 `reviews:v3` 微服务已经稳定，你可以通过应用此 virtual service 将 100% 的流量路由到 `reviews:v3`

```shell
[root@VM_0_2_centos istio-1.2.2]# kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-v3.yaml
virtualservice.networking.istio.io/reviews configured
[root@VM_0_2_centos istio-1.2.2]# kubectl get virtualservice reviews -o yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"networking.istio.io/v1alpha3","kind":"VirtualService","metadata":{"annotations":{},"name":"reviews","namespace":"default"},"spec":{"hosts":["reviews"],"http":[{"route":[{"destination":{"host":"reviews","subset":"v3"}}]}]}}
  creationTimestamp: "2019-08-05T09:48:04Z"
  generation: 5
  name: reviews
  namespace: default
  resourceVersion: "13348"
  selfLink: /apis/networking.istio.io/v1alpha3/namespaces/default/virtualservices/reviews
  uid: f7c51bdf-0023-4520-ba1b-4d38ad65212e
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v3
```

刷新 `/productpage` 时，您将始终看到带有红色星级评分的书评。

在这项任务中，我们使用 Istio 的加权路由功能将流量从旧版本的 `reviews` 服务迁移到新版本。请注意，这和使用容器编排平台的部署功能来进行版本迁移完全不同，后者使用了实例扩容来对流量进行管理。

使用 Istio，两个版本的 `reviews` 服务可以独立地进行扩容和缩容，并不会影响这两个版本服务之间的流量分发。

如果想了解支持自动伸缩的版本路由的更多信息，请查看[使用 Istio 的 Canary Deployments](https://istio.io/blog/2017/0.1-canary/) 。

###### 清理

```shell
kubectl delete -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```



##### TCP流量转移

###### 应用基于权重的 TCP 路由

* 第一个步骤是部署 `tcp-echo` 微服务的 `v1` 版本。

```shell
[root@VM_0_2_centos istio-1.2.2]# kubectl apply -f <(istioctl kube-inject -f samples/tcp-echo/tcp-echo-services.yaml)
service/tcp-echo created

[root@VM_0_2_centos istio-1.2.2]# kubectl get all |grep tcp-echo
pod/tcp-echo-v1-55d4546c49-tnvb2     2/2     Running   0          32s
pod/tcp-echo-v2-c8566b665-4jp47      2/2     Running   0          32s
service/tcp-echo      ClusterIP   10.111.220.32    <none>        9000/TCP   34s
deployment.apps/tcp-echo-v1      1/1     1            1           34s
deployment.apps/tcp-echo-v2      1/1     1            1           33s
replicaset.apps/tcp-echo-v1-55d4546c49     1         1         1       33s
replicaset.apps/tcp-echo-v2-c8566b665      1         1         1       33s
```

* 下一步，把所有目标是 `tcp-echo` 微服务的 TCP 流量路由到 `v1` 版本

```shell
[root@VM_0_2_centos istio-1.2.2]# kubectl apply -f samples/tcp-echo/tcp-echo-all-v1.yaml
gateway.networking.istio.io/tcp-echo-gateway created
destinationrule.networking.istio.io/tcp-echo-destination created
virtualservice.networking.istio.io/tcp-echo created
```

* 确认 `tcp-echo` 服务已经启动并开始运行

```shell
[root@VM_0_2_centos istio-1.2.2]# export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="tcp")].port}')
[root@VM_0_2_centos istio-1.2.2]# echo $INGRESS_PORT
31400
[root@VM_0_2_centos istio-1.2.2]# export INGRESS_HOST=$(minikube ip)
[root@VM_0_2_centos istio-1.2.2]# echo $INGRESS_HOST
172.16.0.2

# 向 tcp-echo 微服务发送一些 TCP 流量：
[root@VM_0_2_centos istio-1.2.2]# for i in {1..10}; do \
> docker run -e INGRESS_HOST=$INGRESS_HOST -e INGRESS_PORT=$INGRESS_PORT -it --rm busybox sh -c "(date; sleep 1) | nc $INGRESS_HOST $INGRESS_PORT"; \
> done
Unable to find image 'busybox:latest' locally
latest: Pulling from library/busybox
ee153a04d683: Pull complete
Digest: sha256:9f1003c480699be56815db0f8146ad2e22efea85129b5b5983d0e0fb52d9ab70
Status: Downloaded newer image for busybox:latest
one Tue Aug  6 08:40:31 UTC 2019
one Tue Aug  6 08:40:35 UTC 2019
one Tue Aug  6 08:40:38 UTC 2019
one Tue Aug  6 08:40:42 UTC 2019
```

不难发现，所有的时间戳都有一个 `one` 前缀，这代表所有访问 `tcp-echo` 服务的流量都被路由到了 `v1` 版本。

* 用下面的命令把 20% 的流量从 `tcp-echo:v1` 转移到 `tcp-echo:v2`：

```shell

[root@VM_0_2_centos istio-1.2.2]# kubectl apply -f samples/tcp-echo/tcp-echo-20-v2.yaml
virtualservice.networking.istio.io/tcp-echo configured

[root@VM_0_2_centos istio-1.2.2]# kubectl get virtualservice tcp-echo -o yaml
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
```

* 测试向 `tcp-echo` 微服务发送更多 TCP 流量：

```shell
[root@VM_0_2_centos istio-1.2.2]# for i in {1..10}; do docker run -e INGRESS_HOST=$INGRESS_HOST -e INGRESS_PORT=$INGRESS_PORT -it --rm busybox sh -c "(date; sleep 1) | nc $INGRESS_HOST $INGRESS_PORT"; done
one Tue Aug  6 08:43:23 UTC 2019
two Tue Aug  6 08:43:27 UTC 2019
one Tue Aug  6 08:43:29 UTC 2019
one Tue Aug  6 08:43:31 UTC 2019
one Tue Aug  6 08:43:33 UTC 2019
one Tue Aug  6 08:43:35 UTC 2019
one Tue Aug  6 08:43:37 UTC 2019
one Tue Aug  6 08:43:39 UTC 2019
two Tue Aug  6 08:43:41 UTC 2019
two Tue Aug  6 08:43:43 UTC 2019
```

现在应该会看到，输出内容中有 20% 的时间戳前缀为 `two`，这意味着 80% 的流量被路由到 `tcp-echo:v1`，其余 20% 流量被路由到了 `v2`。

这个任务里，用 Istio 的权重路由功能，把一部分访问 `tcp-echo` 服务的 TCP 流量被从旧版本迁移到了新版本。容器编排平台中的版本迁移使用的是对特定组别的实例进行伸缩来完成对流量的控制的，两种迁移方式显然大相径庭。

在 Istio 中可以对两个版本的 `tcp-echo` 服务进行独立的扩缩容，伸缩过程中不会对流量的分配结果造成影响，可以阅读博客：[使用 Istio 进行金丝雀部署](https://istio.io/zh/blog/2017/0.1-canary/)，进一步了解相关内容。

###### 清理

```shell
kubectl delete -f samples/tcp-echo/tcp-echo-all-v1.yaml
kubectl delete -f samples/tcp-echo/tcp-echo-services.yaml
```

##### 设置超时时间

初始化bookinfo

```shell
kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```

###### 请求超时

可以在[路由规则](https://istio.io/zh/docs/reference/config/istio.networking.v1alpha3/#httproute)的 `timeout` 字段中来给 http 请求设置请求超时。缺省情况下，超时被设置为 15 秒钟，本文任务中，会把 `reviews`服务的超时设置为一秒钟。为了能观察设置的效果，还需要在对 `ratings` 服务的调用中加入两秒钟的延迟。

* 到 `reviews:v2` 服务的路由定义：

```shell
$ kubectl apply -f - <<EOF
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
        subset: v2
EOF
```

* 在对 `ratings` 服务的调用中加入两秒钟的延迟：

```shell
kubectl apply -f - <<EOF
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
        percent: 100
        fixedDelay: 2s
    route:
    - destination:
        host: ratings
        subset: v1
EOF
```

* http://106.52.172.132:31380/productpage,这时应该能看到 Bookinfo 应用在正常运行（显示了评级的星形符号），但是每次刷新页面，都会出现两秒钟的延迟。

* 接下来在目的为 `reviews:v2` 服务的请求加入一秒钟的请求超时：

```shell
kubectl apply -f - <<EOF
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
        subset: v2
    timeout: 0.5s
EOF
```

* 刷新 Bookinfo 的 Web 页面。

  这时候应该看到一秒钟就会返回，而不是之前的两秒钟，但 `reviews` 的显示已经不见了。

![image-20190806170132549](/Users/xuel/Library/Application Support/typora-user-images/image-20190806170132549.png)

###### 原理

上面的任务中，使用 Istio 为调用 `reviews` 微服务的请求中加入了一秒钟的超时控制，覆盖了缺省的 15 秒钟设置。页面刷新时，`reviews` 服务后面会调用 `ratings` 服务，使用 Istio 在对 `ratings` 的调用中注入了两秒钟的延迟，这样就让 `reviews` 服务要花费超过一秒钟的时间来调用 `ratings` 服务，从而触发了我们加入的超时控制。

这样就会看到 Bookinfo 的页面（ 页面由 `reviews` 服务生成）上没有出现 `reviews` 服务的显示内容，取而代之的是错误信息：**Sorry, product reviews are currently unavailable for this book** ，出现这一信息的原因就是因为来自 `reviews` 服务的超时错误。

如果测试了[故障注入任务](https://istio.io/zh/docs/tasks/traffic-management/fault-injection/)，会发现 `productpage` 微服务在调用 `reviews` 微服务时，还有自己的应用级超时设置（三秒钟）。注意这里我们用路由规则设置了一秒钟的超时。如果把超时设置为超过三秒钟（例如四秒钟）会毫无效果，这是因为内部的服务中设置了更为严格的超时要求。更多细节可以参见[故障处理 FAQ](https://istio.io/zh/docs/concepts/traffic-management/#faq) 的相关内容。

还有一点关于 Istio 中超时控制方面的补充说明，除了像本文一样在路由规则中进行超时设置之外，还可以进行请求一级的设置，只需在应用的外发流量中加入 `x-envoy-upstream-rq-timeout-ms` Header 即可。在这个 Header 中的超时设置单位是毫秒而不是秒。

清理

```shell
kubectl delete -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```

##### 控制 Ingress 流量

在 Kubernetes 环境中，[Kubernetes Ingress 资源](https://kubernetes.io/docs/concepts/services-networking/ingress/) 用于指定应在集群外部公开的服务。在 Istio 服务网格中，更好的方法（也适用于 Kubernetes 和其他环境）是使用不同的配置模型，即 [Istio `Gateway`](https://istio.io/zh/docs/reference/config/istio.networking.v1alpha3/#gateway) 。 `Gateway` 允许将 Istio 功能（例如，监控和路由规则）应用于进入集群的流量。

此任务描述如何配置 Istio 以使用 Istio 在服务网格外部公开服务 `Gateway`。

###### 暴露httpbin服务

* 部署httpbin服务

```shell
kubectl apply -f <(istioctl kube-inject -f samples/httpbin/httpbin.yaml)
```

* 确定IP和端口

```shell
[root@VM_0_2_centos istio-1.2.2]# export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
[root@VM_0_2_centos istio-1.2.2]# echo $INGRESS_PORT
31380
[root@VM_0_2_centos istio-1.2.2]# export INGRESS_HOST=$(minikube ip)
[root@VM_0_2_centos istio-1.2.2]# echo $INGRESS_HOST
172.16.0.2
```

* 使用 Istio 网关配置 Ingress

Ingress [`Gateway`](https://istio.io/zh/docs/reference/config/istio.networking.v1alpha3/#gateway) 描述了在网格边缘操作的负载平衡器，用于接收传入的 HTTP/TCP 连接。它配置暴露的端口、协议等，但与 [Kubernetes Ingress Resources](https://kubernetes.io/docs/concepts/services-networking/ingress/) 不同，它不包括任何流量路由配置。流入流量的流量路由使用 Istio 路由规则进行配置，与内部服务请求完全相同。

让我们看看如何为 `Gateway` 在 HTTP 80 端口上配置流量。

```shell
# 创建一个 Istio Gateway：
[root@VM_0_2_centos istio-1.2.2]# kubectl apply -f - <<EOF
> apiVersion: networking.istio.io/v1alpha3
> kind: Gateway
> metadata:
>   name: httpbin-gateway
> spec:
>   selector:
>     istio: ingressgateway # use Istio default gateway implementation
>   servers:
>   - port:
>       number: 80
>       name: http
>       protocol: HTTP
>     hosts:
>     - "httpbin.example.com"
> EOF
gateway.networking.istio.io/httpbin-gateway created
[root@VM_0_2_centos istio-1.2.2]# kubectl get gw httpbin-gateway
NAME              AGE
httpbin-gateway   12s


# 为通过 Gateway 进入的流量配置路由：
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
spec:
  hosts:
  - "httpbin.example.com"
  gateways:
  - httpbin-gateway
  http:
  - match:
    - uri:
        prefix: /status
    - uri:
        prefix: /delay
    route:
    - destination:
        port:
          number: 8000
        host: httpbin
EOF

[root@VM_0_2_centos istio-1.2.2]# kubectl get vs httpbin
NAME      GATEWAYS            HOSTS                   AGE
httpbin   [httpbin-gateway]   [httpbin.example.com]   7s
```

在这里，我们为服务创建了一个[虚拟服务](https://istio.io/zh/docs/reference/config/istio.networking.v1alpha3/#virtualservice)配置 `httpbin` ，其中包含两条路由规则，允许路径 `/status` 和 路径的流量 `/delay`。

该[网关](https://istio.io/zh/docs/reference/config/istio.networking.v1alpha3/#virtualservice)列表指定，只有通过我们的要求 `httpbin-gateway` 是允许的。所有其他外部请求将被拒绝，并返回 404 响应。

请注意，在此配置中，来自网格中其他服务的内部请求不受这些规则约束，而是简单地默认为循环路由。要将这些（或其他规则）应用于内部调用，我们可以将特殊值 `mesh` 添加到 `gateways` 的列表中。

* 使用 curl 访问 httpbin 服务：

```shell
[root@VM_0_2_centos istio-1.2.2]# curl -I -HHost:httpbin.example.com http://$INGRESS_HOST:$INGRESS_PORT/status/200
HTTP/1.1 200 OK
server: istio-envoy
date: Tue, 06 Aug 2019 09:26:53 GMT
content-type: text/html; charset=utf-8
access-control-allow-origin: *
access-control-allow-credentials: true
content-length: 0
x-envoy-upstream-service-time: 268
```

请注意，这里使用该 `-H` 标志将 `Host` HTTP Header 设置为 “httpbin.example.com”。这一操作是必需的，因为上面的 Ingress `Gateway` 被配置为处理 “httpbin.example.com”，但在测试环境中没有该主机的 DNS 绑定，只是将请求发送到 Ingress IP。

```shell
#访问任何未明确公开的其他 URL，应该会看到一个 HTTP 404 错误：
[root@VM_0_2_centos istio-1.2.2]# curl -I -HHost:httpbin.example.com http://$INGRESS_HOST:$INGRESS_PORT/headers
HTTP/1.1 404 Not Found
date: Tue, 06 Aug 2019 09:27:46 GMT
server: istio-envoy
transfer-encoding: chunked
```

* 使用浏览器访问 Ingress 服务

在浏览器中输入 `httpbin` 服务的地址是不会生效的，这是因为因为我们没有办法让浏览器像 `curl` 一样装作访问 `httpbin.example.com`。而在现实世界中，因为有正常配置的主机和 DNS 记录，这种做法就能够成功了——只要简单的在浏览器中访问由域名构成的 URL 即可，例如 `https://httpbin.example.com/status/200`。

要解决此问题以进行简单的测试和演示，我们可以在 `Gateway` 和 `VirtualService` 配置中为主机使用通配符值 `*`。例如，如果我们将 Ingress 配置更改为以下内容：

```shell
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: httpbin-gateway
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
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
  name: httpbin
spec:
  hosts:
  - "*"
  gateways:
  - httpbin-gateway
  http:
  - match:
    - uri:
        prefix: /headers
    route:
    - destination:
        port:
          number: 8000
        host: httpbin
EOF
```

接下来就可以在浏览器的 URL 中使用,http://106.52.172.132:31380/headers

![image-20190806173530930](/Users/xuel/Library/Application Support/typora-user-images/image-20190806173530930.png)



###### 原理

`Gateway` 配置资源允许外部流量进入 Istio 服务网，并使 Istio 的流量管理和策略功能可用于边缘服务。

在前面的步骤中，我们在 Istio 服务网格中创建了一个服务，并展示了如何将服务的 HTTP 端点暴露给外部流量。

###### 清理

```shell
kubectl delete gateway httpbin-gateway
kubectl delete virtualservice httpbin
kubectl delete --ignore-not-found=true -f samples/httpbin/httpbin.yaml
```

##### 加密ingress gateway

###### 使用 SDS 为 Gateway 提供 HTTPS 加密支持

[控制 Ingress 流量任务](https://istio.io/zh/docs/tasks/traffic-management/ingress)中描述了如何进行配置，通过 Ingress Gateway 把服务的 HTTP 端点暴露给外部。这里更进一步，使用单向或者双向 TLS 来完成开放服务的任务。双向 TLS 所需的私钥、服务器证书以及根证书都由 Secret 发现服务（SDS）完成配置。

* 验证curl工具是否支持



* 生成证书

```shell
yum -y install git
git clone https://github.com/nicholasjackson/mtls-go-example
# 进入代码库
pushd mtls-go-example
# 为 httpbin.example.com 生成证书。注意要把下面命令中的 password 替换为其它值。
./generate.sh httpbin.example.com password

# 看到提示后，所有问题都输入 Y 即可。这个命令会生成四个目录：1_root、2_intermediate、3_application 以及 4_client。这些目录中包含了后续过程所需的客户端和服务端证书。
mkdir ../httpbin.example.com && mv 1_root 2_intermediate 3_application 4_client ../httpbin.example.com
```

* 使用 SDS 配置 TLS Ingress 网关

可以配置 TLS Ingress 网关，让它从 Ingress 网关代理通过 SDS 获取凭据。Ingress 网关代理和 Ingress 网关在同一个 Pod 中运行，监视 Ingress 网关所在命名空间中新建的 `Secret`。在 Ingress 网关中启用 SDS 具有如下好处：

Ingress 网关无需重启，就可以动态的新增、删除或者更新密钥/证书对以及根证书。

无需加载 `Secret` 卷。创建了 `kubernetes` `Secret` 之后，这个 `Secret` 就会被网关代理捕获，并以密钥/证书对和根证书的形式发送给 Ingress 网关。

网关代理能够监视多个密钥/证书对。只需要为每个主机名创建 `Secret` 并更新网关定义就可以了。

1.在 Ingress 网关上启用 SDS，并部署 Ingress 网关代理。

这个功能缺省是禁用的，因此需要在 Helm 中打开 [`istio-ingressgateway.sds.enabled` 开关](https://github.com/istio/istio/blob/release-1.2/install/kubernetes/helm/istio/charts/gateways/values.yaml)，然后生成 `istio-ingressgateway.yaml` 文件：

```shell
wget https://get.helm.sh/helm-v2.14.2-linux-amd64.tar.gz
tar -xf helm-v2.14.2-linux-amd64.tar.gz
mv linux-amd64/helm /usr/bin/
helm version

[root@VM_0_2_centos istio-1.2.2]# helm template install/kubernetes/helm/istio/ --name istio \
> --namespace istio-system -x charts/gateways/templates/deployment.yaml \
> --set gateways.istio-egressgateway.enabled=false \
> --set gateways.istio-ingressgateway.sds.enabled=true > \
> $HOME/istio-ingressgateway.yaml
[root@VM_0_2_centos istio-1.2.2]# kubectl apply -f $HOME/istio-ingressgateway.yaml
deployment.apps/istio-ingressgateway configured
```

2.设置两个环境变量：`INGRESS_HOST` 和 `SECURE_INGRESS_PORT`：

```shell
export SECURE_INGRESS_PORT=$(kubectl -n istio-system \
get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].port}')

export INGRESS_HOST=$(minikube ip)
```

###### 1.为单一主机配置 TLS Ingress 网关

1.启动 `httpbin` 样例：

```shell
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: httpbin
  labels:
    app: httpbin
spec:
  ports:
  - name: http
    port: 8000
  selector:
    app: httpbin
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: httpbin
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: httpbin
        version: v1
    spec:
      containers:
      - image: docker.io/citizenstig/httpbin
        imagePullPolicy: IfNotPresent
        name: httpbin
        ports:
        - containerPort: 8000
EOF
```

2.为 Ingress 网关创建 `Secret：`

```shell
[root@VM_0_2_centos istio-1.2.2]# kubectl create -n istio-system secret generic httpbin-credential \
> --from-file=key=httpbin.example.com/3_application/private/httpbin.example.com.key.pem \
> --from-file=cert=httpbin.example.com/3_application/certs/httpbin.example.com.cert.pem
secret/httpbin-credential created
```

3.创建一个网关，其 `servers:` 字段的端口为 443，设置 `credentialName` 的值为 `httpbin-credential`。这个值就是 `Secret` 的名字。TLS 模式设置为 `SIMPLE`。

```shell
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: mygateway
spec:
  selector:
    istio: ingressgateway # 使用缺省的 Ingress 网关。
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: "httpbin-credential" # 和 Secret 名称一致
    hosts:
    - "httpbin.example.com"
EOF
```

4.配置网关的 Ingress 流量路由，并配置对应的 `VirtualService`：

```shell
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
spec:
  hosts:
  - "httpbin.example.com"
  gateways:
  - mygateway
  http:
  - match:
    - uri:
        prefix: /status
    - uri:
        prefix: /delay
    route:
    - destination:
        port:
          number: 8000
        host: httpbin
EOF
```

5.用 HTTPS 协议访问 `httpbin` 服务：

```shell
curl -v -HHost:httpbin.example.com \
--resolve httpbin.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST \
--cacert httpbin.example.com/2_intermediate/certs/ca-chain.cert.pem \
https://httpbin.example.com:$SECURE_INGRESS_PORT/status/418
```

`httpbin` 服务会返回 [418 I’m a Teapot](https://tools.ietf.org/html/rfc7168#section-2.3.3)。

6.删除网关的 `Secret`，并新建另外一个，然后修改 Ingress 网关的凭据：

```shell
kubectl -n istio-system delete secret httpbin-credential
```

7.使用curl访问

```shell
[root@VM_0_2_centos istio-1.2.2]# curl -v -HHost:httpbin.example.com --resolve httpbin.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST --cacert httpbin.example.com/2_intermediate/certs/ca-chain.cert.pem https://httpbin.example.com:$SECURE_INGRESS_PORT/status/418
* Added httpbin.example.com:443:10.108.162.3 to DNS cache
* About to connect() to httpbin.example.com port 443 (#0)
*   Trying 10.108.162.3...
* Connected to httpbin.example.com (10.108.162.3) port 443 (#0)
* Initializing NSS with certpath: sql:/etc/pki/nssdb
*   CAfile: httpbin.example.com/2_intermediate/certs/ca-chain.cert.pem
  CApath: none
* SSL connection using TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
* Server certificate:
* 	subject: CN=httpbin.example.com,O=Dis,L=Springfield,ST=Denial,C=US
* 	start date: 8月 06 09:48:13 2019 GMT
* 	expire date: 8月 15 09:48:13 2020 GMT
* 	common name: httpbin.example.com
* 	issuer: CN=httpbin.example.com,O=Dis,ST=Denial,C=US
> GET /status/418 HTTP/1.1
> User-Agent: curl/7.29.0
> Accept: */*
> Host:httpbin.example.com
>
< HTTP/1.1 418 Unknown
< server: istio-envoy
< date: Wed, 07 Aug 2019 10:04:24 GMT
< access-control-allow-origin: *
< x-more-info: http://tools.ietf.org/html/rfc2324
< access-control-allow-credentials: true
< content-length: 135
< x-envoy-upstream-service-time: 94
<

    -=[ teapot ]=-

       _...._
     .'  _ _ `.
    | ."` ^ `". _,
    \_;`"---"`|//
      |       ;/
      \_     _/
        `"""`
* Connection #0 to host httpbin.example.com left intact
```

为 TLS Ingress 网关配置多个主机名

可以把多个主机名配置到同一个 Ingress 网关上，例如 `httpbin.example.com` 和 `helloworld-v1.example.com`。Ingress 网关会为每个 `credentialName` 获取一个唯一的凭据。

###### 2.为 TLS Ingress 网关配置多个主机名

可以把多个主机名配置到同一个 Ingress 网关上，例如 `httpbin.example.com` 和 `helloworld-v1.example.com`。Ingress 网关会为每个 `credentialName` 获取一个唯一的凭据。

* 启动 `hellowworld-v1` 示例：

```shell
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: helloworld-v1
  labels:
    app: helloworld-v1
spec:
  ports:
  - name: http
    port: 5000
  selector:
    app: helloworld-v1
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: helloworld-v1
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: helloworld-v1
    spec:
      containers:
      - name: helloworld
        image: istio/examples-helloworld-v1
        resources:
          requests:
            cpu: "100m"
        imagePullPolicy: IfNotPresent #Always
        ports:
        - containerPort: 5000
EOF
```

* 为 Ingress 网关创建一个 Ingress。如果已经创建了 `httpbin-credential`，就可以创建 `helloworld-credential` Secret 了。

```shell
pushd mtls-go-example
./generate.sh helloworld-v1.example.com password
mkdir ../helloworld-v1.example.com && mv 1_root 2_intermediate 3_application 4_client ../helloworld-v1.example.com
popd
kubectl create -n istio-system secret generic helloworld-credential \
--from-file=key=helloworld-v1.example.com/3_application/private/helloworld-v1.example.com.key.pem \
--from-file=cert=helloworld-v1.example.com/3_application/certs/helloworld-v1.example.com.cert.pem

[root@VM_0_3_centos ~]# kubectl get secret -n istio-system
NAME                                                 TYPE                                  DATA   AGE
default-token-q69rn                                  kubernetes.io/service-account-token   3      9d
helloworld-credential                                Opaque                                2      49s
```

* 定义一个网关，其中包含了两个 `server`，都开放了 443 端口。两个 `credentialName` 字段分别赋值为 `httpbin-credential` 和 `helloworld-credential`。`serverCertificate` 以及 `privateKey` 应该为空。

```shell
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: mygateway
spec:
  selector:
    istio: ingressgateway # 使用缺省的 Ingress 网关
  servers:
  - port:
      number: 443
      name: https-httpbin
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: "httpbin-credential"
    hosts:
    - "httpbin.example.com"
  - port:
      number: 443
      name: https-helloworld
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: "helloworld-credential"
    hosts:
    - "helloworld-v1.example.com"
EOF

# 查看
[root@VM_0_3_centos ~]# kubectl describe gw mygateway
Name:         mygateway
Namespace:    default
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"networking.istio.io/v1alpha3","kind":"Gateway","metadata":{"annotations":{},"name":"mygateway","namespace":"default"},"spec...
API Version:  networking.istio.io/v1alpha3
Kind:         Gateway
Metadata:
  Creation Timestamp:  2019-08-07T09:48:12Z
  Generation:          2
  Resource Version:    676604
  Self Link:           /apis/networking.istio.io/v1alpha3/namespaces/default/gateways/mygateway
  UID:                 0e8f2445-2112-46c8-b450-00073f3da98e
Spec:
  Selector:
    Istio:  ingressgateway
  Servers:
    Hosts:
      httpbin.example.com
    Port:
      Name:      https-httpbin
      Number:    443
      Protocol:  HTTPS
    Tls:
      Credential Name:  httpbin-credential
      Mode:             SIMPLE
    Hosts:
      helloworld-v1.example.com
    Port:
      Name:      https-helloworld
      Number:    443
      Protocol:  HTTPS
    Tls:
      Credential Name:  helloworld-credential
      Mode:             SIMPLE
Events:                 <none>
```

* 配置网关的流量路由，配置 `VirtualService`

```shell
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: helloworld-v1
spec:
  hosts:
  - "helloworld-v1.example.com"
  gateways:
  - mygateway
  http:
  - match:
    - uri:
        exact: /hello
    route:
    - destination:
        host: helloworld-v1
        port:
          number: 5000
EOF
```

* 向 `helloworld-v1.example.com` 发送 HTTPS 请求

```shell
curl -v -HHost:helloworld-v1.example.com \
--resolve helloworld-v1.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST \
--cacert helloworld-v1.example.com/2_intermediate/certs/ca-chain.cert.pem \
https://helloworld-v1.example.com:$SECURE_INGRESS_PORT/hello


[root@VM_0_3_centos ~]# curl -v -HHost:helloworld-v1.example.com \
> --resolve helloworld-v1.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST \
> --cacert helloworld-v1.example.com/2_intermediate/certs/ca-chain.cert.pem \
> https://helloworld-v1.example.com:$SECURE_INGRESS_PORT/hello
* Added helloworld-v1.example.com:31390:172.16.0.3 to DNS cache
* About to connect() to helloworld-v1.example.com port 31390 (#0)
*   Trying 172.16.0.3...
* Connected to helloworld-v1.example.com (172.16.0.3) port 31390 (#0)
* Initializing NSS with certpath: sql:/etc/pki/nssdb
*   CAfile: helloworld-v1.example.com/2_intermediate/certs/ca-chain.cert.pem
  CApath: none
* SSL connection using TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
* Server certificate:
* 	subject: CN=helloworld-v1.example.com,O=Dis,L=Springfield,ST=Denial,C=US
* 	start date: 8月 14 08:50:44 2019 GMT
* 	expire date: 8月 23 08:50:44 2020 GMT
* 	common name: helloworld-v1.example.com
* 	issuer: CN=helloworld-v1.example.com,O=Dis,ST=Denial,C=US
> GET /hello HTTP/1.1
> User-Agent: curl/7.29.0
> Accept: */*
> Host:helloworld-v1.example.com
>
< HTTP/1.1 200 OK
< content-type: text/html; charset=utf-8
< content-length: 59
< server: istio-envoy
< date: Wed, 14 Aug 2019 09:14:02 GMT
< x-envoy-upstream-service-time: 132
<
Hello version: v1, instance: helloworld-v1-b99c6449c-xchbg
* Connection #0 to host helloworld-v1.example.com left intact


```

* 发送 HTTPS 请求到 `httpbin.example.com`，还是会看到茶壶：

```shell
curl -v -HHost:httpbin.example.com \
--resolve httpbin.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST \
--cacert httpbin.example.com/2_intermediate/certs/ca-chain.cert.pem \
https://httpbin.example.com:$SECURE_INGRESS_PORT/status/418
```

###### 3.配置双向 TLS Ingress 网关

可以对网关的定义进行扩展，加入[双向 TLS](https://en.wikipedia.org/wiki/Mutual_authentication) 的支持。要修改 Ingress 网关的凭据，就要删除并重建对应的 `Secret`。服务器会使用 CA 证书对客户端进行校验，因此需要使用 `cacert` 字段来保存 CA 证书：

```shell
kubectl -n istio-system delete secret httpbin-credential
kubectl create -n istio-system secret generic httpbin-credential  \
--from-file=key=httpbin.example.com/3_application/private/httpbin.example.com.key.pem \
--from-file=cert=httpbin.example.com/3_application/certs/httpbin.example.com.cert.pem \
--from-file=cacert=httpbin.example.com/2_intermediate/certs/ca-chain.cert.pem
```

* 修改网关定义，设置 TLS 的模式为 `MUTUAL`：

```shell
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
 name: mygateway
spec:
 selector:
   istio: ingressgateway # Istio 的缺省 Ingress 网关
 servers:
 - port:
     number: 443
     name: https
     protocol: HTTPS
   tls:
     mode: MUTUAL
     credentialName: "httpbin-credential" # 和 Secret 名称一致
   hosts:
   - "httpbin.example.com"
EOF
```

* 使用前面的方式尝试发出 HTTPS 请求，会看到失败的过程：

```shell
curl -v -HHost:httpbin.example.com \
--resolve httpbin.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST \
--cacert httpbin.example.com/2_intermediate/certs/ca-chain.cert.pem \
https://httpbin.example.com:$SECURE_INGRESS_PORT/status/418

[root@VM_0_3_centos istio-1.2.2]# curl -v -HHost:httpbin.example.com \
> --resolve httpbin.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST \
> --cacert httpbin.example.com/2_intermediate/certs/ca-chain.cert.pem \
> https://httpbin.example.com:$SECURE_INGRESS_PORT/status/418
* Added httpbin.example.com:31390:172.16.0.3 to DNS cache
* About to connect() to httpbin.example.com port 31390 (#0)
*   Trying 172.16.0.3...
* Connected to httpbin.example.com (172.16.0.3) port 31390 (#0)
* Initializing NSS with certpath: sql:/etc/pki/nssdb
*   CAfile: httpbin.example.com/2_intermediate/certs/ca-chain.cert.pem
  CApath: none
* NSS: client certificate not found (nickname not specified)
* NSS error -12227 (SSL_ERROR_HANDSHAKE_FAILURE_ALERT)
* SSL peer was unable to negotiate an acceptable set of security parameters.
* Closing connection 0
curl: (35) NSS: client certificate not found (nickname not specified)
```

* 在 `curl` 命令中加入客户端证书和私钥的参数，重新发送请求。（客户端证书参数为 `--cert`，私钥参数为 `--key`）

```shell
curl -v -HHost:httpbin.example.com \
--resolve httpbin.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST \
--cacert httpbin.example.com/2_intermediate/certs/ca-chain.cert.pem \
--cert httpbin.example.com/4_client/certs/httpbin.example.com.cert.pem \
--key httpbin.example.com/4_client/private/httpbin.example.com.key.pem \
https://httpbin.example.com:$SECURE_INGRESS_PORT/status/418

[root@VM_0_3_centos istio-1.2.2]# curl -v -HHost:httpbin.example.com \
> --resolve httpbin.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST \
> --cacert httpbin.example.com/2_intermediate/certs/ca-chain.cert.pem \
> --cert httpbin.example.com/4_client/certs/httpbin.example.com.cert.pem \
> --key httpbin.example.com/4_client/private/httpbin.example.com.key.pem \
> https://httpbin.example.com:$SECURE_INGRESS_PORT/status/418
* Added httpbin.example.com:31390:172.16.0.3 to DNS cache
* About to connect() to httpbin.example.com port 31390 (#0)
*   Trying 172.16.0.3...
* Connected to httpbin.example.com (172.16.0.3) port 31390 (#0)
* Initializing NSS with certpath: sql:/etc/pki/nssdb
*   CAfile: httpbin.example.com/2_intermediate/certs/ca-chain.cert.pem
  CApath: none
* NSS: client certificate from file
* 	subject: CN=httpbin.example.com,O=Dis,L=Springfield,ST=Denial,C=US
* 	start date: 8月 06 09:48:15 2019 GMT
* 	expire date: 8月 15 09:48:15 2020 GMT
* 	common name: httpbin.example.com
* 	issuer: CN=httpbin.example.com,O=Dis,ST=Denial,C=US
* SSL connection using TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
* Server certificate:
* 	subject: CN=httpbin.example.com,O=Dis,L=Springfield,ST=Denial,C=US
* 	start date: 8月 06 09:48:13 2019 GMT
* 	expire date: 8月 15 09:48:13 2020 GMT
* 	common name: httpbin.example.com
* 	issuer: CN=httpbin.example.com,O=Dis,ST=Denial,C=US
> GET /status/418 HTTP/1.1
> User-Agent: curl/7.29.0
> Accept: */*
> Host:httpbin.example.com
>
< HTTP/1.1 418 Unknown
< server: istio-envoy
< date: Wed, 14 Aug 2019 09:24:53 GMT
< access-control-allow-origin: *
< x-more-info: http://tools.ietf.org/html/rfc2324
< access-control-allow-credentials: true
< content-length: 135
< x-envoy-upstream-service-time: 5
<

    -=[ teapot ]=-

       _...._
     .'  _ _ `.
    | ."` ^ `". _,
    \_;`"---"`|//
      |       ;/
      \_     _/
        `"""`
* Connection #0 to host httpbin.example.com left intact
```

###### 清理

* 删除网关配置、`VirtualService` 以及 `Secret`：

```shell
$ kubectl delete gateway mygateway
$ kubectl delete virtualservice httpbin
$ kubectl delete --ignore-not-found=true -n istio-system secret httpbin-credential \
helloworld-credential
$ kubectl delete --ignore-not-found=true virtualservice helloworld-v1
```

* 删除证书目录以及用于生成证书的代码库：

```
$ rm -rf httpbin.example.com helloworld-v1.example.com mtls-go-example
```

* 删除用于重新部署 Ingress 网关的文件：

```
$ rm -f $HOME/istio-ingressgateway.yaml
```

* 关闭 `httpbin` 和 `helloworld-v1` 服务：

```
$ kubectl delete service --ignore-not-found=true helloworld-v1
$ kubectl delete service --ignore-not-found=true httpbin
```

##### 控制 Egress 流量

缺省情况下，Istio 服务网格内的 Pod，由于其 iptables 将所有外发流量都透明的转发给了 Sidecar，所以这些集群内的服务无法访问集群之外的 URL，而只能处理集群内部的目标。

本文的任务描述了如何将外部服务暴露给 Istio 集群中的客户端。你将会学到如何通过定义 [`ServiceEntry`](https://istio.io/zh/docs/reference/config/istio.networking.v1alpha3/#serviceentry) 来调用外部服务；或者简单的对 Istio 进行配置，要求其直接放行对特定 IP 范围的访问。

###### 启动示例应用

```shell
kubectl apply -f <(istioctl kube-inject -f samples/sleep/sleep.yaml)
```

将 `SOURCE_POD` 环境变量设置为已部署的 `sleep` pod：

```shell
export SOURCE_POD=$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})
```

###### 配置外部服务

通过配置 Istio `ServiceEntry`，可以从 Istio 集群中访问任何可公开访问的服务。 这里我们会使用 [httpbin.org](http://httpbin.org/) 以及 [www.google.com](https://www.google.com/) 进行试验。

* 创建一个 `ServiceEntry` 对象，放行对一个外部 HTTP 服务的访问：

```shell
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: httpbin-ext
spec:
  hosts:
  - httpbin.org
  ports:
  - number: 80
    name: http
    protocol: HTTP
  resolution: DNS
  location: MESH_EXTERNAL
EOF

[root@VM_0_3_centos istio-1.2.2]# kubectl get se
NAME          HOSTS           LOCATION        RESOLUTION   AGE
httpbin-ext   [httpbin.org]   MESH_EXTERNAL   DNS          3s
```

* 创建一个 `ServiceEntry` 以及 `VirtualService`，允许访问外部 HTTPS 服务。注意：包括 HTTPS 在内的 TLS 协议，在 `ServiceEntry` 之外，还需要创建 TLS `VirtualService`。

```shell
kubectl exec -it $SOURCE_POD -c sleep sh
```

* 向外部 HTTP 服务发出请求：

```shell
curl http://httpbin.org/headers
```

###### 配置外部HTTPS服务

* 创建一个 `ServiceEntry` 以允许访问外部 HTTPS 服务。 对于 TLS 协议（包括 HTTPS），除了 `ServiceEntry` 之外，还需要 `VirtualService`。 没有 `VirtualService`, `ServiceEntry` 所暴露的服务将不被定义。`VirtualService` 必须在 `match` 子句中包含 `tls` 规则和 `sni_hosts` 以启用 SNI 路由。

```shell
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: google
spec:
  hosts:
  - www.google.com
  ports:
  - number: 443
    name: https
    protocol: HTTPS
  resolution: DNS
  location: MESH_EXTERNAL
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: google
spec:
  hosts:
  - www.google.com
  tls:
  - match:
    - port: 443
      sni_hosts:
      - www.google.com
    route:
    - destination:
        host: www.google.com
        port:
          number: 443
      weight: 100
EOF


# 测试由于网络因素，访问https的百度

kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: baidu
spec:
  hosts:
  - www.baidu.com
  ports:
  - number: 443
    name: https
    protocol: HTTPS
  resolution: DNS
  location: MESH_EXTERNAL
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: baidu
spec:
  hosts:
  - www.baidu.com
  tls:
  - match:
    - port: 443
      sni_hosts:
      - www.baidu.com
    route:
    - destination:
        host: www.baidu.com
        port:
          number: 443
      weight: 100
EOF
```

* 执行 `sleep service` 源 pod：

```shell
kubectl exec -it $SOURCE_POD -c sleep sh
```

* 向外部 HTTPS 服务发出请求：

```shell
curl https://www.baidu.com
```

###### 为外部服务设置路由规则

通过 `ServiceEntry` 访问外部服务的流量，和网格内流量类似，都可以进行 Istio [路由规则](https://istio.io/zh/docs/concepts/traffic-management/#规则配置) 的配置。下面我们使用 [`istioctl`](https://istio.io/zh/docs/reference/commands/istioctl/) 为 httpbin.org 服务设置一个超时规则。

* 在测试 Pod 内部，使用 `curl` 调用 httpbin.org 这一外部服务的 `/delay` 端点：

```shell
kubectl exec -it $SOURCE_POD -c sleep sh
time curl -o /dev/null -s -w "%{http_code}\n" http://httpbin.org/delay/5
```

这个请求会在大概五秒钟左右返回一个内容为 `200 (OK)` 的响应。

* 退出测试 Pod，使用 `kubectl` 为 httpbin.org 外部服务的访问设置一个 3 秒钟的超时：

```shell
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin-ext
spec:
  hosts:
    - httpbin.org
  http:
  - timeout: 3s
    route:
      - destination:
          host: httpbin.org
        weight: 100
EOF
```

* 等待几秒钟之后，再次发起 *curl* 请求：

```shell
$ kubectl exec -it $SOURCE_POD -c sleep sh
$ time curl -o /dev/null -s -w "%{http_code}\n" http://httpbin.org/delay/5
```

这一次会在 3 秒钟之后收到一个内容为 `504 (Gateway Timeout)` 的响应。虽然 httpbin.org 还在等待他的 5 秒钟，Istio 却在 3 秒钟的时候切断了请求。

###### 直接调用外部服务

如果想要跳过 Istio，直接访问某个 IP 范围内的外部服务，就需要对 Envoy sidecar 进行配置，阻止 Envoy 对外部请求的[劫持](https://istio.io/zh/docs/concepts/traffic-management/#服务之间的通讯)。可以在 [Helm](https://istio.io/zh/docs/reference/config/installation-options/) 中设置 `global.proxy.includeIPRanges` 变量，然后使用 `kubectl apply` 命令来更新名为 `istio-sidecar-injector` 的 `Configmap`。在 `istio-sidecar-injector` 更新之后，`global.proxy.includeIPRanges` 会在所有未来部署的 Pod 中生效。

使用 `global.proxy.includeIPRanges` 变量的最简单方式就是把内部服务的 IP 地址范围传递给它，这样就在 Sidecar proxy 的重定向列表中排除掉了外部服务的地址了。

内部服务的 IP 范围取决于集群的部署情况。例如 Minikube 中这一范围是 `10.0.0.1/24`，这个配置中，就应该这样更新 `istio-sidecar-injector`：

```shell
helm template install/kubernetes/helm/istio <安装 Istio 时所使用的参数> --set global.proxy.includeIPRanges="10.0.0.1/24" -x templates/sidecar-injector-configmap.yaml | kubectl apply -f -

```

注意这里应该使用和之前部署 Istio 的时候同样的 [Helm 命令](https://istio.io/zh/docs/setup/kubernetes/install/helm)，尤其是 `--namespace` 参数。在安装 Istio 原有命令的基础之上，加入 `--set global.proxy.includeIPRanges="10.0.0.1/24" -x templates/sidecar-injector-configmap.yaml` 即可。

[和前面一样](https://istio.io/zh/docs/tasks/traffic-management/egress/#开始之前)，重新部署 `sleep` 应用。

确保已删除之前部署的 `ServiceEntry` 和 `VirtualService`。

###### 理解原理

这个任务中，我们使用两种方式从 Istio 服务网格内部来完成对外部服务的调用：

1. 使用 `ServiceEntry` (推荐方式)
2. 配置 Istio sidecar，从它的重定向 IP 表中排除外部服务的 IP 范围

第一种方式（`ServiceEntry`）中，网格内部的服务不论是访问内部还是外部的服务，都可以使用同样的 Istio 服务网格的特性。我们通过为外部服务访问设置超时规则的例子，来证实了这一优势。

第二种方式越过了 Istio sidecar proxy，让服务直接访问到对应的外部地址。然而要进行这种配置，需要了解云供应商特定的知识和配置。

###### 清理

* 删除规则：

```
$ kubectl delete serviceentry httpbin-ext google
$ kubectl delete virtualservice httpbin-ext google
```

* 停止 [sleep](https://github.com/istio/istio/tree/release-1.2/samples/sleep) 服务：

```
$ kubectl delete -f samples/sleep/sleep.yaml
```

* 更新 `ConfigMap` `istio-sidecar-injector`，要求 Sidecar 转发所有外发流量：

```
$ helm template install/kubernetes/helm/istio <安装 Istio 时所使用的参数> -x templates/sidecar-injector-configmap.yaml | kubectl apply -f -
```

##### 熔断

本任务展示了用连接、请求以及外部检测来进行熔断配置的过程。

断路器是创建弹性微服务应用程序的重要模式。断路器允许您编写限制故障、延迟峰值以及其他不良网络特性影响的应用程序。

在此任务中，您将配置断路器规则，然后通过故意“跳闸”断路器来测试配置。

启动应用

```shell
kubectl apply -f <(istioctl kube-inject -f samples/httpbin/httpbin.yaml)
```

###### 断路器

* 创建一个 [目标规则](https://istio.io/zh/docs/reference/config/istio.networking.v1alpha3/#destinationrule)，针对 `httpbin` 服务设置断路器：

如果您的 Istio 启用了双向 TLS 身份验证，则必须在应用之前将 TLS 流量策略 `mode：ISTIO_MUTUAL` 添加到 `DestinationRule`。否则请求将产生 503 错误，如[设置目标规则后出现 503 错误](https://istio.io/zh/docs/ops/traffic-management/troubleshooting/#设置目标规则后出现-503-错误)所述。

```shell
kubectl apply -f - <<EOF
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
      consecutiveErrors: 1
      interval: 1s
      baseEjectionTime: 3m
      maxEjectionPercent: 100
EOF
```

* 检查我们的目标规则，确定已经正确建立：

```shell
kubectl get destinationrule httpbin -o yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: httpbin
  ...
spec:
  host: httpbin
  trafficPolicy:
    connectionPool:
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
      tcp:
        maxConnections: 1
    outlierDetection:
      baseEjectionTime: 180.000s
      consecutiveErrors: 1
      interval: 1.000s
      maxEjectionPercent: 100
```

###### 设置客户端

现在我们已经设置了调用 `httpbin` 服务的规则，接下来创建一个客户端，用来向后端服务发送请求，观察是否会触发熔断策略。这里要使用一个简单的负载测试客户端，名字叫 [fortio](https://github.com/istio/fortio)。这个客户端可以控制连接数量、并发数以及发送 HTTP 请求的延迟。使用这一客户端，能够有效的触发前面在目标规则中设置的熔断策略。

* 这里我们会把给客户端也进行 Sidecar 的注入，以此保证 Istio 对网络交互的控制：

  ```
  kubectl apply -f <(istioctl kube-inject -f samples/httpbin/sample-client/fortio-deploy.yaml)
  ```

* 接下来就可以登入客户端 Pod 并使用 Fortio 工具来调用 `httpbin`。`-curl` 参数表明只调用一次：

  ```
  $ FORTIO_POD=$(kubectl get pod | grep fortio | awk '{ print $1 }')
  $ kubectl exec -it $FORTIO_POD  -c fortio /usr/bin/fortio -- load -curl  http://httpbin:8000/get
  
  HTTP/1.1 200 OK
  server: envoy
  date: Tue, 16 Jan 2018 23:47:00 GMT
  content-type: application/json
  access-control-allow-origin: *
  access-control-allow-credentials: true
  content-length: 445
  x-envoy-upstream-service-time: 36
  
  {
    "args": {},
    "headers": {
      "Content-Length": "0",
      "Host": "httpbin:8000",
      "User-Agent": "istio/fortio-0.6.2",
      "X-B3-Sampled": "1",
      "X-B3-Spanid": "824fbd828d809bf4",
      "X-B3-Traceid": "824fbd828d809bf4",
      "X-Ot-Span-Context": "824fbd828d809bf4;824fbd828d809bf4;0000000000000000",
      "X-Request-Id": "1ad2de20-806e-9622-949a-bd1d9735a3f4"
    },
    "origin": "127.0.0.1",
    "url": "http://httpbin:8000/get"
  }
  ```

  不难看出，调用已经成功。接下来做些变化。

###### 触发熔断机制

在上面的熔断设置中指定了 `maxConnections: 1` 以及 `http1MaxPendingRequests: 1`。这意味着如果超过了一个连接同时发起请求，Istio 就会熔断，阻止后续的请求或连接。

* 接下来尝试一下两个并发连接（`-c 2`），发送 20 请求（`-n 20`）：

```shell
kubectl exec -it $FORTIO_POD  -c fortio /usr/bin/fortio -- load -c 2 -qps 0 -n 20 -loglevel Warning http://httpbin:8000/get

[root@VM_0_3_centos istio-1.2.2]# kubectl exec -it $FORTIO_POD  -c fortio /usr/bin/fortio -- load -c 2 -qps 0 -n 20 -loglevel Warning http://httpbin:8000/get
10:31:28 I logger.go:97> Log level is now 3 Warning (was 2 Info)
Fortio 1.3.1 running at 0 queries per second, 2->2 procs, for 20 calls: http://httpbin:8000/get
Starting at max qps with 2 thread(s) [gomax 2] for exactly 20 calls (10 per thread + 0)
10:31:28 W http_client.go:679> Parsed non ok code 503 (HTTP/1.1 503)
Ended after 37.04095ms : 20 calls. qps=539.94
Aggregated Function Time : count 20 avg 0.0035982768 +/- 0.001634 min 0.001340763 max 0.006110438 sum 0.071965536
# range, mid point, percentile, count
>= 0.00134076 <= 0.002 , 0.00167038 , 25.00, 5
> 0.002 <= 0.003 , 0.0025 , 45.00, 4
> 0.003 <= 0.004 , 0.0035 , 55.00, 2
> 0.004 <= 0.005 , 0.0045 , 65.00, 2
> 0.005 <= 0.006 , 0.0055 , 90.00, 5
> 0.006 <= 0.00611044 , 0.00605522 , 100.00, 2
# target 50% 0.0035
# target 75% 0.0054
# target 90% 0.006
# target 99% 0.00609939
# target 99.9% 0.00610933
Sockets used: 3 (for perfect keepalive, would be 2)
Code 200 : 19 (95.0 %)
Code 503 : 1 (5.0 %)
Response Header Sizes : count 20 avg 291.65 +/- 66.91 min 0 max 307 sum 5833
Response Body/Total Sizes : count 20 avg 616.3 +/- 68.43 min 318 max 632 sum 12326
All done 20 calls (plus 0 warmup) 3.598 ms avg, 539.9 qps
```

* 接下来把并发连接数量提高到 3：

```shell
kubectl exec -it $FORTIO_POD  -c fortio /usr/bin/fortio -- load -c 3 -qps 0 -n 30 -loglevel Warning http://httpbin:8000/get

#这时候会观察到，熔断行为按照之前的设计生效了，只有 63.3% 的请求获得通过，剩余请求被断路器拦截了：
Code 200 : 19 (63.3 %)
Code 503 : 11 (36.7 %)
```

* 我们可以查询 `istio-proxy` 的状态，获取更多相关信息：

```shell
$ kubectl exec -it $FORTIO_POD  -c istio-proxy  -- sh -c 'curl localhost:15000/stats' | grep httpbin | grep pending
cluster.outbound|80||httpbin.springistio.svc.cluster.local.upstream_rq_pending_active: 0
cluster.outbound|80||httpbin.springistio.svc.cluster.local.upstream_rq_pending_failure_eject: 0
cluster.outbound|80||httpbin.springistio.svc.cluster.local.upstream_rq_pending_overflow: 12
cluster.outbound|80||httpbin.springistio.svc.cluster.local.upstream_rq_pending_total: 39
```

`upstream_rq_pending_overflow` 的值是 `12`，说明有 `12` 次调用被标志为熔断。

###### 清理

* 清理规则：

```
$ kubectl delete destinationrule httpbin
```

* 关闭 [httpbin](https://github.com/istio/istio/tree/release-1.2/samples/httpbin) 服务和客户端：

```
$ kubectl delete deploy httpbin fortio-deploy
$ kubectl delete svc httpbin
```

##### 镜像

此任务演示了 Istio 的流量镜像功能。

流量镜像，也称为影子流量，是一个以尽可能低的风险为生产带来变化的强大的功能。镜像会将实时流量的副本发送到镜像服务。镜像流量发生在主服务的关键请求路径之外。

在此任务中，首先把所有流量路由到 `v1` 测试服务。然后使用规则将一部分流量镜像到 `v2`。

###### 启动示例应用

按照[安装指南](https://istio.io/zh/docs/setup/)中的说明设置 Istio。

首先部署启用了访问日志的两个版本的 [httpbin](https://github.com/istio/istio/tree/release-1.2/samples/httpbin) 服务：

**httpbin-v1:**

```shell
cat <<EOF | istioctl kube-inject -f - | kubectl create -f -
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: httpbin-v1
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: httpbin
        version: v1
    spec:
      containers:
      - image: docker.io/kennethreitz/httpbin
        imagePullPolicy: IfNotPresent
        name: httpbin
        command: ["gunicorn", "--access-logfile", "-", "-b", "0.0.0.0:80", "httpbin:app"]
        ports:
        - containerPort: 80
EOF
```

**httpbin-v2:**

```shell
cat <<EOF | istioctl kube-inject -f - | kubectl create -f -
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: httpbin-v2
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: httpbin
        version: v2
    spec:
      containers:
      - image: docker.io/kennethreitz/httpbin
        imagePullPolicy: IfNotPresent
        name: httpbin
        command: ["gunicorn", "--access-logfile", "-", "-b", "0.0.0.0:80", "httpbin:app"]
        ports:
        - containerPort: 80
EOF
```

**httpbin Kubernetes service:**



```shell
kubectl create -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: httpbin
  labels:
    app: httpbin
spec:
  ports:
  - name: http
    port: 8000
    targetPort: 80
  selector:
    app: httpbin
EOF
```

启动 `sleep` 服务，这样就可以使用 `curl` 来提供负载了：

**sleep service:**

```shell
cat <<EOF | istioctl kube-inject -f - | kubectl create -f -
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: sleep
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: sleep
    spec:
      containers:
      - name: sleep
        image: tutum/curl
        command: ["/bin/sleep","infinity"]
        imagePullPolicy: IfNotPresent
EOF
```

###### 创建默认路由策略

默认情况下，Kubernetes 在 `httpbin` 服务的两个版本之间进行负载均衡。在此步骤中会更改该行为，把所有流量都路由到 `v1`。

* 创建一个默认路由规则，将所有流量路由到服务的 `v1` ：

如果为 Istio 启用了双向 TLS 认证，在应用前必须将 TLS 流量策略 `mode: ISTIO_MUTUAL` 添加到 `DestinationRule`。否则，请求将发生 503 错误，如[设置目标规则后出现 503 错误](https://istio.io/zh/docs/ops/traffic-management/troubleshooting/#设置目标规则后出现-503-错误)所述。

```
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
spec:
  hosts:
    - httpbin
  http:
  - route:
    - destination:
        host: httpbin
        subset: v1
      weight: 100
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: httpbin
spec:
  host: httpbin
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
EOF
```

现在所有流量已经都转到 `httpbin:v1` 服务。

* 向服务发送一些流量：

```
$ export SLEEP_POD=$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})
$ kubectl exec -it $SLEEP_POD -c sleep -- sh -c 'curl  http://httpbin:8000/headers' | python -m json.tool

{
  "headers": {
    "Accept": "*/*",
    "Content-Length": "0",
    "Host": "httpbin:8000",
    "User-Agent": "curl/7.35.0",
    "X-B3-Sampled": "1",
    "X-B3-Spanid": "eca3d7ed8f2e6a0a",
    "X-B3-Traceid": "eca3d7ed8f2e6a0a",
    "X-Ot-Span-Context": "eca3d7ed8f2e6a0a;eca3d7ed8f2e6a0a;0000000000000000"
  }
}
```



* 查看 `httpbin` pods 的 `v1` 和 `v2` 日志。您可以看到 `v1` 的访问日志和 `v2` 为 `<none>` 的日志：

```
$ export V1_POD=$(kubectl get pod -l app=httpbin,version=v1 -o jsonpath={.items..metadata.name})
$ kubectl logs -f $V1_POD -c httpbin

127.0.0.1 - - [07/Mar/2018:19:02:43 +0000] "GET /headers HTTP/1.1" 200 321 "-" "curl/7.35.0"
```



```
$ export V2_POD=$(kubectl get pod -l app=httpbin,version=v2 -o jsonpath={.items..metadata.name})
$ kubectl logs -f $V2_POD -c httpbin

<none>
```



###### 镜像流量到 v2

* 改变路由规则将流量镜像到 v2:

```
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
spec:
  hosts:
    - httpbin
  http:
  - route:
    - destination:
        host: httpbin
        subset: v1
      weight: 100
    mirror:
      host: httpbin
      subset: v2
EOF
```

此路由规则将 100％ 的流量发送到 `v1` 。最后一节指定镜像到 `httpbin:v2` 服务。当流量被镜像时，请求将通过其主机/授权报头发送到镜像服务附上 `-shadow`。例如，将 `cluster-1` 变为 `cluster-1-shadow`。

此外，重点注意这些被镜像的请求是“即发即弃”的，也就是说这些请求引发的响应是会被丢弃的。

* 发送流量：

```
$ kubectl exec -it $SLEEP_POD -c sleep -- sh -c 'curl  http://httpbin:8000/headers' | python -m json.tool
```



这样就可以看到 `v1` 和 `v2` 中都有了访问日志。`v2` 中的访问日志就是由镜像流量产生的，这些请求的实际目标是 `v1`：

```
$ kubectl logs -f $V1_POD -c httpbin

127.0.0.1 - - [07/Mar/2018:19:02:43 +0000] "GET /headers HTTP/1.1" 200 321 "-" "curl/7.35.0"
127.0.0.1 - - [07/Mar/2018:19:26:44 +0000] "GET /headers HTTP/1.1" 200 321 "-" "curl/7.35.0"
```

```
$ kubectl logs -f $V2_POD -c httpbin

127.0.0.1 - - [07/Mar/2018:19:26:44 +0000] "GET /headers HTTP/1.1" 200 361 "-" "curl/7.35.0"
```

* 如果要检查流量内部，请在另一个控制台上运行以下命令：

```
$ export SLEEP_POD=$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})
$ export V1_POD_IP=$(kubectl get pod -l app=httpbin -l version=v1 -o jsonpath={.items..status.podIP})
$ export V2_POD_IP=$(kubectl get pod -l app=httpbin -l version=v2 -o jsonpath={.items..status.podIP})
$ kubectl exec -it $SLEEP_POD -c istio-proxy -- sudo tcpdump -A -s 0 host $V1_POD_IP or host $V2_POD_IP

tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
05:47:50.159513 IP sleep-7b9f8bfcd-2djx5.38836 > 10-233-75-11.httpbin.default.svc.cluster.local.80: Flags [P.], seq 4039989036:4039989832, ack 3139734980, win 254, options [nop,nop,TS val 77427918 ecr 76730809], length 796: HTTP: GET /headers HTTP/1.1
E..P2.X.X.X.
.K.
.K....P..W,.$.......+.....
..t.....GET /headers HTTP/1.1
host: httpbin:8000
user-agent: curl/7.35.0
accept: */*
x-forwarded-proto: http
x-request-id: 571c0fd6-98d4-4c93-af79-6a2fe2945847
x-envoy-decorator-operation: httpbin.default.svc.cluster.local:8000/*
x-b3-traceid: 82f3e0a76dcebca2
x-b3-spanid: 82f3e0a76dcebca2
x-b3-sampled: 0
x-istio-attributes: Cj8KGGRlc3RpbmF0aW9uLnNlcnZpY2UuaG9zdBIjEiFodHRwYmluLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwKPQoXZGVzdGluYXRpb24uc2VydmljZS51aWQSIhIgaXN0aW86Ly9kZWZhdWx0L3NlcnZpY2VzL2h0dHBiaW4KKgodZGVzdGluYXRpb24uc2VydmljZS5uYW1lc3BhY2USCRIHZGVmYXVsdAolChhkZXN0aW5hdGlvbi5zZXJ2aWNlLm5hbWUSCRIHaHR0cGJpbgo6Cgpzb3VyY2UudWlkEiwSKmt1YmVybmV0ZXM6Ly9zbGVlcC03YjlmOGJmY2QtMmRqeDUuZGVmYXVsdAo6ChNkZXN0aW5hdGlvbi5zZXJ2aWNlEiMSIWh0dHBiaW4uZGVmYXVsdC5zdmMuY2x1c3Rlci5sb2NhbA==
content-length: 0


05:47:50.159609 IP sleep-7b9f8bfcd-2djx5.49560 > 10-233-71-7.httpbin.default.svc.cluster.local.80: Flags [P.], seq 296287713:296288571, ack 4029574162, win 254, options [nop,nop,TS val 77427918 ecr 76732809], length 858: HTTP: GET /headers HTTP/1.1
E.....X.X...
.K.
.G....P......l......e.....
..t.....GET /headers HTTP/1.1
host: httpbin-shadow:8000
user-agent: curl/7.35.0
accept: */*
x-forwarded-proto: http
x-request-id: 571c0fd6-98d4-4c93-af79-6a2fe2945847
x-envoy-decorator-operation: httpbin.default.svc.cluster.local:8000/*
x-b3-traceid: 82f3e0a76dcebca2
x-b3-spanid: 82f3e0a76dcebca2
x-b3-sampled: 0
x-istio-attributes: Cj8KGGRlc3RpbmF0aW9uLnNlcnZpY2UuaG9zdBIjEiFodHRwYmluLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwKPQoXZGVzdGluYXRpb24uc2VydmljZS51aWQSIhIgaXN0aW86Ly9kZWZhdWx0L3NlcnZpY2VzL2h0dHBiaW4KKgodZGVzdGluYXRpb24uc2VydmljZS5uYW1lc3BhY2USCRIHZGVmYXVsdAolChhkZXN0aW5hdGlvbi5zZXJ2aWNlLm5hbWUSCRIHaHR0cGJpbgo6Cgpzb3VyY2UudWlkEiwSKmt1YmVybmV0ZXM6Ly9zbGVlcC03YjlmOGJmY2QtMmRqeDUuZGVmYXVsdAo6ChNkZXN0aW5hdGlvbi5zZXJ2aWNlEiMSIWh0dHBiaW4uZGVmYXVsdC5zdmMuY2x1c3Rlci5sb2NhbA==
x-envoy-internal: true
x-forwarded-for: 10.233.75.12
content-length: 0


05:47:50.166734 IP 10-233-75-11.httpbin.default.svc.cluster.local.80 > sleep-7b9f8bfcd-2djx5.38836: Flags [P.], seq 1:472, ack 796, win 276, options [nop,nop,TS val 77427925 ecr 77427918], length 471: HTTP: HTTP/1.1 200 OK
E....3X.?...
.K.
.K..P...$....ZH...........
..t...t.HTTP/1.1 200 OK
server: envoy
date: Fri, 15 Feb 2019 05:47:50 GMT
content-type: application/json
content-length: 241
access-control-allow-origin: *
access-control-allow-credentials: true
x-envoy-upstream-service-time: 3

{
  "headers": {
    "Accept": "*/*",
    "Content-Length": "0",
    "Host": "httpbin:8000",
    "User-Agent": "curl/7.35.0",
    "X-B3-Sampled": "0",
    "X-B3-Spanid": "82f3e0a76dcebca2",
    "X-B3-Traceid": "82f3e0a76dcebca2"
  }
}

05:47:50.166789 IP sleep-7b9f8bfcd-2djx5.38836 > 10-233-75-11.httpbin.default.svc.cluster.local.80: Flags [.], ack 472, win 262, options [nop,nop,TS val 77427925 ecr 77427925], length 0
E..42.X.X.\.
.K.
.K....P..ZH.$.............
..t...t.
05:47:50.167234 IP 10-233-71-7.httpbin.default.svc.cluster.local.80 > sleep-7b9f8bfcd-2djx5.49560: Flags [P.], seq 1:512, ack 858, win 280, options [nop,nop,TS val 77429926 ecr 77427918], length 511: HTTP: HTTP/1.1 200 OK
E..3..X.>...
.G.
.K..P....l....;...........
..|...t.HTTP/1.1 200 OK
server: envoy
date: Fri, 15 Feb 2019 05:47:49 GMT
content-type: application/json
content-length: 281
access-control-allow-origin: *
access-control-allow-credentials: true
x-envoy-upstream-service-time: 3

{
  "headers": {
    "Accept": "*/*",
    "Content-Length": "0",
    "Host": "httpbin-shadow:8000",
    "User-Agent": "curl/7.35.0",
    "X-B3-Sampled": "0",
    "X-B3-Spanid": "82f3e0a76dcebca2",
    "X-B3-Traceid": "82f3e0a76dcebca2",
    "X-Envoy-Internal": "true"
  }
}

05:47:50.167253 IP sleep-7b9f8bfcd-2djx5.49560 > 10-233-71-7.httpbin.default.svc.cluster.local.80: Flags [.], ack 512, win 262, options [nop,nop,TS val 77427926 ecr 77429926], length 0
E..4..X.X...
.K.
.G....P...;..n............
..t...|.
```



您可以查看流量的请求和响应内容。

###### 清理

* 删除规则：

```
$ kubectl delete virtualservice httpbin
$ kubectl delete destinationrule httpbin
```

* 关闭 [httpbin](https://github.com/istio/istio/tree/release-1.2/samples/httpbin) 服务和客户端：

```
$ kubectl delete deploy httpbin-v1 httpbin-v2 sleep
$ kubectl delete svc httpbin
```



##### 边缘流量控制

###### 1.没有 TLS 的 Ingress gateway

[使用 HTTPS 保护网关](https://istio.io/zh/docs/tasks/traffic-management/secure-ingress/)任务描述了如何配置 HTTPS 入口访问 HTTP 服务。此示例介绍如何配置对 HTTPS 服务的入口访问，即配置 Ingress Gateway 以执行 SNI 直通，而不是终止 TLS 请求的传入。

用于此任务的示例 HTTPS 服务是一个简单的 [NGINX](https://www.nginx.com/) 服务器。 在以下步骤中，首先在 Kubernetes 集群中部署 NGINX 服务。 然后配置网关以通过主机 `nginx.example.com` 提供对此服务的入口访问。

* ###### 生成客户端和服务器证书和密钥

```shell

pushd mtls-go-example
./generate.sh nginx.example.com password
mkdir ../nginx.example.com && mv 1_root 2_intermediate 3_application 4_client ../nginx.example.com
popd

```

* 部署NGINX服务器

创建一个kubernetes secret保存服务器的证书

```shell
kubectl create secret tls nginx-server-certs --key nginx.example.com/3_application/private/nginx.example.com.key.pem --cert nginx.example.com/3_application/certs/nginx.example.com.cert.pem
```

创建NGINX服务器配置文件

```shell
cat <<EOF > ./nginx.conf
events {
}

http {
  log_format main '$remote_addr - $remote_user [$time_local]  $status '
  '"$request" $body_bytes_sent "$http_referer" '
  '"$http_user_agent" "$http_x_forwarded_for"';
  access_log /var/log/nginx/access.log main;
  error_log  /var/log/nginx/error.log;

  server {
    listen 443 ssl;

    root /usr/share/nginx/html;
    index index.html;

    server_name nginx.example.com;
    ssl_certificate /etc/nginx-server-certs/tls.crt;
    ssl_certificate_key /etc/nginx-server-certs/tls.key;
  }
}
EOF
```

创建 Kubernetes [ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/) 保持 NGINX 服务器的配置：

```shell
kubectl create configmap nginx-configmap --from-file=nginx.conf=./nginx.conf
```

部署NGINX服务器：

```shell
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  ports:
  - port: 443
    protocol: TCP
  selector:
    run: my-nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 1
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 443
        volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx
          readOnly: true
        - name: nginx-server-certs
          mountPath: /etc/nginx-server-certs
          readOnly: true
      volumes:
      - name: nginx-config
        configMap:
          name: nginx-configmap
      - name: nginx-server-certs
        secret:
          secretName: nginx-server-certs
EOF
```

要测试 NGINX 服务器是否已成功部署，请从其 sidecar 代理向服务器发送请求， 而不检查服务器的证书（使用 `curl` 的 `-k` 选项）。确保正确打印服务器的证书， 即 `common name` 等于 `nginx.example.com`

```shell
kubectl exec -it $(kubectl get pod  -l run=my-nginx -o jsonpath={.items..metadata.name}) -c istio-proxy -- curl -v -k --resolve nginx.example.com:443:127.0.0.1 https://nginx.example.com

```

* 配置 Ingress Gateway

为端口 443 定义一个带有 `server` 部分的 `Gateway` 。注意 `PASSTHROUGH` `tls` `mode` 指示网关按原样传递入口流量， 而不终止 TLS。

```shell
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: mygateway
spec:
  selector:
    istio: ingressgateway # 使用 istio 默认的 ingress gateway
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: PASSTHROUGH
    hosts:
    - nginx.example.com
EOF
```

配置通过 `Gateway` 进入的流量路由：

```shell
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: nginx
spec:
  hosts:
  - nginx.example.com
  gateways:
  - mygateway
  tls:
  - match:
    - port: 443
      sni_hosts:
      - nginx.example.com
    route:
    - destination:
        host: my-nginx
        port:
          number: 443
EOF
```

按照[确定入口IP和端口](https://istio.io/zh/docs/tasks/traffic-management/ingress/#确定入口-ip-和端口)中的说明定义 `SECURE_INGRESS_PORT` 和 `INGRESS_HOST` 环境变量。

从群集外部访问 NGINX 服务。请注意，服务器返回正确的证书并成功验证（打印 *SSL certificate verify ok* ）

```shell
curl -v --resolve nginx.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST --cacert nginx.example.com/2_intermediate/certs/ca-chain.cert.pem https://nginx.example.com:$SECURE_INGRESS_PORT

```

* 清理

1. 删除创建的 Kubernetes 资源：

   ```
   $ kubectl delete secret nginx-server-certs
   $ kubectl delete configmap nginx-configmap
   $ kubectl delete service my-nginx
   $ kubectl delete deployment my-nginx
   $ kubectl delete gateway mygateway
   $ kubectl delete virtualservice nginx
   ```

2. 删除包含证书的目录和用于生成证书的存储库：

   ```
   $ rm -rf nginx.example.com mtls-go-example
   ```

   

3. 删除此示例中使用的生成的配置文件：

   ```
   $ rm -f ./nginx.conf
   ```

   



* 

















###### 2.Egress 网关的 TLS 发起过程

###### 3.出口流量的 TLS

###### 4.配置 Egress gateway

###### 5.使用通配符主机配置 Egress 流量

###### 6.Egress TLS 流量中的 SNI 监控及策略

###### 7.连接到外部 HTTPS 代理

###### 8.使用 cert-manager 加密 Kubernetes Ingress



#### 安全

##### HTTP 服务的访问控制

Istio 采用基于角色的访问控制方式，本文内容涵盖了为 HTTP 设置访问控制的各个环节。在[认证概念](https://istio.io/zh/docs/concepts/security/)一文中提供了 Istio 安全方面的入门教程。















#### 遥测

##### 指标度量

###### 收集指标和日志

###### 获取 TCP 服务指标

###### 查询 Prometheus 的指标

###### 使用 Grafana 可视化指标度量



##### 分布式追踪

###### 概述

###### Jaeger

###### Zipkin

###### 使用 LightStep [𝑥]PM 进行分布式追踪





##### 日志

###### 获取 Envoy 访问日志

###### 收集日志

###### 使用 Fluentd 记录日志



##### 网格可视化

##### 遥测插件的远程访问



## 4.3 k8s 集群上安装istio

```shell
# 下载istio
curl -L https://git.io/getLatestIstio | ISTIO_VERSION=1.2.2 sh -

export PATH=$PWD/bin:$PATH

# 创建ns
kubectl create namespace istio-system

# 使用 kubectl apply 安装所有的 Istio CRD，命令执行之后，会隔一段时间才能被 Kubernetes API Server 收到：
helm template install/kubernetes/helm/istio-init --name istio-init --namespace istio-system | kubectl apply -f -

```

![image-20190804183420404](/Users/xuel/Library/Application Support/typora-user-images/image-20190804183420404.png)











# 相关链接

* [codelab](https://codelabs.developers.google.com/codelabs/cloud-hello-istio/#0)
* [在kubernetes集群上安装istio并测试bookinfo示例微服务](https://jimmysong.io/posts/deploy-bookinfo-microservices-on-kubernetes-with-istio/)
* [istio中文文档](https://istio.io/zh/)
* [istio 学习笔记](https://skyao.io/learning-istio/)
* https://www.katacoda.com/courses/istio/deploy-istio-on-kubernetes
* https://skyao.io/talk/201811-knative-redefine-serverless/
* https://yq.aliyun.com/articles/221687
* https://jimmysong.io/kubernetes-handbook/develop/minikube.html
* [深入解读 Service Mesh 背后的技术细节 by 刘超](https://www.cnblogs.com/163yun/p/8962278.html)
* [Istio流量管理实现机制深度解析 by 赵化冰](https://zhaohuabing.com/post/2018-09-25-istio-traffic-management-impl-intro/)
* [Service Mesh架构反思：数据平面和控制平面的界线该如何划定？by 敖小剑](https://skyao.io/post/201804-servicemesh-architecture-introspection/)
* [理解 Istio Service Mesh 中 Envoy 代理 Sidecar 注入及流量劫持 by 宋净超](https://jimmysong.io/posts/envoy-sidecar-injection-in-istio-service-mesh-deep-dive/)
* [Service Mesh 深度学习系列——Istio源码分析之pilot-agent模块分析 by 丁轶群](http://www.servicemesher.com/blog/istio-service-mesh-source-code-pilot-agent-deepin)
* https://qii404.me/2018/01/06/minukube.html
* https://mp.weixin.qq.com/s?__biz=MzI3MTI2NzkxMA==&mid=2247486226&idx=1&sn=9573a9ce3d1665da2440ac80a971d2ae&chksm=eac52a3bddb2a32d90a610b247800826152d47889a8ce29635857c19eb84e4f1cbe124a8291f&token=1276805921&lang=zh_CN#rd
* https://zhaohuabing.com/istio-practice/
* https://skyao.io/learning-istio/introduction/information.html
* https://github.com/servicemesher/istio-knowledge-map

