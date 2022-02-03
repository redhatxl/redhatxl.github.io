# 最小化K8s环境部署之Kind

## 一 背景

[kind](https://link.zhihu.com/?target=https%3A//github.com/kubernetes-sigs/kind) 即 Kubernetes In Docker，顾名思义，就是将 k8s 所需要的所有组件，全部部署在一个docker容器中，是一套开箱即用的 k8s 环境搭建方案。使用 kind 搭建的集群无法在生产中使用，但是如果你只是想在本地简单的玩玩 k8s，不想占用太多的资源，那么使用 kind 是你不错的选择。同样，kind 还可以很方便的帮你本地的 k8s 源代码打成对应的镜像，方便测试。

## 二 Kind简介

kind（Kubernetes IN Docker） 是一个基于 docker 构建 Kubernetes 集群的工具，非常适合用来在本地搭建基于 Kubernetes 的开发/测试环境。

### 2.1 kind启动流程

1. 查看本地上是否存在一个基础的安装镜像，默认是 kindest/node:v1.13.4，这个镜像里面包含了需要安装的所有东西，包括了 kubectl、kubeadm、kubelet 二进制文件，以及安装对应版本 k8s 所需要的镜像，都以 tar 压缩包的形式放在镜像内的一个路径下
2. 准备你的 node，这里就是做一些启动容器、解压镜像之类的工作
3. 生成对应的 kubeadm 的配置，之后通过 kubeadm 安装，安装之后还会做另外的一些操作，比如像我刚才仅安装单节点的集群，会帮你删掉 master 节点上的污点，否则对于没有容忍的 pod 无法部署。
4. 启动完毕

### 2.2 Kind vs Minikube

* Kind 不是打包一个虚拟化镜像，而是直接讲 K8S 组件运行在 Docker。

1. 不需要运行 GuestOS 占用资源更低。
2. 不基于虚拟化技术，可以在 VM 中使用。
3. 文件更小，更利于移植。

* **支持多节点 K8S 集群和 HA**

Kind 支持多角色的节点部署，你可以通过配置文件控制你需要几个 Master 节点，几个 Worker 节点，以更好的模拟生产中的实际环境。

## 三 安装部署

### 3.1 MacOS安装

```shell
brew install kind
```



### 3.2 Linux系统安装

Kind 的安装不包括 kubectl和docker，可以实现在服务器安装Kubectl和Docker

#### 3.2.1 docker安装

```yum install -y yum-utils device-mapper-persistent-data lvm2 wget
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

#### 3.2.3 Kind安装

```shell
wget https://github.com/kubernetes-sigs/kind/releases/download/0.2.1/kind-linux-amd64
mv kind-linux-amd64 kind
chmod +x kind
mv kind /usr/local/bin
```

## 四 使用

```shell
# 创建集群
$ kind create cluster

$ kubectl get po -A
NAMESPACE     NAME                                         READY   STATUS    RESTARTS   AGE
kube-system   coredns-86c58d9df4-vwm4m                     1/1     Running   0          6m37s
kube-system   coredns-86c58d9df4-xxk8s                     1/1     Running   0          6m37s
kube-system   etcd-kind-control-plane                      1/1     Running   0          5m35s
kube-system   kube-apiserver-kind-control-plane            1/1     Running   0          5m49s
kube-system   kube-controller-manager-kind-control-plane   1/1     Running   0          5m23s
kube-system   kube-proxy-d8zn6                             1/1     Running   0          6m37s
kube-system   kube-scheduler-kind-control-plane            1/1     Running   0          5m40s
kube-system   weave-net-llqf9                              2/2     Running   1          6m37s
$ kubectl get nodes
NAME                 STATUS   ROLES    AGE     VERSION
kind-control-plane   Ready    master   6m57s   v1.13.4
$ docker ps -a
CONTAINER ID   IMAGE                  COMMAND                  CREATED         STATUS         PORTS                                  NAMES
9a1318b80b5a   kindest/node:v1.13.4   "/usr/local/bin/entr…"   7 minutes ago   Up 7 minutes   33895/tcp, 127.0.0.1:33895->6443/tcp   kind-control-plane
$ docker images
REPOSITORY     TAG       IMAGE ID       CREATED       SIZE
kindest/node   v1.13.4   eaecf3d2c4de   2 years ago   1.58GB
$ kubectl cluster-info
Kubernetes master is running at https://localhost:33895
KubeDNS is running at https://localhost:33895/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

## 五 高阶使用

我们要使用本地文件存储hostpath

* 由于使用mac需要share 目录，记得添加完成后应用重启

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211219084148.png)

```shell
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
- role: worker
  # add a mount from /path/to/my/files on the host to /files on the node
  extraMounts:
  - hostPath: /path/to/my/files/
    containerPath: /files
    # optional: if set, the mount is read-only.
    # default false
    readOnly: true
    # optional: if set, the mount needs SELinux relabeling.
    # default false
    selinuxRelabel: false
    # optional: set propagation mode (None, HostToContainer or Bidirectional)
    # see https://kubernetes.io/docs/concepts/storage/volumes/#mount-propagation
    # default None
    propagation: HostToContainer
		image: kindest/node:v1.21.1
		
		
kind create cluster --config=kind-cluster.yaml

```

# 参考链接

* https://www.psvmc.cn/article/2021-02-27-kubernetes-start-3-kind.html
* https://kind.sigs.k8s.io/docs/user/quick-start
* https://github.com/kubernetes-sigs/kind
* https://docs.docker.com/desktop/mac/