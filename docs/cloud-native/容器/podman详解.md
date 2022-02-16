# Podman详解

## 一 背景



之前使用Docker，但是在一些场景Docker不是很适用，Docker是一个C/S架构，运行容器需要Daemon，但是一下简单测试或者CI/CD中，没有Daemon，或者没有root权限，此时就可以使用其他的一些遵循OCI接口规范的工具，例如红帽的podman，其是fork/exec模型，直接通过 OCI runtime（默认也是 `runc`）来启动容器，无需Daemon后台进程，所以利用podman启动的容器是podman的子进程，



## 二 概念

Podman 是 Libpod 的一部分，它的定义可以简单用这个命令表示：`alias docker=podman`

Libpod 是一个创建容器 pod 的工具和库，它**包含 pod 管理工具 Podman**，Podman 管理 pod、容器、容器镜像和容器卷。

在较高的层面上，Libpod 和 Podman 的作用范围如下：

- 支持多种镜像格式，包括 OCI 和 Docker。
- 支持多种方式下载镜像，包括信任和镜像验证。
- 容器镜像管理，管理镜像层、覆盖文件系统等。
- 全面管理容器生命周期。
- 支持 pod 管理容器组。
- pos 和容器的资源隔离。
- 与 CRI-O 集成以共享容器和后端代码。

支持 Fedora、RHEL 与 Ubuntu 等的不同版本。

1. 允许 Podman CLI 使用 Varlink 后端连接到远程 Podman 实例。
2. 将 Libpod 集成到 CRI-O 中以替换其现有的容器管理后端。
3. 进一步改进 Podman pod 命令
4. 不需要 root（rootless）容器的进一步改进

### 2.1 OCI

因为它们（包括Docker）都遵循OCI （Open Container Initiative）下的相同规范。它们包含了容器运行时、容器分发和容器镜像的规范，其中涵盖了使用容器所需的所有特性。

有了OCI，你可以选择一套最符合你需求的工具，同时你仍然可以享受跟Docker一样使用相同的API和CLI命令。

### 2.2 容器引擎



目前已经有许多容器引擎，但Docker最突出的竞争对手是由红帽开发的Podman。与Docker不同，Podman不需要Daemon来运行，也不需要root特权，这是Docker长期以来一直关注的问题。基于它的名字，Podman不仅可以运行容器，还可以运行pods。

* LXD——LXC （Linux Containers）是一个容器管理器（守护进程）。该工具提供了运行系统容器的能力，这些系统容器提供了更类似于VM的容器环境。它位于非常狭窄的空间，没什么用户，所以除非你有非常具体的实例，否则最好还是使用Docker或Podman。
* CRI-O——当你Google什么是CRI-O你可能会发现它被描述为容器引擎。不过，实际上它只是容器运行时。其实它既不是引擎，也不适合“正常”使用。我的意思是，它是专门为Kubernetes运行时（CRI）而构建的，而不是为最终用户使用的。
* Rkt——rkt（“火箭”）是由CoreOS开发的容器引擎。这里提到这个项目只是为了完整性，因为这个项目已经结束，开发也停止了——所以也就没必要再使用了。





Podman 原来是 CRI-O 项目的一部分，后来被分离成一个单独的项目叫 libpod。Podman 的使用体验和 Docker 类似，不同的是 Podman 没有 daemon。以前使用 Docker CLI 的时候，Docker CLI 会通过 gRPC API 去跟 Docker Engine 说「我要启动一个容器」，然后 Docker Engine 才会通过 OCI Container runtime（默认是 runc）来启动一个容器。这就意味着容器的进程不可能是 Docker CLI 的子进程，而是 Docker Engine 的子进程。

Podman 比较简单粗暴，它不使用 Daemon，而是直接通过 OCI runtime（默认也是 runc）来启动容器，所以容器的进程是 podman 的子进程。这比较像 Linux 的 fork/exec 模型，而 Docker 采用的是 C/S（客户端/服务器）模型。与 C/S 模型相比，fork/exec 模型有很多优势，比如：

系统管理员可以知道某个容器进程到底是谁启动的。

如果利用 cgroup 对 podman 做一些限制，那么所有创建的容器都会被限制。

SD_NOTIFY : 如果将 podman 命令放入 systemd 单元文件中，容器进程可以通过 podman 返回通知，表明服务已准备好接收任务。

socket 激活 : 可以将连接的 socket 从 systemd 传递到 podman，并传递到容器进程以便使用它们。

废话不多说，下面我们直接进入实战环节，本文将手把手教你如何用 podman 来部署静态博客，并通过 Sidecar 模式将博客所在的容器加入到 Envoy mesh 之中。



## 三 实操

### 3.1 安装

#### 3.1.1 Linux安装

Centos 8 默认使用podman

```shell
$ sudo yum -y install podman
```

* 配置加速

改名并备份好文件：`/etc/containers/registries.conf`
再新建一个空的 registries.conf 文件，插入如下内容

```shell
unqualified-search-registries = ["docker.io"]
[[registry]]
prefix = "docker.io"
location = "5980zxy5.mirror.aliyuncs.com"
```



#### 3.1.2 Mac安装

```shell
$ brew install podman
```

启用podman服务

```shell
$ podman images ls
Cannot connect to Podman. Please verify your connection to the Linux system using `podman system connection list`, or try `podman machine init` and `podman machine start` to manage a new Linux VM
Error: unable to connect to Podman socket: Get "http://d/v3.4.0/libpod/_ping": dial unix ///var/folders/wn/367g1v9n1bv0sg1k8qldzym80000gn/T/podman-run--1/podman/podman.sock: connect: no such file or directory

# 需要启动machine 
$ podman machine init
Downloading VM image: fedora-coreos-34.20211004.2.0-qemu.x86_64.qcow2.xz: done
Extracting compressed file
$ podman machine start
INFO[0000] waiting for clients...
INFO[0000] listening tcp://0.0.0.0:7777
INFO[0000] new connection from  to /var/folders/wn/367g1v9n1bv0sg1k8qldzym80000gn/T/podman/qemu_podman-machine-default.sock
Waiting for VM ...
2Machine "podman-machine-default" started successfully
```

注意：再次使用的微vm使用的镜像，不是宿主机的镜像。

### 3.2 基础命令

```shell
Manage pods, containers and images

Usage:
  podman [options] [command]

Available Commands:
  attach      Attach to a running container
  build       Build an image using instructions from Containerfiles
  commit      Create new image based on the changed container
  container   Manage containers
  create      Create but do not start a container
  diff        Display the changes to the object's file system
  events      Show podman events
  exec        Run a process in a running container
  export      Export container's filesystem contents as a tar archive
  generate    Generate structured data based on containers and pods.
  healthcheck Manage health checks on containers
  help        Help about any command
  history     Show history of a specified image
  image       Manage images
  images      List images in local storage
  import      Import a tarball to create a filesystem image
  info        Display podman system information
  init        Initialize one or more containers
  inspect     Display the configuration of object denoted by ID
  kill        Kill one or more running containers with a specific signal
  load        Load image(s) from a tar archive
  login       Login to a container registry
  logout      Logout of a container registry
  logs        Fetch the logs of one or more containers
  manifest    Manipulate manifest lists and image indexes
  network     Manage networks
  pause       Pause all the processes in one or more containers
  play        Play a pod and its containers from a structured file.
  pod         Manage pods
  port        List port mappings or a specific mapping for the container
  ps          List containers
  pull        Pull an image from a registry
  push        Push an image to a specified destination
  restart     Restart one or more containers
  rm          Remove one or more containers
  rmi         Removes one or more images from local storage
  run         Run a command in a new container
  save        Save image(s) to an archive
  search      Search registry for image
  start       Start one or more containers
  stats       Display a live stream of container resource usage statistics
  stop        Stop one or more containers
  system      Manage podman
  tag         Add an additional name to a local image
  top         Display the running processes of a container
  unpause     Unpause the processes in one or more containers
  untag       Remove a name from a local image
  version     Display the Podman Version Information
  volume      Manage volumes
  wait        Block on one or more containers

Options:
  -c, --connection string   Connection to use for remote Podman service
      --help                Help for podman
      --identity string     path to SSH identity file, (CONTAINER_SSHKEY)
      --log-level string    Log messages above specified level (debug, info, warn, error, fatal, panic) (default "error")
      --url string          URL to access Podman service (CONTAINER_HOST) (default "unix:/var/folders/wn/367g1v9n1bv0sg1k8qldzym80000gn/T/podman-run--1/podman/podman.sock")
  -v, --version             version for podman
```

### 3.3 运行一个基础命令

#### 3.3.1 查看信息

```shell
$ podman --remote info
host:
  arch: amd64
  buildahVersion: 1.15.1
  cgroupVersion: v1
  conmon:
    package: conmon-2.0.20-2.module_tl3+88+de755738.x86_64
    path: /usr/bin/conmon
    version: 'conmon version 2.0.20, commit: a9ae6b9ffaadd564332ddf93b4641d48137430bf'
  cpus: 2
  distribution:
    distribution: '"tencentos"'
    version: "3.1"
  eventLogger: file
  hostname: xuel-terraform-cvm-0
  idMappings:
    gidmap: null
    uidmap: null
  kernel: 5.4.119-19-0006
  linkmode: dynamic
  memFree: 2587340800
  memTotal: 3851665408
  ociRuntime:
    name: runc
    package: runc-1.0.0-68.rc92.module_tl3+88+de755738.x86_64
    path: /usr/bin/runc
    version: 'runc version spec: 1.0.2-dev'
  os: linux
  remoteSocket:
    path: /run/podman/podman.sock
  rootless: false
  slirp4netns:
    executable: ""
    package: ""
    version: ""
  swapFree: 0
  swapTotal: 0
  uptime: 8m 7.8s
registries:
  search:
  - registry.access.redhat.com
  - registry.redhat.io
  - docker.io
store:
  configFile: /etc/containers/storage.conf
  containerStore:
    number: 0
    paused: 0
    running: 0
    stopped: 0
  graphDriverName: overlay
  graphOptions:
    overlay.mountopt: nodev,metacopy=on
  graphRoot: /var/lib/containers/storage
  graphStatus:
    Backing Filesystem: extfs
    Native Overlay Diff: "false"
    Supports d_type: "true"
    Using metacopy: "true"
  imageStore:
    number: 0
  runRoot: /var/run/containers/storage
  volumePath: /var/lib/containers/storage/volumes
version:
  APIVersion: 1
  Built: 1618972697
  BuiltTime: Wed Apr 21 10:38:17 2021
  GitCommit: ""
  GoVersion: go1.14.12
  OsArch: linux/amd64
  Version: 2.0.5
```

#### 3.3.2 运行容器

```shell
# 拉取容器
$ podman pull nginx:latest

# 运行容器
$ podman run -itd -p 80:80 nginx:latest
6ccd9783fe8b868ef273618570bfff5d40abea9c45e252e5033a2f4b3f6725f8

# 查看容器
$ podman ps -a
CONTAINER ID  IMAGE                           COMMAND               CREATED        STATUS            PORTS               NAMES
6ccd9783fe8b  docker.io/library/nginx:latest  nginx -g daemon o...  9 seconds ago  Up 8 seconds ago  0.0.0.0:80->80/tcp  affectionate_rubin

# 查看容器日志
$ podman logs 6ccd9783fe8b
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: info: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: info: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
2021/10/08 06:28:55 [notice] 1#1: using the "epoll" event method

```

podman 基本上和docker命令是一致的，如果熟悉docker，可以将podman，alias为docker来使用

```shell
$ alias docker='podman'
$ docker  ps -a
CONTAINER ID  IMAGE                           COMMAND               CREATED        STATUS            PORTS               NAMES
6ccd9783fe8b  docker.io/library/nginx:latest  nginx -g daemon o...  3 minutes ago  Up 3 minutes ago  0.0.0.0:80->80/tcp  affectionate_rubin
$ docker  images
REPOSITORY               TAG     IMAGE ID      CREATED     SIZE
docker.io/library/nginx  latest  f8f4ffc8092c  9 days ago  138 MB
```

docker 项目容器操作再次就不详细演示。

### 3.4 运行pod

podman顾名思义，其可以对接K8s，实现直接创建Pod，

```shell
$ docker pod --help
Manage pods

Description:
  Pods are a group of one or more containers sharing the same network, pid and ipc namespaces.

Usage:
  podman pod [command]

Available Commands:
  create      Create a new empty pod
  exists      Check if a pod exists in local storage
  inspect     Displays a pod configuration
  kill        Send the specified signal or SIGKILL to containers in pod
  pause       Pause one or more pods
  prune       Remove all stopped pods and their containers
  ps          List pods
  restart     Restart one or more pods
  rm          Remove one or more pods
  start       Start one or more pods
  stats       Display a live stream of resource usage statistics for the containers in one or more pods
  stop        Stop one or more pods
  top         Display the running processes of containers in a pod
  unpause     Unpause one or more pods
```

运行pod依赖与pause镜像，需要先拉取下来

```shell

# 如果不能访问k8s.gcr.io，可以从其他源拉取再修改tag
$ docker pull mirrorgooglecontainers/pause:3.1
$ docker tag docker.io/mirrorgooglecontainers/pause:3.1  k8s.gcr.io/pause:3.1
```

启动pod

```shell
# 创建pod
$ docker pod create --name mynginxpod
f26515e8caa7c5d84b15cef6702d1ba6f83be3e6760c96f0619f9f60ac5df1e0

# 查看pod
$ docker  pod ls
POD ID        NAME        STATUS   CREATED        # OF CONTAINERS  INFRA ID
0560f1744dee  mynginxpod  Created  2 seconds ago  1                375497f7b909

# 在pod中运行容器
$ docker run -d --pod mynginxpod nginx:latest
9f9696382299d0eef711c8ec68baa02b49635e7b9a87f22d1d951e9a326b41a9

$ docker  pod ls
POD ID        NAME        STATUS   CREATED         # OF CONTAINERS  INFRA ID
0560f1744dee  mynginxpod  Running  44 seconds ago  2                375497f7b909

# 查看pod中所有容器
$ docker  ps -pa
CONTAINER ID  IMAGE                                       COMMAND               CREATED            STATUS                PORTS   NAMES               POD ID        PODNAME
375497f7b909  docker.io/mirrorgooglecontainers/pause:3.1                        About an hour ago  Up About an hour ago          0560f1744dee-infra  0560f1744dee  mynginxpod
9f9696382299  docker.io/library/nginx:latest              nginx -g daemon o...  About an hour ago  Up About an hour ago          pensive_montalcini  0560f1744dee  mynginxpod

# 查看资源使用情况
$ docker pod top mynginxpod
USER    PID   PPID   %CPU    ELAPSED             TTY   TIME   COMMAND
0       1     0      0.000   1h9m55.211491679s   ?     0s     /pause
root    1     0      0.000   1h9m55.212004807s   ?     0s     nginx: master process nginx -g daemon off;
nginx   30    1      0.000   1h9m55.212053645s   ?     0s     nginx: worker process
nginx   31    1      0.000   1h9m55.212089686s   ?     0s     nginx: worker process
```

### 3.5 导出资源清单

```shell
$ docker  generate kube mynginxpod > mynginxpod.yaml
$ cat mynginxpod.yaml
# Generation of Kubernetes YAML is still under development!
#
# Save the output of this file and use kubectl create -f to import
# it into Kubernetes.
#
# Created with podman-2.0.5
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2021-10-08T08:14:23Z"
  labels:
    app: mynginxpod
  name: mynginxpod
spec:
  containers:
  - command:
    - nginx
    - -g
    - daemon off;
    env:
    - name: PATH
      value: /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
    - name: TERM
      value: xterm
    - name: PKG_RELEASE
      value: 1~buster
    - name: container
      value: podman
    - name: NGINX_VERSION
      value: 1.21.3
    - name: NJS_VERSION
      value: 0.6.2
    - name: HOSTNAME
      value: mynginxpod
    image: docker.io/library/nginx:latest
    name: pensivemontalcini
    resources: {}
    securityContext:
      allowPrivilegeEscalation: true
      capabilities: {}
      privileged: false
      readOnlyRootFilesystem: false
      seLinuxOptions: {}
    workingDir: /
status: {}
---
metadata:
  creationTimestamp: null
spec: {}
status:
  loadBalancer: {}
```

该文件为兼容k8s的pod资源清单文件，可以通过该文件直接创建pod

```shell
$ docker  pod ls
POD ID        NAME        STATUS   CREATED            # OF CONTAINERS  INFRA ID
0560f1744dee  mynginxpod  Running  About an hour ago  2                375497f7b909

# 删除pod
$ docker  pod rm -f mynginxpod
0560f1744dee0492627ea28e5f19664dcee33e51a1ebf2243a6be9b5ab2f6331
$ docker  pod ls
POD ID  NAME    STATUS  CREATED  # OF CONTAINERS  INFRA ID
$ docker play kube mynginxpod.yaml
Trying to pull docker.io/library/nginx:latest...
Getting image source signatures
Copying blob 4ce73aa6e9b0 skipped: already exists
Copying blob 44ac32b0bba8 skipped: already exists
Copying blob bbe0b7acc89c skipped: already exists
Copying blob 07aded7c29c6 skipped: already exists
Copying blob 91d6e3e593db skipped: already exists
Copying blob 8700267f2376 [--------------------------------------] 0.0b / 0.0b
Copying config f8f4ffc809 done
Writing manifest to image destination
Storing signatures
Pod:
3ca51af554315e1c828f81ca29a5dbaf3a256a5037170b4378ef0bc8ad1824c5
Container:
3d5c5a674f6140d86138e288db73eee1594de7598da2dd982f29299cbe97ecef

$ docker  pod ls
POD ID        NAME        STATUS   CREATED         # OF CONTAINERS  INFRA ID
3ca51af55431  mynginxpod  Running  44 seconds ago  2                ba4e1570dc74
```

podman 由两部分组成，一个是 podman CLI，还有一个是 container runtime，container runtime 由 conmon 来负责，主要包括监控、日志、TTY 分配以及类似 out-of-memory 情况的杂事。也就是说，conmon 是所有容器的父进程。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211008162301.png)

### 3.6 运行两个容器

```shell
$ docker  pod ls
POD ID        NAME        STATUS   CREATED         # OF CONTAINERS  INFRA ID
3ca51af55431  mynginxpod  Running  17 minutes ago  2                ba4e1570dc74

# 同一个pod中启动三个容器
$ docker run -d --pod mynginxpod tomcat:latest
08b77aeb728668d2ecea9cffe86d03e1177c903fcfcee8c91fee5816e3172d22
$ docker pod ls
POD ID        NAME        STATUS   CREATED         # OF CONTAINERS  INFRA ID
3ca51af55431  mynginxpod  Running  17 minutes ago  3                ba4e1570dc74
```

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211008163709.png)

## 四 相关概念

### 4.1 容器引擎

Container Engine是一种工具，它为处理镜像和容器提供用户界面，远程仓库提取镜像并将其扩展到磁盘。它看起来也是运行容器，但实际上它的工作是创建容器清单和带有镜像层的目录。然后它将它们传递到容器运行时，如runC或Crun

* Docker：
* Podman：
* LXD——LXC （Linux Containers）是一个容器管理器（守护进程）。该工具提供了运行系统容器的能力，这些系统容器提供了更类似于VM的容器环境。它位于非常狭窄的空间，没什么用户，所以除非你有非常具体的实例，否则最好还是使用Docker或Podman。
* CRI-O——当你Google什么是CRI-O你可能会发现它被描述为容器引擎。不过，实际上它只是容器运行时。其实它既不是引擎，也不适合“正常”使用。我的意思是，它是专门为Kubernetes运行时（CRI）而构建的，而不是为最终用户使用的。
* Rkt——rkt（“火箭”）是由CoreOS开发的容器引擎。这里提到这个项目只是为了完整性，因为这个项目已经结束，开发也停止了——所以也就没必要再使用了。

### 4.2 镜像构建

* Docker
* Buildah红帽开发，目前已经集成在podman中，无守护程序和无根的，并遵循OCI的镜像标准，所以它能保证所构建的镜像和Docker构建的是一样的
* Kaniko谷歌开发，Kaniko也是从Dockerfile构建容器镜像，跟Buildah类似，也不需要守护进程。与Buildah的主要区别在于，Kaniko更专注于在Kubernetes中构建镜像。Kaniko使用gcr.io/ Kaniko -project/executor作为镜像运行。这对于Kubernetes来说是行得通的，但是对于本地构建来说不是很方便，并且在某种程度上违背了它的初衷，因为我们得先使用Docker来运行Kaniko镜像，然后再去构建镜像。也就是说，如果正在为Kubernetes集群中构建镜像的工具进行选型（例如在CI/CD Pipeline中），那么Kaniko可能是一个不错的选择，因为它是无守护程序的，而且（可能）更安全。
  

### 4.3 容器运行时

* runC：是基于OCI容器运行时规范创建的，且最流行的容器运行时。Docker（通过containerd）、Podman和crio使用它，所以几乎所有东西都依赖于LXD。它几乎是所有产品/工具的默认首选项，所以即使你在阅读本文后放弃Docker，但你仍然会用到runC。

* CRI-O：因为它被构建为用于Kubernetes节点上的运行时，可以看到它被描述为“Kubernetes需要的所有运行时，仅此而已”。因此，除非你正在设置Kubernetes集群（或OpenShift集群——CRI-O已经是默认首选项了），否则不大可能会接触到这个。
* containerd：它是一个守护进程，充当各种容器运行时和操作系统的API。在后台，它依赖于runC，是Docker引擎的默认运行时。谷歌Kubernetes引擎（GKE）和IBM Kubernetes服务（IKS）也在使用。它是Kubernetes容器运行时接口的一个部署（与CRI-O相同），因此它是Kubernetes集群运行时的一个很好的备选项。
* Kata:kata containers是由OpenStack基金会管理，但独立于OpenStack项目之外的容器项目。kata containers整合了Intel的 Clear Containers 和 Hyper.sh 的 runV，能够支持不同平台的硬件 （x86-64，arm等），并符合OCI(Open Container Initiative)规范，同时还可以兼容k8s的 CRI（Container Runtime Interface）接口规范。项目包含几个配套组件，即Runtime，Agent， Proxy，Shim等。项目已于6月份release了1.0版本。Kata最大的亮点是解决了传统容器共享内核的安全和隔离问题

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211008182356.png)

## 参考链接

* https://podman.io/
* https://blog.csdn.net/alex_yangchuansheng/article/details/102618128
* https://github.com/containers/podman/blob/main/docs/tutorials/mac_win_client.md