# KubeEdge初探

## 前言

物联网已经产生了数量惊人的数据，随着5G网络的部署，这些数据将呈指数级增长。管理和使用这些数据是一个挑战。

无论是从交通摄像头、气象传感器、电表等会产生信息，这些信息与智能城市环境中，其他摄像头和传感器的数据相结合，在一个中心位置处理起来可能会太多，尤其是当你在预期设备会对事件做出反应时。

超大规模云计算环境中已被普遍使用的Kubernetes（简称K8s），带入到物联网边缘计算场景中。**新成立的Kubernetes物联网边缘工作组将采用运行容器的理念并扩展到边缘，促进K8s在边缘环境中的适用。

- 支持将工业物联网IIoT的连接设备数量扩展到百万量级，既可支持IP设备以直连方式接入K8s云平台，又可支持非IP设备通过物联网网关接入。

- 利用边缘节点，让计算更贴近设备侧，以便减少延迟、降低带宽需求和提高可靠性，满足用户实时、智能、数据聚合和安全需求：

  * 将流数据应用部署到边缘节点，降低设备和云平台之间通信的带宽需求。

  * 部署无服务器应用框架，使得边缘侧无需与云端通讯，便可对某些紧急情况做出快速响应。

- 在混合云和边缘环境中提供通用控制平台，以简化管理和操作。

## 一 背景



![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210923124011.png)

### 1.1 KubeEdge简介

KubeEdge 是一个开源的系统，可将本机容器化应用编排和管理扩展到边缘端设备。 它基于Kubernetes构建，为网络和应用程序提供核心基础架构支持，并在云端和边缘端部署应用，同步元数据。KubeEdge 还支持 **MQTT** 协议，允许开发人员编写客户逻辑，并在边缘端启用设备通信的资源约束。KubeEdge 包含云端和边缘端两部分。

### 1.2 KubeEdge特点

#### 边缘计算

通过在边缘端运行业务逻辑，可以在本地保护和处理大量数据。KubeEdge 减少了边和云之间的带宽请求，加快响应速度，并保护客户数据隐私。

#### 简化开发

开发人员可以编写常规的基于 http 或 mqtt 的应用程序，容器化并在边缘或云端任何地方运行。

#### Kubernetes 原生支持

使用 KubeEdge 用户可以在边缘节点上编排应用、管理设备并监控应用程序/设备状态，就如同在云端操作 Kubernetes 集群一样。

#### 丰富的应用程序

用户可以轻松地将复杂的机器学习、图像识别、事件处理等高层应用程序部署到边缘端。

## 二 KubeEdge简介

### 2.1 KubeEdge架构

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210923124141.png)

### 2.2 架构详解

#### 2.2.1 云上部分

- [CloudHub](https://kubeedge.io/en/docs/architecture/cloud/cloudhub): CloudHub 是一个 Web Socket 服务端，负责监听云端的变化, 缓存并发送消息到 EdgeHub。
- [EdgeController](https://kubeedge.io/en/docs/architecture/cloud/edge_controller): EdgeController 是一个扩展的 Kubernetes 控制器，管理边缘节点和 Pods 的元数据确保数据能够传递到指定的边缘节点。
- [DeviceController](https://kubeedge.io/en/docs/architecture/cloud/device_controller): DeviceController 是一个扩展的 Kubernetes 控制器，管理边缘设备，确保设备信息、设备状态的云边同步。

#### 2.2.2 边缘部分

- [EdgeHub](https://kubeedge.io/en/docs/architecture/edge/edgehub): EdgeHub 是一个 Web Socket 客户端，负责与边缘计算的云服务（例如 KubeEdge 架构图中的 Edge Controller）交互，包括同步云端资源更新、报告边缘主机和设备状态变化到云端等功能。
- [Edged](https://kubeedge.io/en/docs/architecture/edge/edged): Edged 是运行在边缘节点的代理，用于管理容器化的应用程序。
- [EventBus](https://kubeedge.io/en/docs/architecture/edge/eventbus): EventBus 是一个与 MQTT 服务器（mosquitto）交互的 MQTT 客户端，为其他组件提供订阅和发布功能。
- [ServiceBus](https://kubeedge.io/en/docs/architecture/edge/servicebus): ServiceBus是一个运行在边缘的HTTP客户端，接受来自云上服务的请求，与运行在边缘端的HTTP服务器交互，提供了云上服务通过HTTP协议访问边缘端HTTP服务器的能力。
- [DeviceTwin](https://kubeedge.io/en/docs/architecture/edge/devicetwin): DeviceTwin 负责存储设备状态并将设备状态同步到云，它还为应用程序提供查询接口。
- [MetaManager](https://kubeedge.io/en/docs/architecture/edge/metamanager): MetaManager 是消息处理器，位于 Edged 和 Edgehub 之间，它负责向轻量级数据库（SQLite）存储/检索元数据。

## 三 实战部署

### 3.1 keadm部署

注意事项：

- 目前支持`keadm`Ubuntu 和 CentOS 操作系统。RaspberryPi 支持正在进行中。
- 需要超级用户权限（或 root 权限）才能运行。



#### 3.1.1 设置云端（KubeEdge 主节点）

默认情况下`10000`，`10002`边缘节点需要可以访问 Cloudcore 中的端口和端口。

`keadm init`将安装 cloudcore，生成证书并安装 CRD。它还提供了一个可以设置特定版本的标志。

**重要说明：** 1. kubeconfig 或 master 中至少一个必须正确配置，以便用于验证 k8s 集群的版本和其他信息。1.请确保边缘节点可以使用云节点的本地IP连接云节点，或者您需要使用`--advertise-address`标志指定云节点的公共IP 。1. `--advertise-address`（1.3版本后才有效）是云端暴露的地址（会加入到CloudCore证书的SAN中），默认值为本地IP。

例子：

```shell
# keadm init --advertise-address="THE-EXPOSED-IP"(only work since 1.3 release)
```

输出：

```
Kubernetes version verification passed, KubeEdge installation will start...
...
KubeEdge cloudcore is running, For logs visit:  /var/log/kubeedge/cloudcore.log
```

#### 3.1.2 设置边缘端（KubeEdge 工作节点）

* 从云端获取令牌

`keadm gettoken`在**云端**运行将返回令牌，该令牌将在加入边缘节点时使用。

```shell
# keadm gettoken
27a37ef16159f7d3be8fae95d588b79b3adaaf92727b72659eb89758c66ffda2.eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE1OTAyMTYwNzd9.JBj8LLYWXwbbvHKffJBpPd5CyxqapRQYDIXtFZErgYE
```

* 加入边缘节点

`keadm join`将安装 edgecore 和 mqtt。它还提供了一个可以设置特定版本的标志。

例子：

```shell
# keadm join --cloudcore-ipport=192.168.20.50:10000 --token=27a37ef16159f7d3be8fae95d588b79b3adaaf92727b72659eb89758c66ffda2.eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE1OTAyMTYwNzd9.JBj8LLYWXwbbvHKffJBpPd5CyxqapRQYDIXtFZErgYE
```

* **重要说明：** 1. `--cloudcore-ipport`flag 是强制性标志。1. 如果要自动为边缘节点申请证书，`--token`则需要。1.云端和边缘端使用的kubeEdge版本要一致。

输出：

```shell
Host has mosquit+ already installed and running. Hence skipping the installation steps !!!
...
KubeEdge edgecore is running, For logs visit:  /var/log/kubeedge/edgecore.log
```

### 3.2 二进制部署

注意事项：

- 需要超级用户权限（或 root 权限）才能运行。

#### 3.2.1 设置云端（KubeEdge 主节点）

* 创建 CRD

```shell
kubectl apply -f https://raw.githubusercontent.com/kubeedge/kubeedge/master/build/crds/devices/devices_v1alpha2_device.yaml
kubectl apply -f https://raw.githubusercontent.com/kubeedge/kubeedge/master/build/crds/devices/devices_v1alpha2_devicemodel.yaml
kubectl apply -f https://raw.githubusercontent.com/kubeedge/kubeedge/master/build/crds/reliablesyncs/cluster_objectsync_v1alpha1.yaml
kubectl apply -f https://raw.githubusercontent.com/kubeedge/kubeedge/master/build/crds/reliablesyncs/objectsync_v1alpha1.yaml
```

* 准备配置文件

```shell
# cloudcore --minconfig > cloudcore.yaml
```

详情请参考[云配置](https://kubeedge.io/en/docs/setup/config#configuration-cloud-side-kubeedge-master)。

* 运行

```shell
# cloudcore --config cloudcore.yaml
```

#### 3.2.2 设置边缘端（KubeEdge 工作节点）

##### 3.2.2.1 准备配置文件

- 生成配置文件

```shell
# edgecore --minconfig > edgecore.yaml
```

- 在云端获取代币值：

```shell
# kubectl get secret -nkubeedge tokensecret -o=jsonpath='{.data.tokendata}' | base64 -d
```

- 更新 edgecore 配置文件中的令牌值：

```shell
# sed -i -e "s|token: .*|token: ${token}|g" edgecore.yaml
```

这`token`就是上面步骤得到的。

详情请参考[edge的配置](https://kubeedge.io/en/docs/setup/config#configuration-edge-side-kubeedge-worker-node)。

##### 3.2.2.2 运行

如果要在同一台主机上运行 cloudcore 和 edgecore，请先运行以下命令：

```shell
# export CHECK_EDGECORE_ENVIRONMENT="false"
```

启动边缘核：

```shell
# edgecore --config edgecore.yaml
```

运行`edgecore -h`以获取帮助信息并根据需要添加选项。

## 四 反思

K8s正在向边缘计算渗透，它为边缘侧的应用部署提供了便利性，在一定程度上转变了边缘应用与硬件之间的关系，将两者的耦合度降低。通过KubeEdge，拓展“边缘场景”，可帮助用户加速实现云边协同，在海量边、端设备上完成大规模应用的统一交付、运维与管控。

据Gartner估计，到2025年，超过75%的企业生成数据可以在传统数据中心和云之外创建和处理，像Kubernetes这样的编排系统前景光明，它已经被证明是完成这一任务的最佳工具。

## 参考资料

* https://github.com/kubeedge/kubeedge/blob/master/README_zh.md
* https://www.cncf.io/blog/2020/09/25/kubernetes-could-be-the-one-to-make-the-internet-of-things-iot-reach-its-potential/

