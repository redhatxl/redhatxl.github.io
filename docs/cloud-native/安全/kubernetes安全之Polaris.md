## kubernetes安全之Polaris

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220302153810.png)

## 一 polaris简介

Fairwinds' Polaris 让您的集群顺利航行。它运行各种检查以确保使用最佳实践配置 Kubernetes pod 和控制器，从而帮助您避免将来出现问题。

Polaris 可以在三种不同的模式下运行：

* 作为仪表板，您可以审核集群内运行的内容。

* 作为准入控制器，您可以自动拒绝不符合组织政策的工作负载。
* 作为命令行工具，您可以测试本地 YAML 文件，例如作为 CI/CD 流程的一部分。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220302155740.png)





## 二 部署Polaris

### 2.1 DashBoard部署

Polaris 仪表板可以使用 kubectl 或 Helm 安装在集群上。它也可以在本地运行，使用存储在您的 KUBECONFIG 中的凭证连接到您的集群。

仪表板是了解集群或基础设施即代码中的哪些工作负载不符合最佳实践的好方法。

#### 2.1.1 kubectl安装

```bash
kubectl apply -f https://github.com/fairwindsops/polaris/releases/latest/download/dashboard.yaml
kubectl port-forward --namespace polaris svc/polaris-dashboard 8080:80
```

#### 2.1.2 Helm安装

```bash
helm repo add fairwinds-stable https://charts.fairwinds.com/stable
helm upgrade --install polaris fairwinds-stable/polaris --namespace polaris --create-namespace
kubectl port-forward --namespace polaris svc/polaris-dashboard --address 0.0.0.0 8080:80
```

#### 2.1.3 Local Binary

您需要为仪表板设置有效的 KUBECONFIG 才能连接到您的集群。 二进制版本可以从发布页面下载（打开新窗口），也可以使用 Homebrew 安装（打开新窗口）：

```bash
brew tap reactiveops/tap
brew install reactiveops/tap/polaris
polaris dashboard --port 8080
```

您还可以将仪表板指向本地文件系统，而不是实时集群：

```shell
polaris dashboard --port 8080 --audit-path=./deploy/
```

图像界面

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220303100536.png)

查看容器配置异常

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220302191701.png)



### 2.2 Admission Controller

Polaris 可以作为一个接纳控制器运行，作为一个确认的网络挂钩。它接受与仪表板相同的配置，并可以运行相同的验证。Webhook 将拒绝触发危险级别检查的任何工作负载。这表明了 Polaris 更大的目标，不仅仅是通过仪表板可见性来鼓励更好的配置，而是通过这个 webhook 来实际执行它。请注意，Polaris 不会改变您的工作负载，只会阻止不符合配置策略的工作负载。

#### 2.2.1 安装

使用helm进行部署

```shell
 helm repo add fairwinds-stable https://charts.fairwinds.com/stable
helm upgrade --install polaris fairwinds-stable/polaris --namespace polaris --create-namespace \
  --set webhook.enable=true --set dashboard.enable=false
```

### 2.3 Infrastructure as Code

想在一个地方看到所有检测业务的结果吗？

查看 Fairwinds Insights (打开新窗口) Polaris 可以在命令行上用于审核存储在 YAML 文件中的本地 Kubernetes 清单。

这对于针对作为 CI/CD 管道一部分的基础设施代码运行 Polaris 特别有帮助。使用可用的命令行标志，如果您的 Polaris 得分下降到某个阈值以下，或者出现任何危险级别的问题，将导致 CI/CD 失败



## 三 检查项目

### 3.1 Security

容器需要遵循安全规则

| key                          | default   | description                                                  |
| ---------------------------- | --------- | ------------------------------------------------------------ |
| `hostIPCSet`                 | `danger`  | Fails when `hostIPC` attribute is configured.                |
| `hostPIDSet`                 | `danger`  | Fails when `hostPID` attribute is configured.                |
| `notReadOnlyRootFilesystem`  | `warning` | Fails when `securityContext.readOnlyRootFilesystem` is not true. |
| `privilegeEscalationAllowed` | `danger`  | Fails when `securityContext.allowPrivilegeEscalation` is true. |
| `runAsRootAllowed`           | `warning` | Fails when `securityContext.runAsNonRoot` is not true.       |
| `runAsPrivileged`            | `danger`  | Fails when `securityContext.privileged` is true.             |
| `insecureCapabilities`       | `warning` | Fails when `securityContext.capabilities` includes one of the capabilities [listed here(opens new window)](https://github.com/FairwindsOps/polaris/tree/master/checks/insecureCapabilities.yaml) |
| `dangerousCapabilities`      | `danger`  | Fails when `securityContext.capabilities` includes one of the capabilities [listed here(opens new window)](https://github.com/FairwindsOps/polaris/tree/master/checks/dangerousCapabilities.yaml) |
| `hostNetworkSet`             | `warning` | Fails when `hostNetwork` attribute is configured.            |
| `hostPortSet`                | `warning` | Fails when `hostPort` attribute is configured.               |
| `tlsSettingsMissing`         | `warning` | Fails when an Ingress lacks TLS settings.                    |



确保 Kubernetes 的工作负载安全是整个集群安全的重要组成部分。总体目标应该是确保容器以尽可能少的特权运行。这包括避免使用权限提升文件，不使用带有根用户的容器，不给主机网络提供过多的访问权限，以及尽可能使用只读文件系统。

启用 hostNetwork 属性运行的 pod 将可以访问环回设备，监听本地主机上的服务，并且可以用来窥探同一节点上其他吊舱的网络活动。在某些情况下，需要将 hostNetwork 设置为 true，比如部署一个像 Flannel 这样的网络插件。

在容器上设置 hostPort 属性将确保在它部署到的每个节点上的特定端口上都可以访问它。不幸的是，当它被指定时，它限制了一个 pod 可以在集群中实际调度的位置。

### 3.2 Efficiency

这些检查确保配置了 CPU 和内存设置，以便 Kubernetes 能够有效地调度您的工作负载。

为在 Kubernetes 运行的容器配置资源请求和限制是一个重要的最佳实践。设置适当的资源请求将确保所有应用程序具有足够的计算资源。设置适当的资源限制将确保应用程序不会消耗太多资源。

| key                     | default   | description                                                  |
| ----------------------- | --------- | ------------------------------------------------------------ |
| `cpuRequestsMissing`    | `warning` | Fails when `resources.requests.cpu` attribute is not configured. |
| `memoryRequestsMissing` | `warning` | Fails when `resources.requests.memory` attribute is not configured. |
| `cpuLimitsMissing`      | `warning` | Fails when `resources.limits.cpu` attribute is not configured. |
| `memoryLimitsMissing`   | `warning` | Fails when `resources.limits.memory` attribute is not configured. |

### 3.3 Reliability

这些检查有助于确保您的工作负载始终可用，并且正在运行正确的映像。

| key                          | default   | description                                                  |
| ---------------------------- | --------- | ------------------------------------------------------------ |
| `readinessProbeMissing`      | `warning` | Fails when a readiness probe is not configured for a pod.    |
| `livenessProbeMissing`       | `warning` | Fails when a liveness probe is not configured for a pod.     |
| `tagNotSpecified`            | `danger`  | Fails when an image tag is either not specified or `latest`. |
| `pullPolicyNotAlways`        | `warning` | Fails when an image pull policy is not `always`.             |
| `priorityClassNotSet`        | `ignore`  | Fails when a priorityClassName is not set for a pod.         |
| `deploymentMissingReplicas`  | `warning` | Fails when there is only one replica for a deployment.       |
| `missingPodDisruptionBudget` | `ignore`  |                                                              |

## 四 自定义检测

### 4.1 禁用quay.o

```shell
 checks:
  imageRegistry: warning

customChecks:
  imageRegistry:
    successMessage: Image comes from allowed registries
    failureMessage: Image should not be from disallowed registry
    category: Security
    target: Container
    schema:
      '$schema': http://json-schema.org/draft-07/schema
      type: object
      properties:
        image:
          type: string
          not:
            pattern: ^quay.io
```

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220303101623.png)

## 参考链接

* https://polaris.docs.fairwinds.com/

* https://github.com/FairwindsOps/polaris