# Kubernetes安全之Open Policy Agent

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220302131958.png)

## 一 前言

无论是在业务开发还是在业务直接的访问，很多时候，Policy 的实现都与具体的服务耦合在一起，这导致 Policy 很难被单独抽象描述和灵活变更（比如动态加载新的规则）。而且，不同的服务对 Policy 有着不同的描述，比如采用 JSON、YAML 或其他 DSL，这样很难进一步将 Policy 以代码的形式进行管理（Policy As Code）。因此，为了更**灵活和一致**的架构，我们必须将 Policy 的决策过程从具体的服务接耦出去，这也是 OPA 整体架构的核心理念。

## 二 Open Policy Agent

### 2.1 简介

OPA 是一个轻量级的通用策略引擎，可以与您的服务共存。您可以将 OPA 集成为 sidecar、主机级守护进程或库。服务通过执行查询将策略决策卸载给 OPA。OPA 评估策略和数据以生成查询结果(这些结果被发送回客户端)。策略是用高级声明性语言编写的，可以通过 api 或本地文件系统远程将策略加载到 OPA 中。

OPA 将策略决策与策略执行分离。当您的软件需要做出策略决策时，它会查询 OPA 并提供结构化数据（例如 JSON）作为输入。 OPA 接受任意结构化数据作为输入。

开放策略代理(Open Policy Agent，发音为“ oh-pa”)是一个开放源码的通用策略引擎，它统一了跨堆栈的策略实施。OPA 提供了一种高级声明性语言，允许您将策略指定为代码和简单的 api，以卸载软件中的策略决策。您可以使用 OPA 在 microservices、 Kubernetes、 CI/CD 管道、 API 网关等中强制执行策略。

### 2.2 Rego策略代码

在OPA中策略管理使用Rego语言，Rego 是专门为 OPA 构建的高级声明语言。它使得定义策略和解决以下问题变得非常容易: 是否允许 Bob 在/api/v1/products 上执行 GET 请求？他实际上允许查看哪些记录？

Rego 语言中最核心的一个功能的是对 Policy 的描述，即 Rules。**Rules 是一个 if-then 的 logic statement**。Rego 的 Rules 分为两类：**complete rules 和 incremental rules**。

例如下面input_containers[c]就是一个head，剩下的 `{...}` 则为 body。

```go
// 你必须使用kubernetes.admission
package kubernetes.admission

// input_containers[c]函数从请求对象中提取容器。注意，使用了_字符来遍历数组中的所有容器。在Rego中，你不需要定义循环—下划线字符将自动为你完成此操作。
input_containers[c] {
  c := input.request.object.spec.containers[_]
}
// 我们再次为init容器定义函数。请注意，在Rego中，可以多次定义同一个函数。这样做是为了克服Rego函数中不能返回多个输出的限制。当调用函数名时，将执行两个函数，并使用AND操作符组合输出。因此，在我们的例子中，在一个或多个位置中存在一个有特权的容器将违反策略。
input_containers[c] {
  c := input.request.object.spec.initContainers[_]
}

// Deny是默认对象，它将包含我们需要执行的策略。如果所包含的代码计算结果为true，则将违反策略。
deny[msg] {
  // 定义了一个变量，它将容纳pod中的所有容器，并定义的input_containers[c]接收值。
  c := input_containers[_]
  // pod包含“privileged”属性，则该语句为true。
  c.securityContext.privileged
  // 用户尝试运行特权容器时显示给他们的消息。它包括容器名称和违规的安全上下文。
  msg := sprintf("Privileged container is not allowed: %v, securityContext: %v", [c.name, c.securityContext])
}

```

## 三 架构详解

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220302132358.png)

OPA 通过根据策略和数据评估查询输入来生成策略决策。 OPA 和 Rego 与域无关，因此您可以在策略中描述几乎任何类型的不变量。例如：

* 哪些用户可以访问哪些资源。
* 允许哪些子网出口通信。
* 必须将工作负载部署到哪些集群。
* 可以从哪些注册表中下载二进制文件。
* 容器可以使用哪些操作系统功能执行。
* 系统在一天中的哪些时间可以访问。

政策决定不限于简单的是/否或允许/拒绝答案。与查询输入一样，您的策略可以生成任意结构化数据作为输出。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220302140058.png)

我们再次为init容器定义函数。请注意，在Rego中，可以多次定义同一个函数。这样做是为了克服Rego函数中不能返回多个输出的限制。当调用函数名时，将执行两个函数，并使用AND操作符组合输出。因此，在我们的例子中，在一个或多个位置中存在一个有特权的容器将违反策略。

OPA集成

OPA 的运行**依赖于 Rego 运行环境**，因此 OPA 提供了两种主要的使用方式：

- **Go Library**

  直接将 Rego 的运行环境以库的形式整合到用户服务中。当服务需要执行 Rego 代码时，只需要调用相关的 API 即可；

- **REST API**

  如果服务不是以用 Go 编写，为了拥有 Rego 运行环境，此时则必须使用 REST API。OPA 提供了一个 HTTP 服务，当其他组件需要执行 Rego 代码时，则将**代码和数据一起以 REST API 的形式传给 OPA HTTP 服务**，待 Rego 执行完后将相应的结果返回；

## 四 在Kubernetes中使用POA

### 4.1 OPA As An Admission Controller

OPA 作为准入控制器，在部署的时候需要提前预置条件：

* 强制要求 pod 必须有一个 sidecar 容器。此 sidecar 容器可以根据您的安全策略的要求执行审计或日志记录任务。
* 集群启用了 `MutatingAdmissionWebhook` 和`ValidatingAdmissionWebhook`；

* 修改所有资源以具有特定的注释。
* 改变容器镜像以始终指向企业镜像注册表。
* 将节点和 pod 关联性和反关联性选择器设置为 Deployment。

### 4.2 The OPA Gatekeeper

Gatekeeper 是一个相对较新的项目，旨在极大地增强和促进 OPA 和 Kubernetes 之间的集成。那么，Gatekeeper 为普通 OPA 带来了哪些额外的功能？

* 可扩展的参数化策略库。
* 用于创建约束的 Kubernetes 自定义资源定义 (CRD)。
* 另一个用于扩展约束的 CRD（约束模板）。
* 审计能力。

### 4.3 安装Gatekeeper

Kubernetes 1.14 或更高版本。并且需要集群的管理权限。

检查完上述要求后，安装 OPA Gatekeeper 的最简单方法是运行以下命令：

```yaml
  kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/master/deploy/gatekeeper.yaml
```

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220302145409.png)

### 4.4 示例

从创建约束模板开始——同样，该模板定义了您需要定义的参数以及将执行验证的实际 Rego 代码。 k8srequiredregistry_template.yaml 应如下所示：

```yaml
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8srequiredregistry
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredRegistry
      validation:
        # Schema for the `parameters` field
        openAPIV3Schema:
          properties:
            image:
              type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredregistry
        violation[{"msg": msg, "details": {"Registry should be": required}}] {
          input.review.object.kind == "Pod"
          some i
          image := input.review.object.spec.containers[i].image
          required := input.parameters.registry
          not startswith(image,required)
          msg := sprintf("Forbidden registry: %v", [image])
        }
```

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220302151020.png)

```shell
[root@VM-48-7-centos opaexample]# k get ConstraintTemplate k8srequiredregistry
NAME                  AGE
k8srequiredregistry   68s
```

* 应用模版定义

```shell
kubectl apply -f k8srequiredregistry_template.yaml.

apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredRegistry
metadata:
  name: images-must-come-from-gcr
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
  parameters:
    registry: "gcr.io/"

```



```shell
kubectl apply -f all_images_must_come_from_gcr.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: busybox
spec:
  selector:
    matchLabels:
      app: busybox
  replicas: 1
  template:
    metadata:
      labels:
        app: busybox
    spec:
      containers:
      - name: busybox
        image: gcr.io/google-containers/busybox
        command:
        - sh
        - -c
        - sleep 1000000

```

策略会拦截掉镜像为grc.io的。

## 五 策略

### 4.1 受信任镜像仓库

此策略很简单，但功能强大：仅允许从受信任的镜像仓库中拉取容器镜像。从网上随意拉取未知镜像会带来风险，例如恶意软件。通过确保镜像仅来自受信任的镜像仓库，你可以紧密控制镜像安全，减轻软件被攻击的风险，随之提高集群的整体安全性。

#### 4.1.1 相关策略：

- 禁止使用带有” latest “标签的镜像
- 仅使用签名的镜像，或与特定哈希/SHA匹配的镜像

#### 4.1.2 示例

```
package kubernetes.validating.images
 
deny[msg] {
    some i
    input.request.kind.kind == "Pod"
    image := input.request.object.spec.containers[i].image
    not startswith(image, "anchnet./")
    msg := sprintf("Image '%v' comes from untrusted registry", [image])
}
```

### 4.2 禁止使用特权模式

此策略，确保默认情况下容器不能在特权模式下运行。

通常，你要避免在特权模式下运行容器，因为它提供对主机资源和内核功能（包括禁用主机级保护的功能）的访问。

尽管容器在某种程度上是隔离的，但它们最终共享同一内核。这意味着，如果特权容器遭到破坏，则它可能成为破坏整个系统的起点。如果想要在特权模式下运行镜像-仅确保这些时间是例外，而不是常规。

#### 4.2.1 相关策略：

- 禁止不安全的能力
- 禁止容器以root身份运行（要以非root身份运行）
- 设置用户ID

#### 4.2.2 示例：

```
package kubernetes.validating.privileged
 
deny[msg] {
    some c
    input_container[c]
    c.securityContext.privileged
    msg := sprintf("Container '%v' should not run in privileged mode.", [c.name])
}
 
input_container[container] {
    container := input.request.object.spec.containers[_]
}
 
input_container[container] {
    container := input.request.object.spec.initContainers[_]
}
```



## 六 价值

OPA 提出的 Policy Engine 的图景是很美好的，即 **Policy As Code**。如果能够将 Policy 以一种清晰统一的代码形式管理，那么可以我们至少可以获得以下几个好处：

- **全局统一的 Policy 描述方式**

  所有开发者对 Policy 的描述不再仅仅内嵌于业务代码中，也不再以多种不同的形式描述，而是以一种统一的语言进行表达；

- **提升运维效率**

  虽然应用 Policy As Code 初期可能会带来一定的学习成本，但是后期的价值是可以让 Policy 拥有类似代码的特性：

  - 不同的 Policy 代码可以根据需求进行整合，比如每个业务必须先集成公司层面的 Policy 代码才可以开发自身业务相关的 Policy；
  - 将安全准入的规则抽象成 Policy 代码供业务直接使用，从而让业务无需过多关注安全；
  - 不同的 Policy 可以像代码一样进行自动化检查和测试（比如公司或组织层面的合规检查等）；
  - 更方便 Review；

- **提升业务架构灵活度**

  类似于 OPA 的接耦架构可让原来业务实现更为灵活的功能，比如：

  - 动态更新和加载新的 Policy；
  - 多个组件共享同一份 Policy 执行相关逻辑（比如多个组件共享同一个 Authorization 组件）；
  - **可编程能力**：实现新的 Policy 只需要写新的 Policy DSL 即可，无须改变业务和框架层逻辑，理论上编写 DSL 代码的复杂度要远远小于用原生语言开发；

- **更细粒度和高效的安全保障**

  代码化后的 Policy 可以进行细粒度和灵活的描述，比如可以针对特定字段进行特殊判断等。针对 Authorization，可很容易根据这一特点实现黑白名单功能，这比以往使用 RBAC 形式的授权要更为灵活。

  对安全的理解可抽象成 Policy 代码强制自动执行，这样对安全的保障不再仅仅依靠开发者自身意识或运维人员的配置 Review，而是直接以统一的代码形式体现；



## 参考链接

* https://github.com/open-policy-agent/opa
* https://www.openpolicyagent.org/
* https://docs.google.com/presentation/d/16QV6gvLDOV3I0_guPC3_19g6jHkEg3X9xqMYgtoCKrs/edit#slide=id.g107b5c4c845_0_0
* https://thenewstack.io/open-policy-agent-the-top-5-kubernetes-admission-control-policies/
* https://cloud.tencent.com/developer/article/1741460
* https://zhengyinyong.com/post/opa-research/
* https://www.magalix.com/blog/integrating-open-policy-agent-opa-with-kubernetes-a-deep-dive-tutorial
* https://play.openpolicyagent.org/

