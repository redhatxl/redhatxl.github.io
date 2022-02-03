# 最小化K8s环境部署之K3s

# 一 背景

K3s 是 rancher 公司开发维护的一套 K8s 发行版。所谓发行版，就类似于 debian 之于 Linux。内核机制还是和 K8s 一样，但是剔除了很多外部依赖以及 K8s 的 alpha、beta 特性，同时改变了部署方式和运行方式，目的是轻量化 K8s，并将其应用于 IoT 设备（比如树莓派）。 简单来说，K3s 就是阉割版 K8s，消耗资源极少。 K3s **官网文档**中有详细介绍，本文下面大部分也是基于此文档进行部署等操作。



# 二 K3s简介

## 2.1 简介

#### k3s是经CNCF一致性认证的Kubernetes发行版，专为物联网及边缘计算设计。



## 2.2 特点

* 完美适配边缘环境：k3s是一个高可用的、经过CNCF认证的Kubernetes发行版，专为无人值守、资源受限、偏远地区或物联网设备内部的生产工作负载而设计。
* 简单且安全：k3s被打包成单个小于60MB的二进制文件，从而减少了运行安装、运行和自动更新生产Kubernetes集群所需的依赖性和步骤。

* 针对ARM进行优化：ARM64和ARMv7都支持二进制文件和多源镜像。k3s在小到树莓派或大到 AWS a1.4xlarge 32GiB服务器的环境中均能出色工作。

## 2.3 工作原理

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210806162239.png)

这可以看出来，K8s 所有控制面组件最终都以进程的形式运行在 server node 上，不再以静态pod的形式。数据库使用 SQLite ，没有etcd 那么重了。也就是说，当我们安装部署好 K3s 后，使用`kubectl get po -n kube-system` 时，则不会有 apiserver、scheduler 等控制面的pod了。

Agent 端 kubelet 和 kube proxy 都是进程化了，此外容器运行时也由docker 改为 containerd。

server node 和 agent node 通过特殊的代理通道连接。

从这个运行机制确实能感受到 K3s 极大的轻量化了 K8s。

## 2.4 **架构 & 运行机制**

虽说 K3s 其内核就是个 K8s，但是他的整体架构和运行机制都被 rancher 魔改了。K3s 相较于 K8s 其较大的不同点如下：

- 存储etcd 使用 嵌入的 sqlite 替代，但是可以外接 etcd 存储
- apiserver 、schedule 等组件全部简化，并以进程的形式运行在节点上
- 网络插件使用 Flannel， 反向代理入口使用 traefik 代替 ingress nginx
- 默认使用 local-path-provisioner 提供本地存储卷

# 三 安装部署

## 3.1 脚本安装

```shell




```



## 3.2 二进制部署



```shelll
wget -c https://github.com/k3s-io/k3s/releases/download/v1.21.3%2Bk3s1/k3s
# 启动server
./k3s server &

# 启动agent

# On a different node run the below. NODE_TOKEN comes from
# /var/lib/rancher/k3s/server/node-token on your server
sudo k3s agent --server https://myserver:6443 --token ${NODE_TOKEN}
```



# 四 测试

```shell
$./k3s kubectl get nodes
NAME            STATUS   ROLES                  AGE     VERSION
vm-0-3-centos   Ready    control-plane,master   2m22s   v1.21.3+k3s1
```

* 使用kubectl管理

```shell
# 配置文件在Kubeconfig is written to /etc/rancher/k3s/k3s.yaml
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.18.0/bin/linux/amd64/kubectl &&\
chmod +x ./kubectl && mv ./kubectl /usr/bin/kubectl

# 配置文件
$ cp /etc/rancher/k3s/k3s.yaml .kube/config
$ kubectl get nodes
NAME            STATUS   ROLES                  AGE     VERSION
vm-0-3-centos   Ready    control-plane,master   5m37s   v1.21.3+k3s1
```







# 五 其他

# 







# 参考链接

* https://www.rancher.cn/k3s/



* https://github.com/k3s-io/k3s