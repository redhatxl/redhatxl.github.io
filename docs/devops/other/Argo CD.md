## GitOps工具Argo CD实战

# 背景

GitOps 的概念最初来源于 [Weaveworks](https://links.jianshu.com/go?to=https%3A%2F%2Fyq.aliyun.com%2Fgo%2FarticleRenderRedirect%3Furl%3Dhttps%3A%2F%2Fwww.weave.works%2F) 的联合创始人 Alexis 在 2017 年 8 月发表的一篇博客 [GitOps - Operations by Pull Request](https://links.jianshu.com/go?to=https%3A%2F%2Fyq.aliyun.com%2Fgo%2FarticleRenderRedirect%3Furl%3Dhttps%3A%2F%2Fwww.weave.works%2Fblog%2Fgitops-operations-by-pull-request)。其是开发者和运维在集群上线和更新在 Kubernetes 运行的复杂应用程序的一种快速、安全的方法。Alexis的文章文章介绍了 Weaveworks 的工程师如何以 Git 作为事实的唯一真实来源，部署、管理和监控基于 Kubernetes 的 SaaS 应用。

核心：

* 对于K8s和其他云原生技术GitOps可谓一个操作模型，提供了一组最佳实践，对于容器集群和应用的统一部署，管理和监控。

* 端到端的 CICD 管道和 git 工作流都应用于操作和开发通向管理应用程序的开发人员体验之路; 

近来研究GitOps工具Arog，一文走入GitOps之门，体验为开发人员带来的将上线部署结合在Git中的畅快体验，git commit/reserve的上线发布与回滚变的如此简单。

一 概述

## 1.1 什么时候Argo CD

argo CD对k8s是一个声明式的，gitops的持续集成工具。

## 1.2 为什么使用Argo CD

应用的定义/配置和环境应该被声明和版本管理，应用程序部署和生命周期管理应该是自动化的，可审核的且易于理解的。

## 1.3 使用

### 1.3.1 工作方式

Argo CD遵循GitOps模式，该模式使用Git存储库作为定义所需应用程序状态的真实来源。Kubernetes清单可以通过几种方式指定：

- [kustomize](https://kustomize.io/) applications
- [helm](https://helm.sh/) charts
- [ksonnet](https://ksonnet.io/) applications
- [jsonnet](https://jsonnet.org/) files
- Plain directory of YAML/json manifests
- Any custom config management tool configured as a config management plugin

Argo CD可在指定的目标环境中自动部署所需的应用程序状态。应用程序部署可以在Git提交时跟踪对分支，标签的更新，或固定到清单的特定版本。有关可用的不同跟踪策略的更多详细信息，请参见跟踪策略。

### 1.3.2 架构

![image-20200124102542457](/Users/xuel/Library/Application Support/typora-user-images/image-20200124102542457.png)





Argo CD被实现为kubernetes控制器，该控制器连续监视正在运行的应用程序，并将当前的活动状态与所需的目标状态（在Git存储库中指定）进行比较。处于活动状态偏离目标状态的已部署应用程序被视为OutOfSync。Argo CD报告并可视化差异，同时提供了自动或手动将实时状态同步回所需目标状态的功能。在Git存储库中对所需目标状态所做的任何修改都可以自动应用并反映在指定的目标环境中。

# 二 基础概念

在有效使用Argo CD之前，有必要了解该平台所基于的基础技术。还需要了解向您提供的功能以及如何使用它们。以下部分提供了一些有用的链接，可以帮助您理解。

## 2.1 了解基础

### 2.1.1 k8s知识点

- [A Beginner-Friendly Introduction to Containers, VMs and Docker](https://medium.freecodecamp.org/a-beginner-friendly-introduction-to-containers-vms-and-docker-79a9e3e119b)
- [Introduction to Kubernetes](https://courses.edx.org/courses/course-v1:LinuxFoundationX+LFS158x+2T2017/course/)
- [Tutorials](https://kubernetes.io/docs/tutorials/)
- [Hands on labs](https://katacoda.com/courses/kubernetes/)

### 2.1.2 自定义模版

- [Kustomize](https://kustomize.io/)
- [Helm](https://helm.sh/)
- [Ksonnet](https://ksonnet.io/)

### 2.1.3 jenkins集成

- [Jenkins User Guide](https://jenkins.io/)

# 三 核心概念

假设您熟悉核心的Git，Docker，Kubernetes，持续交付和GitOps概念。

* **Application**：清单定义的一组Kubernetes资源。这是一个自定义资源定义（CRD）。

* **Application source type**：使用哪个工具来构建应用程序
* **Target state**：应用程序的期望状态，由Git存储库中的文件表示。
* **Live state**：该应用程序的实时状态。部署了哪些Pod等。
* **Sync status**：实时状态是否与目标状态匹配。部署的应用程序是否与Git所说的相同？
* **Sync**：使应用程序移至其目标状态的过程。例如。通过将更改应用于Kubernetes集群。
* **Refresh**：将Git中的最新代码与实时状态进行比较。找出有什么不同。
* **Health**:应用程序的运行状况是否正常运行？它可以满足请求吗？
* **Tool**:从文件目录创建清单的工具。例如。Kustomize或Ksonnet。请参阅应用程序源类型。

# 四 安装部署

## 4.1 安装Argo CD

```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

这将创建一个新的命名空间argocd，Argo CD服务和应用程序资源将驻留在该命名空间中。

## 4.2 下载Argo CD CLI

从 https://github.com/argoproj/argo-cd/releases/latest.下载Argo CD工具

* mac可以利用如下方式进行安装

```shell
brew tap argoproj/tap
brew install argoproj/tap/argocd
```

* Linux安装

```shell
VERSION=$(curl --silent "https://api.github.com/repos/argoproj/argo-cd/releases/latest" | grep '"tag_name"' | sed -E 's/.*"([^"]+)".*/\1/')
curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/$VERSION/argocd-linux-amd64
chmod +x /usr/local/bin/argocd
```

可以参考：https://argoproj.github.io/argo-cd/cli_installation/

## 4.3 访问Argo CD API server

默认情况下，Argo CD API服务器未使用外部IP公开。要访问API服务器，请选择以下技术之一以公开Argo CD API服务器：

端口转发

```
[root@master ~]# kubectl port-forward svc/argocd-server -n argocd --address 0.0.0.0 8080:443
```

## 4.4 通过CLI登录Argo

初始密码将自动生成为Argo CD API服务器的容器名称。可以使用以下命令进行检索：

```yaml
[root@master ~]# kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server -o name | cut -d'/' -f 2
argocd-server-656f9b895b-bfjvw
```

使用用户名admin和上面的密码，登录到Argo CD的IP或主机名：

```
argocd login <ARGOCD_SERVER>
```

使用以下命令更改密码：

```
argocd account update-password
```

```yaml
# 登录
[root@master ~]# argocd login localhost:8080
WARNING: server certificate had error: x509: certificate signed by unknown authority. Proceed insecurely (y/n)? yes
Username: admin
Password: 
'admin' logged in successfully
Context 'localhost:8080' updated
# 修改密码
[root@master ~]# argocd account update-password
*** Enter current password: 
*** Enter new password: 
*** Confirm new password: 
Password updated
Context 'localhost:8080' updated
```

## 4.5 注册集群以将应用程序部署到（可选）

此步骤将群集的凭据注册到Argo CD，仅在部署到外部群集时才需要。在内部进行部署（到与Argo CD运行所在的同一集群）时，应将https：//kubernetes.default.svc用作应用程序的K8s API服务器地址。

首先列出当前kubconfig中的所有集群上下文：

```
argocd cluster add
```

上面的命令将ServiceAccount（argocd-manager）安装到该kubectl上下文的kube-system命名空间中，并将服务帐户绑定到管理员级别的ClusterRole。Argo CD使用此服务帐户令牌执行其管理任务（即部署/监视）。

可以修改argocd-manager-role角色的规则，使其仅具有对一组有限的名称空间，组和种类的创建，更新，修补，删除特权。但是，在群集作用域中，获取，列出，监视特权是Argo CD起作用所必需的。

## 4.6 从Git存储库创建应用程序

一个guestbook的应用来仓库保护示例，来说明Argo CD如何去工作：https://github.com/argoproj/argocd-example-apps.git

![image-20200124123932233](/Users/xuel/Library/Application Support/typora-user-images/image-20200124123932233.png)

### 4.6.1 通过CLI创建APP

You can access Argo CD using port forwarding: add `--port-forward-namespace argocd` flag to every CLI command or set `ARGOCD_OPTS` environment variable: `export ARGOCD_OPTS='--port-forward-namespace argocd'`:

```
argocd app create guestbook --repo https://github.com/argoproj/argocd-example-apps.git --path guestbook --dest-server https://kubernetes.default.svc --dest-namespace default
```



### 4.6.2 通过web UI创建APP

打开浏览器进入Argo CD外部UI，然后通过在浏览器中访问IP /主机名登录并使用在步骤4中设置的凭据。

![image-20200124124103128](/Users/xuel/Library/Application Support/typora-user-images/image-20200124124103128.png)

给您的应用命名为留言簿，使用项目默认值，并将同步策略保留为“手动”：

![image-20200124124149123](/Users/xuel/Library/Application Support/typora-user-images/image-20200124124149123.png)

Connect the https://github.com/redhatxl/argocd-example-apps.git  repo to Argo CD by setting repository url to the github repo url, leave revision as `HEAD`, and set the path to `guestbook`:

注意：仓库可以使用我fork的，目前以及更改了镜像地址，可以正常拉取，

![image-20200124125638869](/Users/xuel/Library/Application Support/typora-user-images/image-20200124125638869.png)

⚠️注意：Path对应项目中资源清单的目录。	

对于目标，将集群设置为集群内，并将名称空间设置为默认值：

![image-20200124124331693](/Users/xuel/Library/Application Support/typora-user-images/image-20200124124331693.png)

填写完以上信息后，请单击UI顶部的“创建”以创建留言簿应用程序：

![image-20200124125905114](/Users/xuel/Library/Application Support/typora-user-images/image-20200124125905114.png)



## 4.7 同步（部署）应用

创建留言簿应用程序后，您现在可以查看其状态：

```shell
[root@master ~]# argocd app get guestbook
Name:               guestbook
Project:            default
Server:             https://kubernetes.default.svc
Namespace:          default
URL:                https://localhost:8080/applications/guestbook
Repo:               https://github.com/redhatxl/argocd-example-apps.git
Target:             HEAD
Path:               guestbook
SyncWindow:         Sync Allowed
Sync Policy:        <none>
Sync Status:        OutOfSync from HEAD (4973f15)
Health Status:      Missing

GROUP  KIND        NAMESPACE  NAME          STATUS     HEALTH   HOOK  MESSAGE
       Service     default    guestbook-ui  OutOfSync  Missing        
apps   Deployment  default    guestbook-ui  OutOfSync  Missing 
```

由于尚未部署应用程序，并且尚未创建Kubernetes资源，因此应用程序状态最初处于OutOfSync状态。要同步（部署）应用程序，请运行：

```shell
[root@master ~]# argocd app sync guestbook
TIMESTAMP                  GROUP        KIND   NAMESPACE                  NAME    STATUS    HEALTH        HOOK  MESSAGE
2020-01-24T12:59:13+08:00            Service     default          guestbook-ui  OutOfSync  Missing              
2020-01-24T12:59:13+08:00   apps  Deployment     default          guestbook-ui  OutOfSync  Missing              
2020-01-24T12:59:14+08:00            Service     default          guestbook-ui    Synced  Healthy              

Name:               guestbook
Project:            default
Server:             https://kubernetes.default.svc
Namespace:          default
URL:                https://localhost:8080/applications/guestbook
Repo:               https://github.com/redhatxl/argocd-example-apps.git
Target:             HEAD
Path:               guestbook
SyncWindow:         Sync Allowed
Sync Policy:        <none>
Sync Status:        Synced to HEAD (4973f15)
Health Status:      Progressing

Operation:          Sync
Sync Revision:      4973f150497aa18bc5fbd81b19102db2469eb6f0
Phase:              Succeeded
Start:              2020-01-24 12:59:11 +0800 CST
Finished:           2020-01-24 12:59:12 +0800 CST
Duration:           1s
Message:            successfully synced (all tasks run)

GROUP  KIND        NAMESPACE  NAME          STATUS  HEALTH       HOOK  MESSAGE
       Service     default    guestbook-ui  Synced  Healthy            service/guestbook-ui created
apps   Deployment  default    guestbook-ui  Synced  Progressing        deployment.apps/guestbook-ui created
```

![image-20200124130007212](/Users/xuel/Library/Application Support/typora-user-images/image-20200124130007212.png)

此命令从存储库中检索清单，并对清单执行kubectl应用。该留言簿应用程序现在正在运行，您现在可以查看其资源组件，日志，事件和评估的健康状态：

部署完成后状态：

![image-20200124130111558](/Users/xuel/Library/Application Support/typora-user-images/image-20200124130111558.png)



![image-20200124130129877](/Users/xuel/Library/Application Support/typora-user-images/image-20200124130129877.png)

```shell
[root@master ~]# kubectl  get all -l app=guestbook-ui
NAME                                READY   STATUS    RESTARTS   AGE
pod/guestbook-ui-7bc795dc8c-m69fl   1/1     Running   0          3m6s

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/guestbook-ui-7bc795dc8c   1         1         1       3m6s
```

修改guestbook的访问方式为NodePort

![image-20200124130534172](/Users/xuel/Library/Application Support/typora-user-images/image-20200124130534172.png)

![image-20200124130627176](/Users/xuel/Library/Application Support/typora-user-images/image-20200124130627176.png)

# 五 操作手册

## 5.1 概述

本指南适用于希望为其他开发人员安装和配置Argo CD的管理员和操作员。

## 5.2 架构概述

![image-20200124130822382](/Users/xuel/Library/Application Support/typora-user-images/image-20200124130822382.png)

### 5.2.1 API Server

API服务器是gRPC / REST服务器，它公开了Web UI，CLI和CI / CD系统使用的API。它具有以下职责：

* 应用管理和状态报告
* 应用调用操作（同步，回滚，用户定义）
* 仓库和集群认证管理
* 向外部身份提供者的身份验证和身份验证委派
* RBAC强制执行
* 监听/转发Git webhook events

### 5.2.2 Repository Server

仓库服务是一个内部的服务，维护了一个本地的git仓库缓存了应用的manifests，利用他来生成和返回kubernetes的manifests当提供一下的输入的时候

* 仓库URL
* 版本（commit，tag，branch）
* 应用补丁
* 模版设置

### 5.2.3 Application Controller

应用控制器是一个k8s的控制器，利用他来持续的监控正在运行的应用和对比当前的状态，和存活的状态分配已经期望的目标状态（来源于制定的repo）， 他检测outofsync应用状态和可选的任务，它调用任何用户自定义的hooks对于声明周期事件



## 5.3 声明设置

### 5.3.1 快速参考

可以使用Kubernetes清单声明性地定义Argo CD应用程序，项目和设置。

| Name                                                         | Kind        | Description                                                  |
| :----------------------------------------------------------- | :---------- | :----------------------------------------------------------- |
| [`argocd-cm.yaml`](https://argoproj.github.io/argo-cd/operator-manual/argocd-cm.yaml) | ConfigMap   | General Argo CD configuration                                |
| [`argocd-secret.yaml`](https://argoproj.github.io/argo-cd/operator-manual/argocd-secret.yaml) | Secret      | Password, Certificates, Signing Key                          |
| [`argocd-rbac-cm.yaml`](https://argoproj.github.io/argo-cd/operator-manual/argocd-rbac-cm.yaml) | ConfigMap   | RBAC Configuration                                           |
| [`argocd-tls-certs-cm.yaml`](https://argoproj.github.io/argo-cd/operator-manual/argocd-tls-certs-cm.yaml) | ConfigMap   | Custom TLS certificates for connecting Git repositories via HTTPS (v1.2 and later) |
| [`argocd-ssh-known-hosts-cm.yaml`](https://argoproj.github.io/argo-cd/operator-manual/argocd-ssh-known-hosts-cm.yaml) | ConfigMap   | SSH known hosts data for connecting Git repositories via SSH (v1.2 and later) |
| [`application.yaml`](https://argoproj.github.io/argo-cd/operator-manual/application.yaml) | Application | Example application spec                                     |
| [`project.yaml`](https://argoproj.github.io/argo-cd/operator-manual/project.yaml) | AppProject  | Example project spec                                         |

### 5.3.2 应用

Application CRD是Kubernetes资源对象，代表环境中已部署的应用程序实例。它由两个关键信息定义：

* 对Git中所需状态的源引用（存储库，修订版，路径，环境）对

* 目标集群和名称空间的目标引用。

最低的应用程序规范如下：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: guestbook
```



参考 [application.yaml](https://argoproj.github.io/argo-cd/operator-manual/application.yaml) 查看附加字段

名称空间必须与Argo cd的名称空间匹配，通常为argocd。

默认情况下，删除应用程序将不会执行级联删除，从而删除其资源。如果需要这种行为，则必须添加终结器-您可能不希望这样做。

您可以创建一个可以创建其他应用程序的应用程序，而该应用程序又可以创建其他应用程序。这样，您就可以声明性地管理可以协调部署和配置的一组应用程序。请参阅群集引导。

AppProject CRD是Kubernetes资源对象，代表应用程序的逻辑分组。它由以下关键信息定义：

* sourceRepos引用项目中的应用程序可以从中提取清单的存储库。
* 目标引用项目中的应用程序可以部署到的群集和名称空间。
* 实体的角色列表及其在项目中对资源的访问权限的定义。



# 参考链接

* https://argoproj.github.io/argo-cd/?spm=a2c4e.10696291.0.0.6fc419a4zmeq9o
* https://www.weave.works/blog/gitops-is-cloud-native?spm=a2c4e.10696291.0.0.6e0319a4UJmgT6

