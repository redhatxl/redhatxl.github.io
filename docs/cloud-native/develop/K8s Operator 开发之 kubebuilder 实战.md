## 一 背景
对于业务应用，需要对其进行先k8s内置资源进行一系列运维操作，因此编写业务的operator必不可少，在此了解到kubebuilder 是社区认可度很高的一种官方、标准化 Operator 框架，可以利用其非常方便的编写业务operator，以此来扩展 Kubernetes API

### 1.1 kubebuilder是什么
Kubebuilder 是一个使用 CRDs 构建 K8s API 的 SDK，主要是：

* 提供脚手架工具初始化 CRDs 工程，自动生成 boilerplate 代码和配置；
* 提供代码库封装底层的 K8s go-client；

方便用户从零开始开发 CRDs，Controllers 和 Admission Webhooks 来扩展 K8s。

### 1.2 功能
自定义资源 CRD（Custom Resource Definition）可以扩展 Kubernetes API，掌握 CRD 是成为 Kubernetes 高级玩家的必备技能，本文将介绍 CRD 和 Controller 的概念，并对 CRD 编写框架 Kubebuilder 进行深入分析，让您真正理解并能快速开发 CRD。

### 1.3 基本概念

- **CRD (Custom Resource Definition)**: 允许用户自定义 Kubernetes 资源，是一个类型；
- **CR (Custom Resourse)**: CRD 的一个具体实例；
- **webhook**: 它本质上是一种 HTTP 回调，会注册到 apiserver 上。在 apiserver 特定事件发生时，会查询已注册的 webhook，并把相应的消息转发过去。

按照处理类型的不同，一般可以将其分为两类：一类可能会修改传入对象，称为 mutating webhook；一类则会只读传入对象，称为 validating webhook。

- **工作队列**: controller 的核心组件。它会监控集群内的资源变化，并把相关的对象，包括它的动作与 key，例如 Pod 的一个 Create 动作，作为一个事件存储于该队列中；
- **controller** :它会循环地处理上述工作队列，按照各自的逻辑把集群状态向预期状态推动。不同的 controller 处理的类型不同，比如 replicaset controller 关注的是副本数，会处理一些 Pod 相关的事件；
- **operator**:operator 是描述、部署和管理 kubernetes 应用的一套机制，从实现上来说，可以将其理解为 CRD 配合可选的 webhook 与 controller 来实现用户业务逻辑，即 operator = CRD + webhook + controller。

## 二 架构即基本概念

### 2.1 架构图
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/89a57f198ebe4efc9d35d92cb94171c8~tplv-k3u1fbpfcp-zoom-1.image)

### 2.2 基础概念

#### 2.2.1 GVKs&GVRs
GVK = GroupVersionKind，GVR = GroupVersionResource。


* API Group & Versions（GV）
API Group 是相关 API 功能的集合，每个 Group 拥有一或多个 Versions，用于接口的演进。


* Kinds & Resources（GVR）
每个 GV 都包含多个 API 类型，称为 Kinds，在不同的 Versions 之间同一个 Kind 定义可能不同， Resource 是 Kind 的对象标识（resource type），一般来说 Kinds 和 Resources 是 1:1 的，比如 pods Resource 对应 Pod Kind，但是有时候相同的 Kind 可能对应多个 Resources，比如 Scale Kind 可能对应很多 Resources：deployments/scale，replicasets/scale，对于 CRD 来说，只会是 1:1 的关系。

每一个 GVK 都关联着一个 package 中给定的 root Go type，比如 apps/v1/Deployment 就关联着 K8s 源码里面 k8s.io/api/apps/v1 package 中的 Deployment struct，我们提交的各类资源定义 YAML 文件都需要写：

apiVersion：这个就是 GV 。
kind：这个就是 K。

根据 GVK K8s 就能找到你到底要创建什么类型的资源，根据你定义的 Spec 创建好资源之后就成为了 Resource，也就是 GVR。GVK/GVR 就是 K8s 资源的坐标，是我们创建/删除/修改/读取资源的基础。

#### 2.2.3 Scheme
每一组 Controllers 都需要一个 Scheme，提供了 Kinds 与对应 Go types 的映射，也就是说给定 Go type 就知道他的 GVK，给定 GVK 就知道他的 Go type，比如说我们给定一个 Scheme: "tutotial.kubebuilder.io/api/v1".CronJob{} 这个 Go type 映射到 batch.tutotial.kubebuilder.io/v1 的 CronJob GVK，那么从 Api Server 获取到下面的 JSON:
```
{
    "kind": "CronJob",
    "apiVersion": "batch.tutorial.kubebuilder.io/v1",
    ...
}
```
就能构造出对应的 Go type了，通过这个 Go type 也能正确地获取 GVR 的一些信息，控制器可以通过该 Go type 获取到期望状态以及其他辅助信息进行调谐逻辑。



#### 2.2.4 Manager
Kubebuilder 的核心组件，具有 3 个职责：

* 负责运行所有的 Controllers；
* 初始化共享 caches，包含 listAndWatch 功能；
* 初始化 clients 用于与 Api Server 通信。

#### 2.2.5 Cache

Kubebuilder 的核心组件，负责在 Controller 进程里面根据 Scheme 同步 Api Server 中所有该 Controller 关心 GVKs 的 GVRs，其核心是 GVK -> Informer 的映射，Informer 会负责监听对应 GVK 的 GVRs 的创建/删除/更新操作，以触发 Controller 的 Reconcile 逻辑。

#### 2.2.6 Controller
Kubebuidler 为我们生成的脚手架文件，我们只需要实现 Reconcile 方法即可。

#### 2.2.7 Client

在实现 Controller 的时候不可避免地需要对某些资源类型进行创建/删除/更新，就是通过该 Clients 实现的，其中查询功能实际查询是本地的 Cache，写操作直接访问 Api Server。



#### 2.2.8 Index
由于 Controller 经常要对 Cache 进行查询，Kubebuilder 提供 Index utility 给 Cache 加索引提升查询效率。
#### 2.2.9 Finalizer

在一般情况下，如果资源被删除之后，我们虽然能够被触发删除事件，但是这个时候从 Cache 里面无法读取任何被删除对象的信息，这样一来，导致很多垃圾清理工作因为信息不足无法进行，K8s 的 Finalizer 字段用于处理这种情况。在 K8s 中，只要对象 ObjectMeta 里面的 Finalizers 不为空，对该对象的 delete 操作就会转变为 update 操作，具体说就是 update  deletionTimestamp 字段，其意义就是告诉 K8s 的 GC“在deletionTimestamp 这个时刻之后，只要 Finalizers 为空，就立马删除掉该对象”。

所以一般的使用姿势就是在创建对象时把 Finalizers 设置好（任意 string），然后处理 DeletionTimestamp 不为空的 update 操作（实际是 delete），根据 Finalizers 的值执行完所有的 pre-delete hook（此时可以在 Cache 里面读取到被删除对象的任何信息）之后将 Finalizers 置为空即可。
#### 2.2.10 OwnerReference

K8s GC 在删除一个对象时，任何 ownerReference 是该对象的对象都会被清除，与此同时，Kubebuidler 支持所有对象的变更都会触发 Owner 对象 controller 的 Reconcile 方法。


## 三 实战kubebuilder
### 3.1 需求环境
* go version v1.15+.
* docker version 17.03+.
* kubectl version v1.11.3+.
* kustomize v3.1.0+
* Kind 本地开发可安装kind

能够访问 Kubernetes v1.11.3+ 集群


* 环境
创建:k8s.io和sigs.k8s.io两个目录
    * k8s.io直接拉去k8s项目源码，并将其中kubernetes/staging/src/k8s.io拷贝出来即可
    * sigs.k8s.io 需要创建目录，clone k8s-sigs/controller-runtime项目


```
# 在gopath的src目录下拉取k8s源码
$ pwd
/Users/xuel/workspace/goworkspace/src
$ ls
github.com        golang.org        google.golang.org gopkg.in
# 拷贝k8s源码中的k8s.io到上层目录
$ cp -R kubernetes/staging/src/k8s.io k8s.io

# 在gopath 中创建sigs.k8s.io,并clone controller-runtime
$ mkdir /Users/xuel/workspace/goworkspace/src/sigs.k8s.io && cd sigs.k8s.io && git clone https://github.com/kubernetes-sigs/controller-runtime.git

$ ls
drwxr-xr-x  24 xuel  staff   768B Jan  3 18:46 github.com
drwxr-xr-x   3 xuel  staff    96B Mar 22  2020 golang.org
drwxr-xr-x   3 xuel  staff    96B May 21  2020 google.golang.org
drwxr-xr-x   3 xuel  staff    96B May 21  2020 gopkg.in
drwxr-xr-x  28 xuel  staff   896B Jan 28 19:53 k8s.io
drwxr-xr-x  41 xuel  staff   1.3K Jan 28 19:52 kubernetes
drwxr-xr-x   3 xuel  staff    96B Jan 28 19:57 sigs.k8s.io
```


### 3.2 创建项目
* 创建目录，初始化系统
```
$ kubebuilder init --domain imoc-operator
```
目录结构如下：
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9cb22643bbe740fc99742b9a4a520eec~tplv-k3u1fbpfcp-zoom-1.image)


### 3.3 创建API
运行下面的命令，创建一个新的 API（组/版本）为 “webapp/v1”，并在上面创建新的 Kind(CRD) “Guestbook”。
```
kubebuilder create api --group webapp --version v1 --kind Guestbook

```
如果你在 Create Resource [y/n] 和 Create Controller [y/n] 中按y，那么这将创建文件 api/v1/guestbook_types.go ，该文件中定义相关 API ，而针对于这一类型 (CRD) 的对账业务逻辑生成在 controller/guestbook_controller.go 文件中。
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/62d086b6476143cb8ec4e13e3edfc146~tplv-k3u1fbpfcp-zoom-1.image)


### 3.4 编译运行controller
kubebuilder自动生成的controller源码地址是：`$GOPATH/src/helloworld/controllers/guestbook_controller.go` ， 内容如下：
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7c7b8d8691ea467685a459471d028251~tplv-k3u1fbpfcp-zoom-1.image)

### 3.5 安装crd到集群
* 将 CRD 安装到集群中
```
make install
```
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d09c461763cf474e94f20a65c5590b51~tplv-k3u1fbpfcp-zoom-1.image)
* 运行控制器（这将在前台运行，如果你想让它一直运行，请切换到新的终端）。
```
make run
```
此时控制台输出以下内容，这里要注意，controller是在kubebuilder电脑上运行的，一旦使用Ctrl+c中断控制台，就会导致controller停止：
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4a617221af8c4955b84da70519837165~tplv-k3u1fbpfcp-zoom-1.image)
### 3.6 创建Guestbook资源的实例

* 现在kubernetes已经部署了Guestbook类型的CRD，而且对应的controller也已正在运行中，可以尝试创建Guestbook类型的实例了(相当于有了pod的定义后，才可以创建pod)；
* kubebuilder已经自动创建了一个类型的部署文件：$GOPATH/src/helloworld/config/samples/webapp_v1_guestbook.yaml ，内容如下，很简单，接下来咱们就用这个文件来创建Guestbook实例：
```yaml
apiVersion: webapp.com.bolingcavalry/v1
kind: Guestbook
metadata:
  name: guestbook-sample
spec:
  # Add fields here
  foo: bar
```
#### 3.6.1 安装pod
```shell
$ kubectl apply -f config/samples/
$ kubectl get Guestbook
```
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d546784e4d2d4d12a0fcb26ace874c4b~tplv-k3u1fbpfcp-zoom-1.image)
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2ae2f6ce645f4df0829bd5dfc3bb9274~tplv-k3u1fbpfcp-zoom-1.image)

### 3.7 将controller制作为docker镜像
1. 至此，咱们已经体验过了kubebuilder的基本功能，不过实际生产环境中controller一般都会运行在kubernetes环境内，像上面这种运行在kubernetes之外的方式就不合适了，咱们来试试将其做成docker镜像然后在kubernetes环境运行；
2. 这里有个要求，就是您要有个kubernetes可以访问的镜像仓库，例如局域网内的Harbor，或者公共的hub.docker.com，我这为了操作方便选择了hub.docker.com，使用它的前提是拥有hub.docker.com的注册帐号；
3. 在kubebuilder电脑上，打开一个控制台，执行docker login命令登录，根据提示输入hub.docker.com的帐号和密码，这样就可以在当前控制台上执行docker push命令将镜像推送到hub.docker.com上了（这个网站的网络很差，可能要登录好几次才能成功）；
4. 执行以下命令构建docker镜像并推送到hub.docker.com，镜像名为bolingcavalry/guestbook:002：
```shell
make docker-build docker-push IMG=bolingcavalry/guestbook:002
```
5. hub.docker.com的网络状况不是一般的差，kubebuilder电脑上的docker一定要设置镜像加速，上述命令如果遭遇超时失败，请重试几次，此外，构建过程中还会下载诸多go模块的依赖，也需要您耐心等待，也很容易遇到网络问题，需要多次重试，所以，最好是使用局域网内搭建的Habor服务；
6. 最终，命令执行成功后输出如下：
```shell
[root@kubebuilder helloworld]# make docker-build docker-push IMG=bolingcavalry/guestbook:002
```
build会链接国外网站，需要注意翻墙
镜像推送上去后，可以查看镜像信息
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/344f97e06b934607b0fcd21090c1ba99~tplv-k3u1fbpfcp-zoom-1.image)
```shell
$  make docker-build docker-push IMG=127.0.0.1:5000/guesstbook:v1
/Users/xuel/workspace/goworkspace/src/github.com/kaliarch/imoc-operator/bin/controller-gen object:headerFile="hack/boilerplate.go.txt" paths="./..."
go fmt ./...
go vet ./...
/Users/xuel/workspace/goworkspace/src/github.com/kaliarch/imoc-operator/bin/controller-gen "crd:trivialVersions=true,preserveUnknownFields=false" rbac:roleName=manager-role webhook paths="./..." output:crd:artifacts:config=config/crd/bases
mkdir -p /Users/xuel/workspace/goworkspace/src/github.com/kaliarch/imoc-operator/testbin
test -f /Users/xuel/workspace/goworkspace/src/github.com/kaliarch/imoc-operator/testbin/setup-envtest.sh || curl -sSLo /Users/xuel/workspace/goworkspace/src/github.com/kaliarch/imoc-operator/testbin/setup-envtest.sh https://raw.githubusercontent.com/kubernetes-sigs/controller-runtime/v0.7.0/hack/setup-envtest.sh
source /Users/xuel/workspace/goworkspace/src/github.com/kaliarch/imoc-operator/testbin/setup-envtest.sh; fetch_envtest_tools /Users/xuel/workspace/goworkspace/src/github.com/kaliarch/imoc-operator/testbin; setup_envtest_env /Users/xuel/workspace/goworkspace/src/github.com/kaliarch/imoc-operator/testbin; go test ./... -coverprofile cover.out
fetching envtest tools@1.19.2 (into '/Users/xuel/workspace/goworkspace/src/github.com/kaliarch/imoc-operator/testbin')
x bin/
x bin/etcd
x bin/kubectl
x bin/kube-apiserver
setting up env vars
?       github.com/kaliarch/imoc-operator       [no test files]
?       github.com/kaliarch/imoc-operator/api/v1        [no test files]
ok      github.com/kaliarch/imoc-operator/controllers   12.959s coverage: 0.0% of statements
docker build -t 127.0.0.1:5000/guesstbook:v1 .
Sending build context to Docker daemon  335.5MB
Step 1/14 : FROM golang:1.15 as builder
1.15: Pulling from library/golang
b9a857cbf04d: Pull complete 
d557ee20540b: Pull complete 
3b9ca4f00c2e: Pull complete 
667fd949ed93: Pull complete 
547cc43be03d: Pull complete 
0977886e8147: Pull complete 
cceccf7c7738: Pull complete 
Digest: sha256:de97bab9325c4c3904f8f7fec8eb469169a1d247bdc97dcab38c2c75cf4b4c5d
Status: Downloaded newer image for golang:1.15
 ---> 5f46b413e8f5
Step 2/14 : WORKDIR /workspace
 ---> Running in 597efa584096
Removing intermediate container 597efa584096
 ---> a21979056316
Step 3/14 : COPY go.mod go.mod
 ---> b6c4b03d5126
Step 4/14 : COPY go.sum go.sum
 ---> f1af7c95cdc8
Step 5/14 : RUN go mod download
 ---> Running in baf57375b805
Removing intermediate container baf57375b805
 ---> 62e488ee06f5
Step 6/14 : COPY main.go main.go
 ---> 72c3d023e770
Step 7/14 : COPY api/ api/
 ---> b164eb864a85
Step 8/14 : COPY controllers/ controllers/
 ---> 843af6a782ec
Step 9/14 : RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 GO111MODULE=on go build -a -o manager main.go
 ---> Running in af2881daee7c
Removing intermediate container af2881daee7c
 ---> cf6ef1542da6
Step 10/14 : FROM gcr.io/distroless/static:nonroot
nonroot: Pulling from distroless/static
9e4425256ce4: Pull complete 
Digest: sha256:b89b98ea1f5bc6e0b48c8be6803a155b2a3532ac6f1e9508a8bcbf99885a9152
Status: Downloaded newer image for gcr.io/distroless/static:nonroot
 ---> 88055b6758df
Step 11/14 : WORKDIR /
 ---> Running in 35900ca6d19f
Removing intermediate container 35900ca6d19f
 ---> 902a3991fa3b
Step 12/14 : COPY --from=builder /workspace/manager .
 ---> 5af066bf1214
Step 13/14 : USER 65532:65532
 ---> Running in b44fbfb3c52b
Removing intermediate container b44fbfb3c52b
 ---> 6ca11554d8fa
Step 14/14 : ENTRYPOINT ["/manager"]
 ---> Running in 716538bf799a
Removing intermediate container 716538bf799a
 ---> a98e090c1e68
Successfully built a98e090c1e68
Successfully tagged 127.0.0.1:5000/guesstbook:v1
docker push 127.0.0.1:5000/guesstbook:v1
The push refers to repository [127.0.0.1:5000/guesstbook]
babc932481e7: Pushed 
8651333b21e7: Pushed 
v1: digest: sha256:5dbb4e549c0dff1a4edba19ea5a35f9e21deeabe2fcefbc6b6358fb849dd61e2 size: 739
$ curl -XGET http://127.0.0.1:5000/v2/_catalog                   
{"repositories":["guesstbook"]}

```

7. 部署controller 镜像到集群内
```shell
$ make deploy IMG=127.0.0.1:5000/guesstbook:v1
/Users/xuel/workspace/goworkspace/src/github.com/kaliarch/imoc-operator/bin/controller-gen "crd:trivialVersions=true,preserveUnknownFields=false" rbac:roleName=manager-role webhook paths="./..." output:crd:artifacts:config=config/crd/bases
cd config/manager && /Users/xuel/workspace/goworkspace/src/github.com/kaliarch/imoc-operator/bin/kustomize edit set image controller=127.0.0.1:5000/guesstbook:v1
/Users/xuel/workspace/goworkspace/src/github.com/kaliarch/imoc-operator/bin/kustomize build config/default | kubectl apply -f -
namespace/imoc-operator-system created
customresourcedefinition.apiextensions.k8s.io/guestbooks.webapp.com.bolingcavalry configured
role.rbac.authorization.k8s.io/imoc-operator-leader-election-role created
clusterrole.rbac.authorization.k8s.io/imoc-operator-manager-role created
clusterrole.rbac.authorization.k8s.io/imoc-operator-metrics-reader created
clusterrole.rbac.authorization.k8s.io/imoc-operator-proxy-role created
rolebinding.rbac.authorization.k8s.io/imoc-operator-leader-election-rolebinding created
clusterrolebinding.rbac.authorization.k8s.io/imoc-operator-manager-rolebinding created
clusterrolebinding.rbac.authorization.k8s.io/imoc-operator-proxy-rolebinding created
configmap/imoc-operator-manager-config created
service/imoc-operator-controller-manager-metrics-service created
deployment.apps/imoc-operator-controller-manager created


```
* 查看部署进集群的控制器
查看镜像拉取异常，
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f76edee15498427db36a765470996248~tplv-k3u1fbpfcp-zoom-1.image)

* 在此使用阿里云镜像仓库

```shell
$ sudo docker login --username=1123845260@qq.com registry.cn-shanghai.aliyuncs.com
$ sudo docker pull registry.cn-shanghai.aliyuncs.com/kaliarch/slate:[镜像版本号]

$ sudo docker login --username=1123845260@qq.com registry.cn-shanghai.aliyuncs.com
$ sudo docker tag [ImageId] registry.cn-shanghai.aliyuncs.com/kaliarch/slate:[镜像版本号]
$ sudo docker push registry.cn-shanghai.aliyuncs.com/kaliarch/slate:[镜像版本号]

1. 登陆阿里云镜像仓库
密码：阿里控制台账号

2. 修改tag
docker tag 127.0.0.1:5000/guesstbook:v1 registry.cn-shanghai.aliyuncs.com/kaliarch/guesstbook:v1
3. 推送镜像
docker push registry.cn-shanghai.aliyuncs.com/kaliarch/guesstbook:v1
```
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/70bb20e32373416c9e6b50534727585f~tplv-k3u1fbpfcp-zoom-1.image)


重新部署
```shell
make deploy IMG=registry.cn-shanghai.aliyuncs.com/kaliarch/guesstbook:v1
```
查看已经成功运行
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e118b9896e6f4f0cbf80de1c8d0d70fe~tplv-k3u1fbpfcp-zoom-1.image)

8. 查看详细信息
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8a93ebaa3a2b4d32ba05d98026723141~tplv-k3u1fbpfcp-zoom-1.image)
显示这个pod实际上有两个容器，用kubectl describe命令细看，分别是kube-rbac-proxy和manager

9. 查看日志

```shell
kubectl logs -f imoc-operator-controller-manager-648b4877c6-4bpp9 -n imoc-operator-system  -c manager

```

## 四 清理
### 4.1 清理kubebuilder
```shell
make uninstall
```

### 4.2 清理kind
```shell
kind delete cluster
```

## 五 其他

通过了解，我们可以看到 Kubebuilder 提供的功能对于快速编写 CRD 和 Controller 是十分有帮助的，无论是云原生中知名的开源项目例如 Istio、Knative 等知名项目还是各种自定义 Operators，都大量使用了 CRD，将各种组件抽象为 CRD，据此可以将为自己业务编写对应的CRD，使得业务更加生于云长于云。

## 参考链接
* https://cloudnative.to/kubebuilder/quick-start.html
* https://www.cnblogs.com/alisystemsoftware/p/11580202.html
* https://blog.csdn.net/boling_cavalry/article/details/113089414
* https://zhuanlan.zhihu.com/p/83957726 