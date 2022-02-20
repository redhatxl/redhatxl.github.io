# Velero 二次开发

## 一 准备工作

### 1.1 开始之前

* 熟悉行为准则[Code of Conduct](https://github.com/vmware-tanzu/velero/blob/v1.7.0/CODE_OF_CONDUCT.md)。 
* 参阅 [CONTRIBUTING.md](https://github.com/vmware-tanzu/velero/blob/v1.7.0/CONTRIBUTING.md)以获取开发准则。

### 1.2 创建设计文档

拥有一个高层次的设计文档，其中包含建议的变更和影响，可以帮助维护人员评估是否应该合并一个主要变更。要发出设计拉请求，可以将 design/_ template.md 文件中的模板复制到新的 Markdown 文件中。

### 1.3 寻找路径

你可以加入 Velero 社区，以不同的方式做出贡献，包括帮助我们设计或测试新功能。对于任何我们考虑添加的重要特性，我们从设计文档开始。你可以在这里找到一个正在进行中的新设计的列表: https://github.com/vmware-tanzu/velero/pulls?q=is%3aopen+is%3apr+label%3adesign。请随时回顾并帮助我们完成您的输入。您还可以使用: + 1: 和:-1: 对问题进行投票，正如我们的功能增强请求和 Bug 问题模板中所解释的那样。这将帮助我们量化问题的重要性并对其进行优先排序。关于如何与我们的维护者和社区联系的信息，参加我们的在线会议，或者找到好的第一个问题，可以从我们的 Velero 社区页面开始。请浏览我们的资源列表，包括过去的在线社区会议、博客文章和其他资源的播放列表，以帮助您熟悉我们的项目: Velero 资源。

## 二 开始开发

### 2.1 更新生成文件

会pull一个velero/build-image 的镜像

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211022135127.png)

如果您做了以下更改，请运行 make update 以重新生成文件:

- Add/edit/remove command line flags and/or their help text
- Add/edit/remove commands or subcommands
- Add new API types
- Add/edit/remove plugin protobuf message or service definitions

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211022130941.png)

下面的文件是从源代码自动生成的:

- The clientset
- Listers
- Shared informers
- Documentation
- Protobuf/gRPC types

您可以运行 make verify 以确保所有生成的文件（clientset、listers、shared informers、docs）都是最新的。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211022135610.png)

## 2.2 Linting

您可以运行 make lint 来执行构建映像中的 golangci-lint，或者执行构建映像外部的 local-lint。两者都使 lint 和使局部-lint 将只运行临时对变化。

使用 lint-all 对整个代码库运行 linter。 默认的 linter 是通过 LINTERS 变量在 Makefile 中定义的。 您还可以通过运行命令来覆盖默认的 linter 列表 

```shell
$ make lint LINTERS=gosec
```

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211022140830.png)

## 2.3 单元测试

```shell
make test
```

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211022142653.png)

## 2.4 vendor 依赖

如果您需要添加或更新供应商依赖项，请参阅供应商：[Vendoring dependencies](https://velero.io/docs/v1.7/vendoring-dependencies/).

## 2.5 使用主分支

如果您正在开发或使用主分支，请注意您可能需要更新 Velero CRD 以在其他开发工作完成时获得新的更改。

```bash
$ velero install --crds-only --dry-run -o yaml | kubectl apply -f -
```

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211022145350.png)

注意: 如果 Velero CLI 无法发现 Kubernetes 首选的 CRD API 版本，则可以更改默认的 CRD API 版本(v1beta1或 v1)。Kubernetes 版本 < 1.16的首选 CRD API 版本是 v1beta1; 

Kubernetes 版本 > = 1.16的首选 CRD API 版本是 v1。

# 三 使用 Tilt 进行快速迭代 Velero 开发

## 3.1 概要

本文档描述了如何将 [Tilt](https://tilt.dev/) 与任何集群一起使用，以实现简化的工作流，从而提供简单的部署和快速的迭代构建。这种设置允许继续部署 Velero 服务器，如果指定，则允许任何提供程序插件或 restic daemonset。它通过以下方式完成这项工作:

* 部署必要的 Kubernetes 资源，例如 Velero CRD 和 Velero 部署
* 为 Velero 和（如果指定）提供程序插件构建本地二进制文件作为 local_resource
* 调用 docker_build 将任何二进制文件实时更新到容器/初始化容器中并触发重新启动

Tilt 会在 velero/tilt-resources 下寻找配置文件。此目录中的大多数文件都被 gitignored 忽略，因此您可以根据需要配置您的设置。

## 3.2 先决条件

1. Docker v19.03或更新的 
2. Kubernetes cluster v1.10或更高版本(不一定是 Kind) 
3. Tilt v0.12.0或更新版本 
4. 在本地克隆 Velero 项目存储库
5. 访问 S3 对象存储
6. 克隆您想要更改和部署的任何提供程序插件 (可选的，必须配置为由 Velero Tilt 的设置部署，更多信息见下)

注意: 要正确配置你使用的任何插件，请参考插件文档。

## 3.3 入门

### 3.3.1 配置

* 将 velero/tilt-resources/examples 下的所有示例文件复制到 velero/tilt-resources 中。
* 配置 velero_v1_backupstoragelocation.yaml 文件和存储凭证/密钥的云文件。
* 运行`tilt up`

### 3.3.2 创建 tilt 设置文件

创建一个名为tilt-settings.json 的配置文件，并将其放在velero/tilt-resources 的本地副本中。或者，您可以复制并粘贴在 velero/tilt-resources/examples 中找到的示例文件。

例如：

```json
{
    "default_registry": "",
    "enable_providers": [
        "aws",
        "gcp",
        "azure",
        "csi"
    ],
    "providers": {
        "aws": "../velero-plugin-for-aws",
        "gcp": "../velero-plugin-for-gcp",
        "azure": "../velero-plugin-for-microsoft-azure",
        "csi": "../velero-plugin-for-csi"
    },
    "allowed_contexts": [
        "development"
    ],
    "enable_restic": false,
    "create_backup_locations": true,
    "setup-minio": true,
    "enable_debug": false,
    "debug_continue_on_start": true
}
```

### 3.3.3 tilt-settings.json 字段

* **default_registry**：(String, default=”")，如果你需要推送镜像到镜像仓库需要改字段，更详细可参考：[Tilt *documentation](https://docs.tilt.dev/api.html#api.default_registry)
* **provider_repos**：(Array[]String, default=[])，您要更改的所有提供程序插件的路径列表。每个提供者必须有一个tilt-provider.json 文件来描述如何构建提供者。
* **enable_providers** (Array[]String, default=[]):要启用的提供程序插件列表。更多详细信息请参阅提供程序插件。注意: 当不对插件进行更改时，不需要将它们加载到 Tilt 中: 可以在 Velero 部署中指定一个现有的映像和版本，Tilt 将加载这些内容。

* **allowed_contexts** (Array, default=[]):允许 Tilt 使用的 kubeconfig 上下文列表。有关更多详细信息，请参阅关于 *allow_k8s_contexts 的 Tilt 文档。注意： Kind 是自动允许的。
* **enable_restic** (Bool, default=false):指示是否部署restic Daemonset。如果设置为 true，Tilt 将查找包含 Velero restic DaemonSet 配置的 velero/tilt-resources/restic.yaml 文件。
* **create_backup_locations** (Bool, default=false):指示是否创建一个或多个备份存储位置。如果设置为 true，Tilt 将查找 velero/tilt-resources/velero_v1_backupstoragelocation.yaml 文件，其中至少包含一个 Velero 备份存储位置的配置。
* **setup-minio** (Bool, default=false):如果要在集群内运行的 Minio 实例中配置备份存储位置，请将此配置为 true。

* **enable_debug** (Bool, default=false):如果要使用 [Delve](https://github.com/go-delve/delve) 调试 velero 进程，请将此配置为 true。
* **debug_continue_on_start** (Bool, default=true):如果您希望 velero 进程在调试模式下在启动时继续，请将此配置为 true。请参阅  [Delve CLI documentation](https://github.com/go-delve/delve/blob/master/Documentation/usage/dlv.md) 文档。

## 3.4 创建要部署的 Kubernetes 资源文件

在 velero/tilt-resources/examples 目录中，提供了所有需要的 Kubernetes 资源文件，以便随时使用示例。你只需要将它们移动到 velero/tilt-resources 级别。

因为 Velero Kubernetes 的部署和恢复的 DaemonSet 包含所有要使用的插件的配置，所以这些资源的文件应该由用户提供，因此您可以选择加载哪个提供者插件作为 init 容器。目前，所提供的样例文件已经配置了 Velero 支持的所有插件，可以根据需要随意删除任何插件。

为了使 Velero 完全运行，它还需要至少一个备份存储位置。提供了一个示例文件，需要使用对象存储的特定配置对其进行修改。请参阅下一小节以获得更多关于此的详细信息。

## 3.5 配置一个backup storage location

您必须配置 velero/tilt-resources/velero _ v1 _ backupstoragellocation。根据您的存储提供程序使用适当的值。参考[plugin documentation](https://velero.io/plugins/)，了解特定提供者的备份存储位置配置需要哪些字段/值对。

下面是为 Velero 配置备份存储位置的一些方法。

### 3.5.1 使用公有云存储

按照提供程序文档来配置存储。我们有一个所有已知对象存储提供程序的列表，以及相应的 Velero 插件。[list of all known object storage providers](https://velero.io/docs/v1.5/supported-providers/) 

### 3.5.2 使用MinIO作为对象存储

注意：要将 MinIO 用作对象存储，您需要使用 [`AWS` plugin](https://github.com/vmware-tanzu/velero-plugin-for-aws)，并将存储位置配置为将 spec.provider 设置为 aws，将 spec.config.region 设置为 minio。例子：

```yaml
spec:
  config:
    region: minio
    s3ForcePathStyle: "true"
    s3Url: http://minio.velero.svc:9000
  objectStorage:
    bucket: velero
  provider: aws
```

以下是使用 MinIO 作为存储的两种方法:

1. 作为在集群内运行的 MinIO 实例（不要在生产环境中这样做！）

在tilt-settings.json 文件中，设置“setup-minio”：true。这将配置一个 Kubernetes 部署，其中包含集群内正在运行的 MinIO 实例。在集群外公开 MinIO 需要额外的步骤( [extra steps](https://velero.io/docs/v1.5/contributions/minio/#expose-minio-outside-your-cluster-with-a-service))。

要访问这个存储，您需要使用 kubectl port-forward-n svc/MinIO 9000将 MinIO 端口转发到本地计算机，从而在集群之外暴露 MinIO。通过在 BSL 配置中添加 publicUrl: http://localhost:9000，更新 BSL 配置，将其作为其“公共 URL”。这对于下载备份文件是必要的。

注意：使用此设置，当您的集群终止时，其中的存储和任何备份/恢复也会终止。

2. 作为在 Docker 容器中本地运行的独立 MinIO 实例 请参阅[these instructions](https://github.com/vmware-tanzu/velero/discussions/3381)以在您的计算机上本地运行 MinIO，作为独立的而不是在 Pod 上运行它。

请参阅我们的[locations documentation](https://velero.io/docs/v1.5/locations/)以了解更多备份位置的工作原理。

## 3.6 配置提供者凭据（secret）

无论您使用什么对象存储提供程序，都要在 velero/tilt-resources/cloud 文件中配置凭据。阅读插件文档，了解提供者的凭据需要哪些字段/值对。Tilt 文件将调用 Kustomize 在硬编码的密钥 secret.cloud-credentials 下创建秘密。名称空间中的 data.cloud。在 velero/tilt-resources/examples/cloud 中，有一个为 MinIO 存储凭证正确格式化的示例凭证文件。

## 3.7 使用 Delve 配置调试

如果您希望调试 Velero 进程，可以通过将字段 enable _ debug 设置为 true 在 tilt-resources/tile-settings 中启用调试模式。Json 文件。这将使您能够使用深入来调试进程。通过启用调试模式，将在调试模式下构建 Velero 可执行文件(使用 flags-gcflags = “-n-l”禁用优化和内联) ，并在使用 [`dlv exec`](https://github.com/go-delve/delve/blob/master/Documentation/usage/dlv_exec.md)的 Velero 部署中启动该进程。

调试服务器将接受端口2345上的连接，而 Tilt 被配置为将该端口转发到本地计算机。一旦 Tilt 运行并且 Velero 资源准备就绪，您就可以连接到调试服务器开始调试。要连接到会话，可以通过在本地运行 dlv connect 127.0.0.1:2345来使用深入的 CLI。有关如何使用深入研究的更多指导，请参阅 [Delve CLI documentation](https://github.com/go-delve/delve/tree/master/Documentation/cli)。钻研也可以在许多[editors and IDEs](https://github.com/go-delve/delve/blob/master/Documentation/EditorIntegration.md).

默认情况下，当处于调试模式时，Velero 进程将在启动时继续。这意味着进程将一直运行，直到设置了断点。可以通过在倾斜资源/平铺设置中将字段 debug _ continue _ on _ start 设置为 false 来禁用此选项。Json 文件。当此设置被禁用时，Velero 进程将不会继续运行，直到通过您的深入会话发出一个 continue 指令。

退出调试会话时，CLI 和编辑器集成通常会询问是否应该停止远程进程。保持远程进程运行并断开与调试会话的连接非常重要。通过停止远程进程，这将导致 Velero 容器停止和荚重新启动。如果正在进行备份，那么这些备份将处于不新鲜状态，因为在 Velero pod 重新启动时它们不会恢复。

## 3.8 运行Tilt

要启动您的开发环境，请运行：

```bash
tilt up
```

这将输出地址到一个网络浏览器界面，在那里你可以监视倾斜的状态和每个倾斜资源的日志。经过短暂的时间后，您应该已经有了一个正在运行的开发环境，现在应该能够创建备份/恢复并完全操作 Velero 了。

注意：退出 Tilt 后运行倾斜向下将 Tiltfile 中指定的[will delete all resources](https://docs.tilt.dev/cli/tilt_down.html) 

提示：为 velero/_tuiltbuild/local/velero 创建别名，您不必运行 make local 来获得 Velero CLI 的更新版本，只需使用别名即可。

请参阅文档[how Velero works](https://velero.io/docs/v1.5/how-velero-works/).

## 3.9 提供插件

提供者必须提供一个tilt-provider.json 文件来描述如何构建它。下面是一个例子：

```json
{
  "plugin_name": "velero-plugin-for-aws",
  "context": ".",
  "image": "velero/velero-plugin-for-aws",
  "live_reload_deps": [
    "velero-plugin-for-aws"
  ],
  "go_main": "./velero-plugin-for-aws"
}
```

## 3.10 实时更新

每个配置为由 Velero 的 Tilt 设置部署的提供者插件都有一个 live _ reload_deps 列表。这定义了 Tilt 应该监视的文件和/或目录的更改。修改依赖项后，Tilt 在本地计算机上重新构建提供程序的二进制文件，将二进制文件复制到 init 容器，并触发 Velero 容器的重新启动。这比为每次更改重新生成容器映像要快得多。它还有助于保持每个开发映像的大小尽可能小(容器映像不需要整个 go 工具链、源代码、模块依赖性等)。

# 四 从源代码构建

## 4.1 先决条件

* 访问 Kubernetes 集群，版本 1.7 或更高版本。 
* 集群上的 DNS 服务器 
* 安装了 kubectl 
* Go 安装（最低版本 1.8）

## 4.2 获取源码

* 从最新源码获取

```shell
mkdir $HOME/go
export GOPATH=$HOME/go
go get github.com/vmware-tanzu/velero	
```

哪里是 Go 的导入路径。 

对于 Go 开发，建议将 Go 导入路径（本例中为 $HOME/go）添加到您的路径中。

* 制定版本拉取

从[release page](https://github.com/vmware-tanzu/velero/releases)下载名为 Source code 的归档文件，并将其作为 src/ github.com/vmware-tanzu/velero 

文件在 Go 导入路径中解压缩。注意，Makefile 目标假定从 git 存储库构建。当从存档构建时，您只能使用下面描述的 go build 命令。

## 4.3 构建

根据您的需要，有许多不同的方法来建立 velero。这一节概述了主要的可能性。当使用 make 进行构建时，它将把二进制文件放在` _ output/bin/$GOOS/$GOARCH `下。例如，您可以在这里找到 darwin 的二进制文件: _ output/bin/darwin/amd64/velero，以及 linux 的二进制文件:` _ output/bin/linux/amd64/velero`。Make 还将拼接版本和 git 提交信息，以便 velero 版本显示适当的输出。

注意: velero install 还将使用版本信息来确定要部署哪个标记的映像。如果您想覆盖部署的映像，请使用映像标志(参见下面关于如何构建映像的说明)。

## 4.4 构建二进制

要在本地机器上构建为操作系统和架构编译的 velero 二进制文件，请运行以下两个命令之一:

```bash
go build ./cmd/velero
make local
```

## 4.5 交叉编译

要在本地机器上的构建容器中构建针对 linux/amd64 的 velero 二进制文件，请运行：

对于任何特定平台，运行 make build-<GOOS>-<GOARCH>。 例如，要为 Mac 构建，请运行 make build-darwin-amd64。 Velero 的 Makefile 有一个方便的目标，all-build，它构建以下平台：

- linux-amd64
- linux-arm
- linux-arm64
- linux-ppc64le
- darwin-amd64
- windows-amd64

## 4.6 编译镜像和升级velero

如果在安装 Velero 后，您想将其部署使用的映像更改为包含您的代码更改的映像，您可以通过更新映像来实现：

```bash
kubectl -n velero set image deploy/velero velero=myimagerepo/velero:$VERSION
```

要构建 Velero 容器映像，您需要先配置 buildx。

## 4.7 Buildx

Docker Buildx 是一个 CLI 插件，它通过 Moby BuildKit builder 工具包提供的特性来扩展 Docker 命令。它提供了与 docker 构建相同的用户体验，并具有许多新特性，比如创建有作用域的构建实例和并发构建多个节点。

更多信息可参考： [docker docs](https://docs.docker.com/buildx/working-with-buildx/) and in the [buildx github](https://github.com/docker/buildx) repo.

## 4.8 镜像构建

设置 \$REGISTRY 环境变量。例如，如果你想创建 gcr.io/my-REGISTRY/velero:main 图像，将 \$REGISTRY 设置为 gcr.io/my-REGISTRY。如果未设置此变量，则默认值为 velero。

可以选择，设置 ​\$VERSION 环境变量来更改图像标记或 $BIN 来更改要为其构建容器图像的二进制文件。然后，跑:

注意：要为 velero 和 velero-restic-restore-helper 构建构建容器镜像，请运行：make all-containers

## 4.9 将容器映像发布到注册表

要将容器映像发布到注册表，需要进行以下一次性设置：

1. 如果您正在构建跨平台容器映像

1. ```bash
   $ docker run --rm --privileged multiarch/qemu-user-static --reset -p yes
   ```

2. 创建并引导一个新的 docker buildx 构建器

```bash
$ docker buildx create --use --name builder
  builder
$ docker buildx inspect --bootstrap
  [+] Building 2.6s (1/1) FINISHED
  => [internal] booting buildkit                                2.6s
  => => pulling image moby/buildkit:buildx-stable-1             1.9s
  => => creating container buildx_buildkit_builder0             0.7s
Name:   builder
Driver: docker-container

Nodes:
Name:      builder0
Endpoint:  unix:///var/run/docker.sock
Status:    running
Platforms: linux/amd64, linux/arm64, linux/ppc64le, linux/s390x, linux/386, linux/arm/v7, linux/arm/v6
```

注意：如果没有上述设置， docker buildx inspect --bootstrap 的输出将是：

```bash
$ docker buildx inspect --bootstrap
Name:   default
Driver: docker

Nodes:
Name:      default
Endpoint:  default
Status:    running
Platforms: linux/amd64, linux/arm64, linux/ppc64le, linux/s390x, linux/386, linux/arm/v7, linux/arm/v6
```

并且 REGISTRY=myrepo BUILDX_OUTPUT_TYPE=registry make 容器将失败并显示以下错误

```bash
$ REGISTRY=ashishamarnath BUILDX_PLATFORMS=linux/arm64 BUILDX_OUTPUT_TYPE=registry make container
auto-push is currently not implemented for docker driver
make: *** [container] Error 1
```

完成上述一次性设置后，现在 docker buildx inspect --bootstrap 的输出应该是这样的

```bash
$ docker buildx inspect --bootstrap
Name:   builder
Driver: docker-container

Nodes:
Name:      builder0
Endpoint:  unix:///var/run/docker.sock
Status:    running
Platforms: linux/amd64, linux/arm64, linux/riscv64, linux/ppc64le, linux/s390x, linux/386, linux/arm/v7, linux/arm/v
```

现在通过运行 make container 命令并将 $BUILDX_OUTPUT_TYPE 设置为 registry 来构建和推送容器镜像

```bash
$ REGISTRY=myrepo BUILDX_OUTPUT_TYPE=registry make container
```

## 4.10 跨平台编译

支持的 Docker buildx 平台：

- `linux/amd64`
- `linux/arm64`
- `linux/arm/v7`
- `linux/ppc64le`

对于任何特定平台，运行 BUILDX_PLATFORMS=<GOOS>/<GOARCH> make container 

例如，要为 arm64 构建映像，请运行：

```bash
BUILDX_PLATFORMS=linux/arm64 make container
```

注意：默认情况下，$BUILDX_PLATFORMS 设置为 linux/amd64 使用 buildx，您还可以同时构建所有支持的平台并将多架构映像推送到注册表。例如：

```bash
REGISTRY=myrepo VERSION=foo BUILDX_PLATFORMS=linux/amd64,linux/arm64,linux/arm/v7,linux/ppc64le BUILDX_OUTPUT_TYPE=registry make all-containers
```

注意: 当同时为多于1个平台构建时，您需要将 BUILDX _ output _ type 设置为注册表，因为本地多拱图像尚不受支持。注意: 如果你想更新图片但不改变它的名字，你必须触发 Kubernetes 选择新的图片。一种方法是删除 Velero 部署 pod:

```bash
kubectl -n velero delete pods -l deploy=velero
```



# 五 本地运行

在开发中本地运行 Velero

在本地运行 Velero 服务器可以加速迭代开发。这样就无需在每次更改时重建 Velero 服务器映像并将其重新部署到集群中。

## 5.1 使用远程集群在本地运行 Velero

Velero 以 Kubernetes API 服务器作为端点(根据 kubeconfig 配置)运行，因此 Velero 服务器和客户机使用相同的客户机-go 与 Kubernetes 进行通信。这意味着 Velero 服务器可以像在远程集群中运行一样在本地运行。

## 5.2 先决条件

在运行 Velero 时，您需要确保设置以下所有权限: 

* 从源集群和名称空间读取所有数据的访问权限
  * 对来自源集群和命名空间的所有数据的读取权限
  * 写访问目标集群和命名空间

* 写访问目标集群和命名空间
  * 对卷的读/写访问
  * 对备份数据的对象存储进行读/写访问
* [BackupStorageLocation](https://velero.io/docs/v1.5/api-types/backupstoragelocation/)Velero 服务器的对象定义
* 可选[VolumeSnapshotLocation](https://velero.io/docs/v1.5/api-types/volumesnapshotlocation/) Velero 服务器的对象定义，用于拍摄 PV 快照

## 5.3 步骤

### 5.3.1 安装velero

请参阅有关如何在某些特定提供商中安装 Velero 的文档：[Install overview](https://velero.io/docs/v1.5/basic-install/)

### 5.3.2 将部署规模缩小到零

使用 velero install 命令将 Velero 安装到集群中后，您将 Velero 部署缩小到 0，这样它就不会同时在远程集群上运行，并可能导致事情不同步：

```
kubectl scale --replicas=0 deployment velero -n velero
```

### 5.3.3 在本地启动Velero服务器

* 要在本地运行服务器，请根据所需的二进制文件使用完整路径。例如，如果您使用 Mac，并使用 AWS 作为提供程序，下面是如何使用完整路径运行从源代码构建的二进制文件: `AWS_SHARED_CREDENTIALS_FILE=<path-to-credentials-file> ./_output/bin/darwin/amd64/velero`.或者，您可以将 velero 二进制文件添加到 PATH 中。

* 启动服务器：velero 服务器 [CLI 标志]。以下 CLI 标志可能有助于自定义，但请参阅 velero server --help 以获取完整详细信息：
  * --log-level：设置 Velero 服务器的日志级别（默认信息，使用 debug 进行最多的日志记录）
  * --kubeconfig：设置 Velero 服务器用来与 Kubernetes apiserver 通信的 kubeconfig 文件的路径（默认为 $ KUBECONFIG）
  * --namespace：Velero 服务器应在其中查找备份、计划、恢复的集合命名空间（默认 velero）
  * --plugin-dir：设置Velero服务器查找插件的目录（默认/plugins）
    * --plugin-dir 标志要求插件二进制文件存在于本地，并应设置为包含此构建二进制文件的目录。
  * --metrics-address：设置暴露 Prometheus 指标的绑定地址和端口（默认：8085）

# 六 代码标准

## 6.1 打卡一个pr

当开启拉取请求时，请填写提供模板的清单。这将帮助其他人正确地分类和审查您的拉请求。

## 6.2 添加更新日志

作者需要在其拉取请求中包含一个更改日志文件。更改日志文件应该是在更改日志/未发布文件夹中创建的新文件。文件应该遵循 pr-用户名的变数命名原则，文件的内容应该是你的文本更新日志。

```
velero/changelogs/unreleased   <- folder
    000-username            <- file
```

将其添加到 PR 中。

如果 PR 不保证更改日志，则可以通过在 PR 上应用 changelog-not-required 标签来跳过对更改日志的 CI 检查。

## 6.3 版权头部

每当修改源代码文件时，版权声明应该更新为我们的标准版权声明。也就是说，它应该读作“ Velero 贡献者的版权”对于新文件，必须添加完整的版权和许可标题。请注意，文档文件不需要版权标题。

## 6.4 代码

* 日志消息大写。
* 错误消息保持小写。
* 仅对从非 velero 代码直接返回的错误（例如对 Kubernetes 服务器的 API 调用）包装/添加堆栈。

- ```bash
  errors.WithStack(err)
  ```

* 更喜欢使用 Kubernetes 包集中的实用程序。

- ```bash
  k8s.io/apimachinery/pkg/util/sets
  ```

## 6.5 导包

对于导入，我们使用以下约定：

```shell
<group><version><api | client | informer | ...>
```

Example

```go
import (
    corev1api "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    corev1client "k8s.io/client-go/kubernetes/typed/core/v1"
    corev1listers "k8s.io/client-go/listers/core/v1"
   
    velerov1api "github.com/vmware-tanzu/velero/pkg/apis/velero/v1"
    velerov1client "github.com/vmware-tanzu/velero/pkg/generated/clientset/versioned/typed/velero/v1"
)
```

## 6.6 Mocks

我们使用一个包来为我们的接口生成模拟。

示例：如果你想改变这个模拟:https://github.com/vmware-tanzu/velero/blob/v1.5.1/pkg/restic/mocks/restorer.go

运行：

```bash
go get github.com/vektra/mockery/.../
cd pkg/restic
mockery -name=Restorer
```

可能需要运行 make update 来更新导入。

## 6.7 Kubernetes 标签

在生成标签值时，请确保将它们传递到标签中。GetValidName () helper 函数。这将有助于确保值是要存储和查询的适当长度和格式。一般来说，uid 作为标签值持久存在是安全的。此函数与没有限制的注释值无关。

## 6.8 DCO Sign off

该项目的所有作者保留对其作品的版权。然而，为了确保他们只提交他们有权的作品，我们要求每个人通过签署他们的作品来承认这一点。

本 repo 中的任何版权声明都应将作者指定为“Velero 贡献者”。

要在你的作品上签名，只需要在提交消息的末尾加上这样一行:

```fallback
Signed-off-by: Joe Beda <joe@heptio.com>
```

这可以通过 git commit 的 --signoff 选项轻松完成。

通过这样做，您声明您可以证明以下内容：(from https://developercertificate.org/):

```fallback
Developer Certificate of Origin
Version 1.1

Copyright (C) 2004, 2006 The Linux Foundation and its contributors.
1 Letterman Drive
Suite D4700
San Francisco, CA, 94129

Everyone is permitted to copy and distribute verbatim copies of this
license document, but changing it is not allowed.


Developer's Certificate of Origin 1.1

By making a contribution to this project, I certify that:

(a) The contribution was created in whole or in part by me and I
    have the right to submit it under the open source license
    indicated in the file; or

(b) The contribution is based upon previous work that, to the best
    of my knowledge, is covered under an appropriate open source
    license and I have the right under that license to submit that
    work with modifications, whether created in whole or in part
    by me, under the same open source license (unless I am
    permitted to submit under a different license), as indicated
    in the file; or

(c) The contribution was provided directly to me by some other
    person who certified (a), (b) or (c) and I have not modified
    it.

(d) I understand and agree that this project and the contribution
    are public and that a record of the contribution (including all
    personal information I submit with it, including my sign-off) is
    maintained indefinitely and may be redistributed consistent with
    this project or the open source license(s) involved.
```

# 六 网站引导

## 6.1 本地运行

对网站进行更改时，请在提交 PR 之前在本地运行该网站并手动验证您的更改。 在项目的根目录，运行：

```bash
make serve-docs
```

这会在容器中运行所有 Hugo 依赖项。 或者，为了快速加载网站，在 velero/site/ 目录下运行：

```bash
hugo serve
```

有关如何在本地运行网站的更多信息，请参阅我们的 [Hugo documentation](https://gohugo.io/getting-started/)。

## 6.2 添加博客文章

要添加博客文章，请在 /site/content/posts/ 文件夹中创建一个新的 Markdown (.MD) 文件。一篇博客文章需要以下前言。

```yaml
title: "Title of the blog"
excerpt: Brief summary of thee blog post that appears as a preview on velero.io/blogs
author_name: Jane Smith
slug: URL-For-Blog
# Use different categories that apply to your blog. This is used to connect related blogs on the site
categories: ['velero','release']
# Image to use for blog. The path is relative to the site/static/ folder
image: /img/posts/example-image.jpg
# Tag should match author to drive author pages. Tags can have multiple values.
tags: ['Velero Team', 'Nolan Brubaker']
```

例如，在标签字段中包含 author_name 值，以便列出作者帖子的页面可以正常工作，https://velero.io/tags/carlisia-campos/.

理想情况下，每个博客都有一个独特的图像可以在博客主页上使用，但如果您不包含图像，则将使用默认的 Velero 徽标。使用小于 70KB 的图像并将其添加到 /site/static/img/posts 文件夹。



# 七 文档风格

参考：https://velero.io/docs/v1.5/style-guide/





# 参考链接

* https://github.com/tilt-dev/tilt
* https://tilt.dev/
* https://velero.io/docs/v1.5/tilt/







