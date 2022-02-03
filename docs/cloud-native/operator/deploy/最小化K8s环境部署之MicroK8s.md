# 最小化K8s环境部署之MicroK8s

## 一 背景

MicroK8s是目前最小、最快与Kubernetes全面兼容的集群系统，主要用于工作站和小型团队，但是目前镜像并没有与snap打包在一起，还在gcr.io上，国内下载上还是有问题。MicroK8s适合离线开发、原型开发和测试，尤其是运行VM作为小、便宜、可靠的k8s用于CI/CD。支持arm架构，也适合开发 IoT 应用，通过 MicroK8s 部署应用到小型Linux设备上。

## 二 MicroK8s特点

* **MicroK8轻巧** ：团队成员希望最小的Kubernetes用于笔记本电脑和工作站的开发。 MicroK8s提供了轻量级的独立Kubernetes，在Ubuntu上运行时，它与Azure AKS，Amazon EKS和Google GKE兼容。
* **MicroK8很简单** ：MicroK8s通过单软件包安装来最大程度地减少管理和操作，该软件包没有活动部件（开箱即用），并且包括所有依赖项。
* **MicroK8是安全的** ：对于所有安全问题，更新始终可用，并且可以立即应用或安排更新以适合企业的维护周期。 此外，MicroK8具有最新的隔离功能，可在工作站上安全运行。 通过将Kubernetes，Docker.io，iptables和CNI的所有二进制文件打包在单个snap软件包中，可以实现这种隔离。
* **MicroK8是最新的** ：MicroK8s跟踪上游Kubernetes，并在上游Kubernetes发行的同一天发布beta，发行候选版本和最终版本。 您可以跟踪最新的Kubernetes或坚持使用从1.10开始的任何Kubernetes版本。 当出现新的主要Kubernetes版本时，您可以自动升级或使用单个命令进行升级。
* **MicroK8是全面的** ：MicroK8s包括精选的清单，用于常见的Kubernetes功能和服务。 MicroK8带有Docker注册表，使用户可以在笔记本电脑上制作，推送和部署容器。

## 三 安装部署

### 3.1 运行环境

- 操作系统 Ubuntu 18.04 LTS 或16.04 LTS 环境 (或其他支持 `snapd` 的操作系统- see the [snapd documentation](https://www.oschina.net/action/GoToLink?url=https%3A%2F%2Fsnapcraft.io%2Fdocs%2Finstalling-snapd))。
- 至少 20G 磁盘空间， （建议）4G 内存。

```shell
$ snap install microk8s --classic
2021-08-06T16:56:05+08:00 INFO Waiting for automatic snapd restart...
microk8s (1.21/stable) v1.21.3 from Canonical✓ installed
```

## 四 使用

```shell
$ microk8s kubectl get nodes
NAME             STATUS     ROLES    AGE    VERSION
vm-0-17-ubuntu   NotReady   <none>   2m7s   v1.21.3-3+90fd5f3d2aea0a
$ microk8s kubectl get services
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.152.183.1   <none>        443/TCP   2m27s

# 检测服务状态
$ microk8s status --wait-ready

# 启用相关组建
$ microk8s enable dashboard dns registry istio

# 查看k8s
$ microk8s kubectl get all --all-namespaces

# 访问dashboard
$ microk8s dashboard-proxy

# 使用以有kubectl管理
$ sudo microk8s kubectl config view --raw > $HOME/.kube/config

# 查看插件
$ microk8s.status
```

使用宿主机kubectl管理集群

```shell
$ curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.18.0/bin/linux/amd64/kubectl &&\
chmod +x ./kubectl &&\
$ mv ./kubectl /usr/bin/kubectl

$ microk8s kubectl config view --raw > $HOME/.kube/config
$ kubectl get po -A
NAMESPACE     NAME                                      READY   STATUS     RESTARTS   AGE
kube-system   calico-kube-controllers-f7868dd95-s4c5m   0/1     Pending    0          5m33s
kube-system   calico-node-8mxlc                         0/1     Init:0/3   0          5m27s	
```

## 五 其他

与Minikube不同，IT管理员或开发人员可以使用MicroK8s创建多节点集群。如果MicroK8s在Linux上运行，甚至不需要VM。在Windows和macOS上，MicroK8s使用名为Multipass的VM框架为Kubernetes集群创建VM。


# 参考链接

* https://github.com/ubuntu/microk8s

* https://microk8s.io/

  