# 开源容器安全平台之NeuVector

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220304181513.png)

## 一 背景

容器安全一直是企业构建和运行 Kubernetes 应用的关键需求，NeuVector项目使Rancher用户能够满足整个应用生命周期中的主要安全场景要求，包括深入的网络可视化、检查和微隔离；漏洞检测、配置和合规管理；以及风险分析、威胁检测和事件响应。NeuVector 项目将成为 Rancher 高级集群安全功能的基础 。

NeuVector成为了业界首个端到端的开源容器安全平台，唯一为容器化工作负载提供企业级零信任安全的解决方案。

NeuVector 是业界领先的安全和合规解决方案，已被全球知名企业广泛采用；其代码库的开源不仅使 NeuVector 成为开源社区的首选技术，还为受严格监管的客户（包括政企、金融）提供了更可靠的保证。

NeuVector开源容器镜像可以安装在任何Kubernetes集群上。

支持包括红帽OpenShift、VMWare Tanzu、Google GKE、Amazon EKS、Microsoft Azure AKS等在内的众多企业级容器管理平台。

NeuVector 将驱动 SUSE 旗舰 Kubernetes 管理平台——SUSE Rancher 的容器安全创新，此举将有助于推动Kubernetes安全领域的重大生态系统革新，此前这一领域通常由闭源的专有解决方案主导。

## 二 NeuVector简介

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220119204325.png)

### 2.1 功能特性

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

### 2.2 全生命周期安全

* 构建:镜像构建扫描,避免生成有隐患的镜像
* 部署:通过准入控制策略机制,避免有隐患的镜像和不符合策略要求的容器部署到环境
* 运行:四层/七层防火墙避免外部攻击和数据窃取
* 运行:东西向网络动态微隔离,避免内部攻击扩展。WAF防火墙避免外部攻击。
* 运行:容器内病毒、木马、破解器防护.
* 主机、Runtime、 K8S级别安全基线扫描，合规性评估。

### 2.3 优势

* 开放性: 100%开源,无需担心供应商锁定。
* 灵活性:灵活部署各类Kubernetes发行版, Rancher、Openshift、EKS、 GKE、ACK、 TKE。
* 可靠性: 7年迭代,成熟稳定产品。
* 专业性:专业支持服务,保障业务安全可靠持续运行。

## 三 组件构成即部署模式

NeuVector 运行时容器安全方案包括四种类型安全容器：Controllers，Enforcers，Managers，Scanners。

其能够部署为一个 Allinone 的特殊容器，也能即将个功能组合在一个容器总，当然也可以在虚拟机或单个操作系统的裸机上面部署。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220119203818.png)

### 3.1 Controller

Controller 管理 NeuVector Enforcer 容器集群。它还为管理控制台提供 REST api。虽然典型的测试部署只有一个 Controller，但是建议在高可用性配置中使用多个 Controller。控制器是 Kubernetes Production 部署示例 yaml 中的默认控制器。

### 3.2 Enforce

Enforcer 是一个轻量级容器，用于强制执行安全策略。应该在每个节点(主机)上部署一个执行器，例如作为一个守护进程集。注意: 对于 Docker 本地(非 Kubernetes)部署，执行器容器和控制器不能部署在同一个节点上(下面的 All-in-One 情况除外)。

### 3.3 Manager

Manager 是一个无状态容器，它为用户提供 Web-UI（仅限 HTTPS）和 CLI 控制台以管理 NeuVector 安全解决方案。可以根据需要部署多个 Manager 容器。

### 3.4 Scanner

扫描器是一个容器，它执行对图像、容器和节点的漏洞和顺应性扫描。它通常作为一个副本部署，并可以扩大到所需的许多并行扫描仪，以提高扫描性能。Controller 以循环的方式将扫描作业分配给每个可用的扫描器，直到所有扫描完成。扫描仪还包含最新的 CVE 数据库，并由 NeuVector 定期更新。

## 四 平台功能

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220304182557.png)

NeuVector提供操作系统/Runtime/K8s/容器应用三个层面安全业务进行保护。

## 五 部署

### 5.1 Helm部署

* 添加 repo

```shell
helm repo add neuvector https://neuvector.github.io/neuvector-helm/
helm search repo neuvector/core
```

* #### Kubernetes部署

```shell
kubectl create namespace neuvector
kubectl create serviceaccount neuvector -n neuvector
helm install neuvector --namespace neuvector neuvector/core  --set registry=docker.io  --set
tag=5.0.0-preview.1 --set=controller.image.repository=neuvector/controller.preview --
set=enforcer.image.repository=neuvector/enforcer.preview --set 
manager.image.repository=neuvector/manager.preview --set 
cve.scanner.image.repository=neuvector/scanner.preview --set cve.updater.image.repository=neuvector/updater.preview


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

Helm-chart 参数查看：

https://github.com/neuvector/neuvector-helm/tree/master/charts/core

### 5.2 资源清单部署

#### 5.2.1 安装环境

```shell
软件版本：
Kubernetes：1.20.14
Docker：19.03.15
NeuVector：5.0.0-preview.1
```

#### 5.2.2 开始执行安装

* 创建 namespace

```
kubectl create namespace neuvector
```

* 部署 CRD( Kubernetes 1.19+ 版本)

```
kubectl apply -f https://raw.githubusercontent.com/neuvector/manifests/main/kubernetes/crd-k8s-1.19.yaml
```

* 部署 CRD(Kubernetes 1.18或更低版本)

```
kubectl apply -f https://raw.githubusercontent.com/neuvector/manifests/main/kubernetes/crd-k8s-1.16.yaml
```

* 配置 RBAC

```
kubectl create clusterrole neuvector-binding-app --verb=get,list,watch,update --resource=nodes,pods,services,namespaces
kubectl create clusterrole neuvector-binding-rbac --verb=get,list,watch --resource=rolebindings.rbac.authorization.k8s.io,roles.rbac.authorization.k8s.io,clusterrolebindings.rbac.authorization.k8s.io,clusterroles.rbac.authorization.k8s.io
kubectl create clusterrolebinding neuvector-binding-app --clusterrole=neuvector-binding-app --serviceaccount=neuvector:default
kubectl create clusterrolebinding neuvector-binding-rbac --clusterrole=neuvector-binding-rbac --serviceaccount=neuvector:default
kubectl create clusterrole neuvector-binding-admission --verb=get,list,watch,create,update,delete --resource=validatingwebhookconfigurations,mutatingwebhookconfigurations
kubectl create clusterrolebinding neuvector-binding-admission --clusterrole=neuvector-binding-admission --serviceaccount=neuvector:default
kubectl create clusterrole neuvector-binding-customresourcedefinition --verb=watch,create,get --resource=customresourcedefinitions
kubectl create clusterrolebinding  neuvector-binding-customresourcedefinition --clusterrole=neuvector-binding-customresourcedefinition --serviceaccount=neuvector:default
kubectl create clusterrole neuvector-binding-nvsecurityrules --verb=list,delete --resource=nvsecurityrules,nvclustersecurityrules
kubectl create clusterrolebinding neuvector-binding-nvsecurityrules --clusterrole=neuvector-binding-nvsecurityrules --serviceaccount=neuvector:default
kubectl create clusterrolebinding neuvector-binding-view --clusterrole=view --serviceaccount=neuvector:default
kubectl create rolebinding neuvector-admin --clusterrole=admin --serviceaccount=neuvector:default -n neuvector
```

* 检查是否有以下 RBAC 对象

```
kubectl get clusterrolebinding  | grep neuvectorkubectl get rolebinding -n neuvector | grep neuvector
kubectl get clusterrolebinding  | grep neuvector
neuvector-binding-admission                            ClusterRole/neuvector-binding-admission                            44hneuvector-binding-app                                  ClusterRole/neuvector-binding-app                                  44hneuvector-binding-customresourcedefinition             ClusterRole/neuvector-binding-customresourcedefinition             44hneuvector-binding-nvadmissioncontrolsecurityrules      ClusterRole/neuvector-binding-nvadmissioncontrolsecurityrules      44hneuvector-binding-nvsecurityrules                      ClusterRole/neuvector-binding-nvsecurityrules                      44hneuvector-binding-nvwafsecurityrules                   ClusterRole/neuvector-binding-nvwafsecurityrules                   44hneuvector-binding-rbac                                 ClusterRole/neuvector-binding-rbac                                 44hneuvector-binding-view                                 ClusterRole/view                                                   44h
```

```
kubectl get rolebinding -n neuvector | grep neuvectorneuvector-admin         ClusterRole/admin            44hneuvector-binding-psp   Role/neuvector-binding-psp   44h
```

* 部署 NeuVector
  * 底层 Runtime 为 Docker

```
kubectl apply -f https://raw.githubusercontent.com/neuvector/manifests/main/kubernetes/5.0.0/neuvector-docker-k8s.yaml
```

底层 Runtime 为 Containerd（对于 k3s 和 rke2 可以使用此 yaml 文件）

```
kubectl apply -f https://raw.githubusercontent.com/neuvector/manifests/main/kubernetes/5.0.0/neuvector-containerd-k8s.yaml
```

1.21 以下的 Kubernetes 版本会提示以下错误，将 yaml 文件下载将 batch/v1 修改为 batch/v1beta1

```
error: unable to recognize "https://raw.githubusercontent.com/neuvector/manifests/main/kubernetes/5.0.0/neuvector-docker-k8s.yaml": no matches for kind "CronJob" in version "batch/v1"
```

1.20.x cronjob 还处于 beta 阶段，1.21 版本开始 cronjob 才正式 GA 。

默认部署web-ui使用的是loadblance类型的Service，为了方便访问修改为NodePort，也可以通过 Ingress 对外提供服务

```
kubectl patch  svc neuvector-service-webui  -n neuvector --type='json' -p '[{"op":"replace","path":"/spec/type","value":"NodePort"},{"op":"add","path":"/spec/ports/0/nodePort","value":30888}]'
```

访问 https://node_ip:30888

默认密码为 admin/admin

#### 5.2.3 访问

由于我采用minikube部署，临时菜哟哦那个port-forward访问测试。

```shel
kubectl port-forward --address 0.0.0.0 -n neuvector service/neuvector-service-webui 22222:8443
```

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220304204541.png)

## 六 功能测试

### 6.1 Dashboard

在neuvector的dashboard页面，除了有集群的健康体检，还有入口和出口暴露，已经TOP安全事件/资源/策略等。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220304211008.png)

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220304205144.png)

### 6.2 Network Activity



![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220304205913.png)

### 6.3 资产

#### 6.3.1 平台

在平台模块，可以看到当前集群运行的K8s版本，及当前集群中存在的CVE漏洞及详细漏洞信息。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220306132518.png)

#### 6.3.2 主机

在改模块可以看到运行K8s集群的Node节点资源信息，保护操作系统及资源软件版本等信息。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220306132704.png)

* 合规信息

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220306132741.png)

* 漏洞信息

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220306132802.png)

* 容器信息

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220306132908.png)

#### 6.3.3 容器

该模块可以看到集群运行的所有容器，并可以对单个容器进行查询详细信息，漏洞信息，容器进程，容器实时监控等。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220306133152.png)

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220306133056.png)

#### 6.3.4 镜像库

镜像库可以添加公有云镜像，或自建镜像，对其中存储的镜像进行安全扫描。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220306133536.png)

#### 6.3.5 系统组件

该模块是对NeeuVector的系统自己例如控制器/扫描器/代理器进行信息展示和监控统计。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220306133402.png)

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220306133455.png)

### 6.4 策略

* 准入控制

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220306133812.png)

* 策略了

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220306133857.png)



### 6.5 安全隐患

* 漏洞

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220306134000.png)

* 合规

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220306134026.png)

### 6.7 通知/设置

* 风险报告

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220306134101.png)

* 事件

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220306134122.png)

## 其他

* NeuVector支持多集群管理。
* 支持丰富的策略及报告导出。

## 参考链接

* https://open-docs.neuvector.com/
* https://mp.weixin.qq.com/s/iUpDaokUKt4Uf3m9aNRRVQ
* https://github.com/neuvector/neuvector
* https://github.com/neuvector/manager
* https://mp.weixin.qq.com/s/kOGNT2L2HVMibyyM6Ri2KQ