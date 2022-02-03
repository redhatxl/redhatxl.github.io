# 最小化K8s环境部署之Minikube

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210806225917.png)

## 一 背景

最近需要给开发相关同时培训K8s，集群方式部署负责且占用资源多，简单快捷高效的单机版K8s环境，可谓开发人员不错的选择，minikube就是为解决这个问题而衍生出来的工具，它基于go语言开发， 是一个易于在本地运行 Kubernetes 的工具，可在你的笔记本电脑上的虚拟机内轻松创建单机版 Kubernetes 集群。便于尝试 Kubernetes 或使用 Kubernetes 日常开发。可以在单机环境下快速搭建可用的k8s集群，非常适合测试和本地开发。如果没有服务器或在本地笔记本安装，则可以在线使用https://labs.play-with-k8s.com/来体验K8s。

## 二 Minikube简介

### 2.1 主要组件

#### **localkube**

为了运行和管理 Kubernetes 的组件，Minikube 中使用了 Spread's 的 localkube，localkube 是一个独立的 Go 语言的二进制包，包含了所有 Kubernetes 的主要组件，并且以不同的 goroutine 来运行。

#### **libmachine**

为了支持 MacOS 和 Windows，Minikube 在内部使用 libmachine 创建或销毁虚拟机，可以将它理解为一个虚拟机的驱动程序。至于在 Linux 上，由于集群可以直接本地运行，所以避免设置虚拟机。

### 2.2 启动流程

- 通过 libmachine 启动虚拟机，生成 Docker 相关证书及配置文件，启动 Docker;
- 生成 Kubernetes 相关配置文件和 addons，以及相关证书，拷贝至虚拟机;
- 基于之前的配置文件，生成启动脚本，启动 Kubernetes 集群，并可以通过 Client 进行访问。

#### MacOS/Windows

- minikube -> libmachine -> virtualbox/hyper V -> linux VM -> localkube

#### Linux

- minikube -> docker -> localkube

## 三 安装部署

### 3.1 Mac安装

#### 3.1.1 预先条件

- virtual box安装

可以从此链接下载：https://www.virtualbox.org/wiki/Downloads

- kubectl 安装

```shell
brew install kubectl
```

#### 3.1.2 安装

官方出品的minikube，默认连接的是google官方站点，由于众所周知的原因可以利用阿里云修改过的minikube，目前已经替换了其中国外的镜像源

```shell
curl -Lo minikube http://kubernetes.oss-cn-hangzhou.aliyuncs.com/minikube/releases/v0.30.0/minikube-darwin-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/
```

#### 3.1.3 启动

```shell
minikube start --vm-driver=virtualbox --registry-mirror=https://registry.docker-cn.com
```

注：如果首次失败了(比如：步骤一中的安全设置没勾选，导致无法启用），可以先尝试minikute delete 删除原来的machine。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210806155459.png)



### 3.2 Linux服务器安装

* 基础环境

```shell
CentOS 7.8
```

#### 3.2.1 docker安装

``` yum install -y yum-utils device-mapper-persistent-data lvm2 wget
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum clean all && yum makecache fast
yum -y install docker-ce
systemctl start docker
```

#### 3.2.2 kubelet安装

```shell
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.18.0/bin/linux/amd64/kubectl &&\
chmod +x ./kubectl &&\
mv ./kubectl /usr/bin/kubectl
```

#### 3.2.3 minikube安装

```shell
$ curl -Lo minikube https://kubernetes.oss-cn-hangzhou.aliyuncs.com/minikube/releases/v1.18.1/minikube-linux-amd64 && $ chmod +x minikube && sudo mv minikube /usr/local/bin/

# 使用docker驱动不能使用root用户，新建minikube用户用于启动
$ useradd minikube
$ usermod -a -G docker minikube
$ su minikube
# 切换到minikube用户进行安装
$ minikube start --driver=docker
😄  Centos 7.9.2009 (amd64) 上的 minikube v1.18.1
🎉  minikube 1.20.0 is available! Download it: https://github.com/kubernetes/minikube/releases/tag/v1.20.0
💡  To disable this notice, run: 'minikube config set WantUpdateNotification false'

✨  根据用户配置使用 docker 驱动程序
👍  Starting control plane node minikube in cluster minikube
🚜  Pulling base image ...
🔥  Creating docker container (CPUs=2, Memory=2200MB) ...
    > kubelet.sha256: 64 B / 64 B [--------------------------] 100.00% ? p/s 0s
    > kubectl.sha256: 64 B / 64 B [--------------------------] 100.00% ? p/s 0s
    > kubeadm.sha256: 64 B / 64 B [--------------------------] 100.00% ? p/s 0s
    > kubectl: 38.37 MiB / 38.37 MiB [---------------] 100.00% 2.68 MiB p/s 14s
    > kubeadm: 37.40 MiB / 37.40 MiB [---------------] 100.00% 2.18 MiB p/s 17s
    > kubelet: 108.73 MiB / 108.73 MiB [-------------] 100.00% 4.11 MiB p/s 26s

    ▪ Generating certificates and keys ...
    ▪ Booting up control plane ...
    ▪ Configuring RBAC rules ...
🔎  Verifying Kubernetes components...
    ▪ Using image registry.cn-hangzhou.aliyuncs.com/google_containers/storage-provisioner:v4 (global image repository)
🌟  Enabled addons: storage-provisioner, default-storageclass
💡  kubectl not found. If you need it, try: 'minikube kubectl -- get pods -A'
🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default

### 查看信息
$ kubectl cluster-info
### 进入到minikube
$ minikube ssh
$ docker ps
```



## 四 使用

使用下列命令可以打开控制面板，自动跳转浏览器查看

```
minikube dashboard
```

使用下面命令可以查看虚拟机ip

```
minikube ip
```

查看状态使用下面命令

```
minikube status
```

其他命令可以使用minikube —help查看帮助，如下

```
➜  bin minikube --help
Minikube is a CLI tool that provisions and manages single-node Kubernetes clusters optimized for development workflows.

Usage:
  minikube [command]

Available Commands:
  addons         Modify minikube's kubernetes addons
  cache          Add or delete an image from the local cache.
  completion     Outputs minikube shell completion for the given shell (bash or zsh)
  config         Modify minikube config
  dashboard      Access the kubernetes dashboard running within the minikube cluster
  delete         Deletes a local kubernetes cluster
  docker-env     Sets up docker env variables; similar to '$(docker-machine env)'
  help           Help about any command
  ip             Retrieves the IP address of the running cluster
  logs           Gets the logs of the running instance, used for debugging minikube, not user code
  mount          Mounts the specified directory into minikube
  profile        Profile sets the current minikube profile
  service        Gets the kubernetes URL(s) for the specified service in your local cluster
  ssh            Log into or run a command on a machine with SSH; similar to 'docker-machine ssh'
  ssh-key        Retrieve the ssh identity key path of the specified cluster
  start          Starts a local kubernetes cluster
  status         Gets the status of a local kubernetes cluster
  stop           Stops a running local kubernetes cluster
  update-check   Print current and latest version number
  update-context Verify the IP address of the running cluster in kubeconfig.
  version        Print the version of minikube

Flags:
      --alsologtostderr                  log to standard error as well as files
  -b, --bootstrapper string              The name of the cluster bootstrapper that will set up the kubernetes cluster. (default "kubeadm")
  -h, --help                             help for minikube
      --log_backtrace_at traceLocation   when logging hits line file:N, emit a stack trace (default :0)
      --log_dir string                   If non-empty, write log files in this directory
      --logtostderr                      log to standard error instead of files
  -p, --profile string                   The name of the minikube VM being used.
                                            This can be modified to allow for multiple minikube instances to be run independently (default "minikube")
      --stderrthreshold severity         logs at or above this threshold go to stderr (default 2)
  -v, --v Level                          log level for V logs
      --vmodule moduleSpec               comma-separated list of pattern=N settings for file-filtered logging

Use "minikube [command] --help" for more information about a command.
```

随后使用k8s的命令即可在本机进行测试，如

```
kubectl get pods
```

## 参考链接

* https://github.com/kubernetes/minikube
* https://minikube.sigs.k8s.io/docs/start/
* https://zhuanlan.zhihu.com/p/38268410
* https://kubernetes.io/blog/2016/07/minikube-easily-run-kubernetes-locally/