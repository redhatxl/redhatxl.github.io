# Kubernetes安全分析



## Kubernetes 大体可以分为两类

* Node 安全
* 应用安全

Kubernetes 的安全工具多得很，有不同的功能、范围以及授权方式。因此我们建立了这个 Kubernetes 安全工具列表，其中有来自不同厂商的开源项目和商业平台，读者可以根据兴趣和需要进行了解和选择。

## **Kubernetes 安全工具——分类**

为了方便读者浏览目录，我们把这些工具按照主要功能和范围进行了分类：

- Kubernetes 镜像扫描和静态分析
- Kubernetes 运行时安全
- Kubernetes 网络安全
- 镜像分发和机密管理
- Kubernetes 安全审计
- 端到端的 Kubernetes 安全商业产品

我们最爱的容器编排平台已经成熟，会有越来越多的 Kubernetes 安全工具涌现出来，如果读者发现我们列表的错漏，请在 Twitter 上联系 @sysdig。

言归正传。

## **Kubernetes 镜像扫描**

### **Anchore**

主页：https://anchore.com

许可：免费（Apache）以及商业产品

Anchore 引擎不但能够对容器镜像进行分析，更可以使用用户自定义的策略来完成自定义的安全检查。

除了利用 CVE [数据库](https://cloud.tencent.com/solution/database?from=10680)来对已知威胁进行扫描之外，Anchore 还提供了很多附加标准可以进行配置，来作为扫描策略的一部分：Dockerfile 检查、凭据泄露、语言相关内容（mpm、maven 等）、软件许可等。

### **Clair**

主页：https://coreos.com/clair

许可：免费（Apache）

Clair 是最早开源的镜像扫描项目之一，也是 Quay 镜像库的安全扫描引擎。Clair 能从很多数据源中拉取 CVE 信息，其中包括来自 Debian、RedHat 或者 Ubuntu 安全团队的特定发行版的威胁列表。

和 Anchore 不同的是，Clair 专注于威胁检测和 CVE 匹配的功能，也提供了一定的扩展性，让用户通过实现可插接驱动来实现扩展。

### **Dagda**

主页：https://github.com/eliasgranderubio/dagda

许可：免费（Apache）

Dagda 会针对容器镜像中已知的漏洞、特洛伊、病毒、恶意软件和其它恶意威胁进行静态分析。

和其它的 Kubernetes 安全工具相比，Dagda 有两个与众不同之处：

- 原生集成了 ClamAV，不仅可以扫描镜像，还能用作防毒软件。
- Dagda 还提供了运行时保护功能。从 Docker 守护进程实时收集事件，并和 Falco 集成识别安全事件。

### **KubeXray**

主页：https://github.com/jfrog/kubexray

许可：免费（Apache），但是需要从 JFrog Xray（商业产品）获取数据。

KubeXray 监听 Kubernetes API Server 的事件，并利用 JFrog Xray（商业产品）的元数据来确认只有符合策略要求的 Pod 才能运行。

KubeXray 不只会对新建或者更新的容器部署进行审计（Kuberentes 准入控制就是这样），还能动态的根据新的安全策略对运行中的容器进行检查，并删除有漏洞的镜像所对应的资源。

### **Snyk**

主页：https://snyk.io/

许可：免费（Apache）以及商业产品

Snyk 是一个特别的漏洞[检测工具](https://cloud.tencent.com/product/tools?from=10680)，其特点是着眼于开发工作流，自称是开发第一的解决方案。

Snyk 会直接链接到代码仓库，解析项目结构，并分析引入的代码及其直接和间接依赖。Snyk 支持很多流行的编程语言，还能发现潜在的许可风险。

### **Trivy**

主页：https://github.com/knqyf263/trivy

许可：免费（AGPL）

Trivy 是个简单全面的容器漏洞检测工具，能够方便的和 CI/CD 进行集成。它的安装和操作都很简单，只需要一个二进制文件，无需安装数据库和其它的附加内容。

Trivy 的简便性的一个缺点是需要学习如何解析和转发它的 JSON 输出，这才能方便其它工具进行调用。

## **Kubernetes 运行时安全**

### **Falco**

主页：https://falco.org/

许可：免费（Apache）

Falco 是一个云原生的运行时安全工具，CNCF 成员项目。

利用 Sysdig 的 Linux 内核指令和系统调用分析，Falco 能够深入理解系统行为。它的运行时规则引擎能够检测应用、容器、主机以及 Kubernetes 的反常行为。

凭借 Falco，在每个 Kubernetes 节点部署一个代理，无需修改或者注入第三方代码或者加入 Sidecar 容器，就能够得到完整的运行时可见性以及威胁检测。

### **Linux 运行时安全框架**

原生的 Linux 框架其实不能算作是“Kubernetes 安全工具”，但它们的运行时安全上下文是可以包括在 Kubernetes 的 Pod 安全策略之中的（PSP），所以还是值得一提。

AppArmor 为容器内的进程附加一个安全档案，其中定义了文件系统、权限、网络访问规则、库链接等。这是一个访问控制系统，会阻止未经授权的动作发生。

SELinux 是一个 Linux 内核安全模块，和 AppArmor 有点相似，常常被拉来做比较。SELinux 更加强大，粒度更细，也比 AppArmor 更有弹性，学习曲线更加陡峭、也更加复杂。

Seccomp 和 seccomp-bpf 允许对系统调用进行过滤，可以防止用户的二进制文对主机操作系统件执行通常情况下并不需要的危险操作。它和 Falco 有些类似，不过 Seccomp 没有为容器提供特别的支持。

### **开源版 Sysdig**

主页：https://www.sysdig.com/opensource

许可：免费（Apache）

Sysdig 是一个全面的 Linux 系统（在 Windows 和 Mac OSX 下也提供了有限支持）下的观察、排错和调试工具。可以用来对主机操作系统以及运行其上的容器进行详细的监控和观察。

Sysdig 还对容器运行时以及 Kubernetes 元数据提供了原生支持，能在收集到的系统活动数据中加入额外的维度和标签。Sysdig 提供了很多方式来探索 Kubernetes 集群：可以使用 kubectl capture 创建一个是检点的快照，或者使用 kubectl dig 来进行交互访问。

## **Kubernetes 网络安全**

### **Aporeto**

主页：https://www.aporeto.com/

许可：商业

Aporeto 提供了“从网络和基础设施中解耦的安全性”。这意味着你的 Kubernetes 服务不只是获得了一个本地 ID（也就是 Kubernetes ServiceAccount），还有一个全局 ID/指纹，可以以此为基础和任何其它服务进行安全和双向校验的通信。

Aporeto 生成的唯一身份，不仅可以提供给 Kubernetes 或者容器，还能提供给主机、云函数和用户使用，根据这些身份和网络安全策略的配置，可以选择性的对通信进行放行或者阻断。

### **Calico**

主页：https://www.projectcalico.org/

许可：免费（Apache）

Calico 经常随容器编排系统一同部署，用于实现容器之间的虚拟网络。在基础的网络功能之外，Calico 项目还实现了 Kubernetes 网络策略规范，以及自己的一套安全策略，其中包括了端点的 ACL 和基于注解的入栈/出栈网络安全规则。

### **Cilium**

主页：https://www.cilium.io/

许可：免费（Apache）

Cilium 提供了容器防火墙以及网络安全功能，适用于 Kubernetes 和微服务负载。Cilium 依赖于一种新的 Linux 内核技术——BPF，用它来执行核心数据路径的过滤、监控、重整、重定向等功能。

Cilium 能够根据容器身份（Docker 或者 Kubernetes 标签和元数据）进行网络访问策略的定义。Cilium 还能理解并过滤多种 HTTP、gRPC 这样的 L7 协议（例如可以设置两个 Kubernetes 部署之间 REST API 的访问性）。

### **Istio**

主页：https://istio.io/

许可：免费（Apache）

Istio 是广为人知的服务网格产品，它通过部署平台无关的控制平面，并把所有托管服务流量重新路由到动态配置的 Envoy Proxy 上完成网格功能。Istio 占据了通信的主动权，能够为微服务和容器实现多种网络安全策略。

Istio 网络安全能力包括：透明的 TLS 加密，能够自动把微服务通信升级为 HTTPS，并且它具备 RBAC 以及鉴权能力，可以在集群中不同工作负载之间进行通信时进行接受或者拒绝的决策。

### **Tigera**

主页：https://www.tigera.io/

许可：商业

Tigera 的“Kubernetes 防火墙”技术，以零信任网络的理念来加固 Kubernetes 的网络安全。

与其它 Kubernetes 原生网络解决方案类似，Tigera 利用 Kubernetes 元数据来识别集群中的不同服务和实体，提供跨多云或混合单体-容器基础设施运行时检测、持续合规性检查和网络监控能力。

### **Trireme**

主页：https://www.aporeto.com/opensource/

许可：免费（Apache）

Trireme 是一个简单直接的 Kubernetes 网络策略规范的实现。它有一个与众不同的特点：无需一个中心控制平面来对网格进行协调，因此这个方案具备很好的伸缩能力。Trireme 通过在每个节点上安装代理的方式来影响主机的 TCP/IP 网络栈。

## **镜像分发和机密管理**

### **Grafeas**

主页：https://grafeas.io/

许可：免费（Apache）

Grafeas 是一个开源的 API，用于对软件供应链进行审计和监管。泛泛而论，Grafeas 是一个元数据和审计日志收集工具，可以用来跟踪组织中的安全合规实践。

这种集中起来的信息可以帮助用户回答类似这样的安全问题：

- 某个容器是谁构建并签名的？
- 所有的安全扫描和策略检查都通过了么？什么时候的事？这些工具都输出了什么信息？
- 谁把它部署到生产环境的？用什么参数部署的？

### **Portieris**

主页：https://github.com/IBM/portieris

许可：免费（Apache）

Portieris 是一个 Kubernetes 准入控制器，可以用于内容信任。它依赖 Notary 服务器以此作为信任源头并签署工件。

一旦修改了 Kubernetes 的工作负载，Portieris 就会为请求的容器镜像拉取签名信息和内容信任策略，如果需要的话，还可以修改 API 对象的内容，并以签署版本的镜像来进行替换。

### **Vault**

主页：https://www.vaultproject.io/

许可：免费（MPL）

Vault 是一个用于存储机密（例如密码、Token、Secret 等）的高度安全的存储方案，它支持很多高级功能，例如临时安全令牌或者受编排的密钥翻转。

可以用 Helm 在 Kubernetes 集群中部署 Vault 的 Chart，使用 Consul 作为存储后端。它支持 Kubernetes 的本地资源，比如说 ServiceAccount Token，甚至还能作为缺省的 Kubernetes Secret 仓库。

## **Kubernetes 安全审计**

### **Kube-bench**

主页：https://github.com/aquasecurity/kube-bench

许可：免费（Apache）

Kube-Bench 是一个 Go 应用，它会运行 CIS Kubernetes 基准测试中的测试，来检查 Kubernetes 部署的安全程度。

Kube-Bench 会扫描你的 Kubernetes 集群组件（ETCD、API、Controller Manager 等）、敏感文件授权、不安全的帐号或者开放端口、资源配额、API 速率限制等方面查找不安全的配置参数。

### **Kube-Hunter**

主页：https://github.com/aquasecurity/kube-hunter

许可：免费（Apache）

Kube-Hunter 在 Kubernetes 集群中查找安全弱点（例如远程代码执行或者信息泄露）。可以把 Kube-Hunter 作为一个远程扫描器，来从外部攻击者的视角来观察你的集群；也可以用 Pod 的方式来运行。

Kube-Hunter 有个特别之处就是“active hunting”，它不仅会报告问题，而且还会尝试利用在 Kubernetes 集群中发现的问题，这种操作可能对集群有害，应小心使用。

### **Kubeaudit**

主页：https://github.com/Shopify/kubeaudit

许可：免费（MIT）

Kubeaudit 是一个免费的[命令行工具](https://cloud.tencent.com/product/cli?from=10680)，由 Shopify 提供，用于对 Kubernetes 的配置进行多方面的审计。其中包含无限制的镜像使用、以 Root 身份运行、特权运行以及缺省的 ServiceAccount 等。

Kubeaudit 由很多其它有趣的功能，例如扫描处理本地的 YAML 来查找配置缺陷和安全问题，并且自动修复。

### **Kubesec**

主页：https://kubesec.io/

许可：免费（Apache）

Kubesec 是一个比较特别的 Kubernetes 安全工具，它会直接对 YAML 进行扫描，查找其中描述的 Kubernetes 资源是否使用了较弱的安全参数。

例如它可以检测到 Pod 的过高权限，使用 root 作为默认用户、附加到主机网络命名空间、危险的加载操作（例如 `/proc` 或者 Docker Socket）。它还提供了一个在线的演示，可以在上面提交 YAML 体验这一功能。

### **Open Policy Agent**

主页：https://www.openpolicyagent.org/

许可：免费（Apache）

OPA 的目标是把安全策略和最佳实践从特定的运行时平台（Docker、Kubernetes、Mesosphere、Openshift 等）中解耦出来。

例如可以把 OPA 作为 Kubernetes 准入控制器后端进行部署，这样 OPA 代理就可以接管安全决策，根据自定义安全约束，对请求进行校验、拒绝甚至是就地修改。OPA 使用一种自有的 DSL（Rego）编写策略。

## **端到端的 Kubernetes 安全商业产品**

我们决定创建一个单独的分类来介绍商业产品，这是因为它们经常会覆盖安全工作的多个方面。下表做了一些简要对比。

|                            | 镜像扫描 | 容器合规 | 运行时安全 | 网络安全 | Forensics | Kubernetes 审计 |
| :------------------------- | :------- | :------- | :--------- | :------- | :-------- | :-------------- |
| AquaSec                    | Y        | Y        | Y          | Y        | Y         | Y               |
| Capsule8                   |          |          | Y          | Y        | Y         | Y               |
| Caviring                   | Y        | Y        | Y          |          |           | Y               |
| Google SCC                 | Y        |          | Y          | 插件     | Y         |                 |
| Layered Insight            | Y        | Y        | Y          | Y        |           |                 |
| NeuVector                  | Y        | Y        | Y          | Y        | Y         | Y               |
| StackRox                   | Y        | Y        | Y          | Y        | Y         | Y               |
| Sysdig Secure              | Y        | Y        | Y          | Y        | Y         | Y               |
| Tenable Container security | Y        | Y        | Y          |          |           |                 |
| Twistlock                  | Y        | Y        | Y          | Y        | Y         | Y               |

### **Aqua Security**

主页：https://www.aquasec.com/

许可：商业

AquaSec 是一个针对容器和云负载的商业安全工具，包括：

- 能够集成到容器仓库或者 CICD 的镜像扫描。
- 能够检测容器修改或异常行为的运行时保护。
- 容器原生的应用程序防火墙。
- 针对云服务的 Serverless 安全。
- 集成到事件日志的合规和审计报告。

### **Capsule 8**

主页：https://capsule8.com/

许可：商业

Capsule 8 在你的自建或云端 Kubernetes 集群中部署探针，从而集成到基础设施之中。这个探针会搜集主机和网络指标，通过这些数据和攻击行为模式进行匹配。

Capsule 8 团队负责在 0 day 攻击到达你的集群之前进行检测和阻止。他们的安全团队能够将安全规则推送到探针上，从而阻止软件威胁。

### **Cavirin**

主页：https://www.cavirin.com/

许可：商业

Cavirin 专注于为不同的安全标准化机构提供企业版本。它的镜像扫描功能，还可以与 CI/CD 管道进行集成，在将不合规的镜像推送到镜像库之前阻止它们。

Cavirin 安全套件使用机器学习为网络安全状态提供类似信用的评分，提供补救技巧，以改善安全状况或安全标准合规性。

### **Google Cloud Security Command Center**

主页：https://cloud.google.com/security-command-center/

许可：商业

Google SCC 能帮安全团队收集数据、识别威胁并在业务损失之前对其采取行动。

SCC 是一个统一的控制面板，在这里可以集成不同的安全报告、资产清单以及第三方安全引擎。

SCC 提供的 API 可以集成来自不同来源（Sysdig Secure 或者 Falco）的 Kubernetes 安全事件。

### **Layered Insight (Qualys)**

主页：https://layeredinsight.com/

许可：商业

Layered Insight（现在是 Qualys 的一部分）是围绕“嵌入式安全性”的概念设计的。它用静态分析技术扫描原有镜像漏洞并通过 CVE 检查后，Layered Insight 会注入一个二进制代理，生成一个中间镜像。

这个二进制代理包括容器网络流量、I/O 流以及应用程序活动的运行时安全性探测，还包括基础架构运营商或 DevOps 团队提供的自定义安全检查内容。

### **Neuverctor**

主页：https://neuvector.com/

许可：商业

NeuVector 通过分析网络活动和应用程序行为，为每个映像创建定制的安全配置文件，来执行容器安全基准和运行时保护。它还可以主动阻止威胁，通过修改本地网络防火墙来隔离可疑活动。

NeuVector 的网络集成，标记为“安全网格”，能够对服务网格中的所有网络连接执行数据包深度检查和 L7 过滤。

### **StackRox**

主页：https://www.stackrox.com/

许可：商业

StackRox 容器安全平台的设计目标是涵盖 Kubernetes 集群中应用程序的整个生命周期。与此列表中的其它商业方案一样，它会根据观察到的容器行为生成运行时配置文件，并会在发现异常情况时自动发出警报。

StackRox 平台还将使用 CIS Kubernetes 基准以及其他容器合规性基准，对 Kubernetes 配置进行评估。

### **Sysdig Secure**

主页：https://sysdig.com/products/secure/

许可：商业

Sysdig Secure 在整个容器生命周期内对云原生应用程序实施保护。它把镜像扫描，运行时保护和取证结合在一起，以识别漏洞、阻止威胁，执行合规性并对微服务中的活动进行审计。

一些重要功能包括：

Scanning images in a registry or as part of the CI/CD process to uncover vulnerable libraries, packages, and configuration Run-time detection to protect containers in production with behavioral profiles Record pre- and post-attack activity through system calls with microsecond level granularity 250+ out of the box compliance checks to keep your configuration secure

- 在镜像库中，或作为 CI/CD 过程的一部分对镜像进行扫描，以发现易受攻击的库、包和配置内容。
- 运行时检测，使用行为配置文件来保护生产中的容器。
- 通过系统调用，在毫秒一级对攻击前后的行为进行记录。
- 开箱即用的超过 250 项合规性检查，帮助用户保持配置安全。

### **Tenable Container Security**

 主页：https://www.tenable.com/products/tenable-io/container-security

许可：商业

在容器问世之前，Tenable 在安全行业广为人知，它的 Nusus 是一款流行的漏洞扫描和安全审计工具。

Tenable Container security 利用他们在计算机安全领域的经验，将 CI/CD 与漏洞数据库、专门的恶意软件检测引擎和安全威胁补救建议集成在一起。

### **Twistlock (Palo Alto Networks)**

主页：https://www.twistlock.com/

许可：商业

Twistlock 自诩为云优先的、容器优先的平台，提供与云提供商（AWS，Azure，GCP）、容器编排器（Kubernetes，Mesospehere，Openshift，Docker），Serverless 运行时，网格框架和 CI/CD 工具的特定集成。

除了通常的容器安全企业功能，如 CI/CD 管道集成或镜像扫描，Twistlock 使用机器学习技术来生成行为模式和容器感知网络规则。

Twistlock 被 Palo Alto Networks 收购，Palo Alto Networks 也是 Evident.io 和 Redlock 安全解决方案的所有者。期待这三个平台合而为一，整合到 Palo Alto 的 PRISMA 中。

## 参考链接

* https://zhuanlan.zhihu.com/p/446569931

* https://www.redhat.com/en/resources/kubernetes-adoption-security-market-trends-2021-overview

* https://kubernetes.io/docs/concepts/policy/pod-security-policy/
* https://github.com/stackrox/kube-linter
* https://github.com/aquasecurity/kube-hunter

* https://cloud.tencent.com/developer/article/1468809?from=article.detail.1741460