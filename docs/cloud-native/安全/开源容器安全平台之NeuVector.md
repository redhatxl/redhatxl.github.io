# 开源容器安全平台之NeuVector

# 一 背景

**NeuVector成为了业界首个端到端的开源容器安全平台，唯一为容器化工作负载提供企业级零信任安全的解决方案**。

NeuVector 是业界领先的安全和合规解决方案，已被全球知名企业广泛采用；其代码库的开源不仅使 NeuVector 成为开源社区的首选技术，还为受严格监管的客户（包括政企、金融）提供了更可靠的保证。

**NeuVector开源容器镜像可以安装在任何Kubernetes集群上**。

**支持包括红帽OpenShift、VMWare Tanzu、Google GKE、Amazon EKS、Microsoft Azure AKS等在内的众多企业级容器管理平台**。

NeuVector 将驱动 SUSE 旗舰 Kubernetes 管理平台——SUSE Rancher 的容器安全创新，此举将**有助于推动Kubernetes安全领域的重大生态系统革新**，此前这一领域通常由闭源的专有解决方案主导。

容器安全一直是企业构建和运行 Kubernetes 应用的关键需求，**NeuVector项目使Rancher用户能够满足整个应用生命周期中的主要安全场景要求，包括深入的网络可视化、检查和微隔离；漏洞检测、配置和合规管理；以及风险分析、威胁检测和事件响应**。NeuVector 项目将成为 Rancher 高级集群安全功能的基础 。

# 二 NeuVector简介

NeuVector 提供强大的端到端容器安全平台。这包括对容器、Pod 和主机的端到端漏洞扫描和完整的运行时保护，包括：

* CI/CD 漏洞管理和准入控制，使用 Jenkins 插件扫描镜像、扫描注册表并强制实施准入控制规则以将其部署到生产环境中。
* 违规保护，发现行为并创建基于白名单的策略来检测违反正常行为的行为。
* 威胁检测，检测容器上的常见应用程序攻击，例如 DDoS 和 DNS 攻击。
* DLP 和 WAF 传感器。检查网络流量以防止敏感数据丢失，并检测常见的 OWASP Top10 WAF 攻击。
* 运行时漏洞扫描。扫描注册表、镜像和正在运行的容器编排平台和主机以查找常见 (CVE) 以及特定于应用程序的漏洞。
* 合规与审计。自动运行 Docker Bench 测试和 Kubernetes CIS Benchmarks。
* 端点/主机安全。检测权限升级，监控主机和容器内的进程和文件活动，并监控容器文件系统的可疑活动。
* 多集群管理。从单个控制台监控和管理多个 Kubernetes 集群。

NeuVector 的其他特性包括隔离容器和通过 SYSLOG 和 webhooks 导出日志的能力，为调查启动数据包捕获，以及与 OpenShift RBACs、 LDAP、 Microsoft AD 和 SSO 与 SAML 的集成。注意: 隔离意味着所有网络通信都被阻塞。容器将保持并继续运行——只是没有任何网络连接。Kubernetes 不会启动一个容器来替换隔离的容器，因为 api-服务器仍然能够到达容器。

# 三 组件构成即部署模式

NeuVector 运行时容器安全方案包括四种类型安全容器：Controllers，Enforcers，Managers，Scanners。

其能够部署为一个 Allinone 的特殊容器，也能即将个功能组合在一个容器总，当然也可以在虚拟机或单个操作系统的裸机上面部署。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220119203818.png)

## 3.1 Controller

Controller 管理 NeuVector Enforcer 容器集群。它还为管理控制台提供 REST api。虽然典型的测试部署只有一个 Controller，但是建议在高可用性配置中使用多个 Controller。控制器是 Kubernetes Production 部署示例 yaml 中的默认控制器。

## 3.2 Enforce

Enforcer 是一个轻量级容器，用于强制执行安全策略。应该在每个节点(主机)上部署一个执行器，例如作为一个守护进程集。注意: 对于 Docker 本地(非 Kubernetes)部署，执行器容器和控制器不能部署在同一个节点上(下面的 All-in-One 情况除外)。

## 3.3 Manager

Manager 是一个无状态容器，它为用户提供 Web-UI（仅限 HTTPS）和 CLI 控制台以管理 NeuVector 安全解决方案。可以根据需要部署多个 Manager 容器。

## 3.4 Scanner

扫描器是一个容器，它执行对图像、容器和节点的漏洞和顺应性扫描。它通常作为一个副本部署，并可以扩大到所需的许多并行扫描仪，以提高扫描性能。Controller 以循环的方式将扫描作业分配给每个可用的扫描器，直到所有扫描完成。扫描仪还包含最新的 CVE 数据库，并由 NeuVector 定期更新。

# 四 架构

这是 NeuVector 的一般架构概述。未显示的是单独的扫描仪容器，它也可以作为独立的管道扫描仪运行。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220119204325.png)



# 五 部署

## 5.1 Helm部署

* 添加 repo

```shell
$ helm repo add neuvector https://neuvector.github.io/neuvector-helm/
$ helm search repo neuvector/core
```

* #### Kubernetes部署

```shell
kubectl create namespace neuvector
kubectl create serviceaccount neuvector -n neuvector
kubectl create secret docker-registry regsecret -n neuvector --docker-server=https://index.docker.io/v1/ --docker-username=1832990 --docker-password=WWW.51idc.com --docker-email=18329903316@163.com

helm install my-release --namespace neuvector neuvector/core  --set imagePullSecrets=regsecret

NAME: my-release
LAST DEPLOYED: Wed Jan 19 21:04:03 2022
NAMESPACE: neuvector
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Get the NeuVector URL by running these commands:
  NODE_PORT=$(kubectl get --namespace neuvector -o jsonpath="{.spec.ports[0].nodePort}" services neuvector-service-webui)
  NODE_IP=$(kubectl get nodes --namespace neuvector -o jsonpath="{.items[0].status.addresses[0].address}")
  echo https://$NODE_IP:$NODE_PORT
```



# 参考链接

* https://open-docs.neuvector.com/
* https://mp.weixin.qq.com/s/iUpDaokUKt4Uf3m9aNRRVQ
* https://github.com/neuvector/neuvector
* https://github.com/neuvector/manager