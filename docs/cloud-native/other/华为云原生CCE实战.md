# 华为云原生CCE实战

# 一 背景

近来和同事共同开发的迁移平台项目想进行容器化改造，顺应大趋势往容器化这边靠，项目前端平台利用Django开发，后端Restful API利用高性能Web框架Tornado完成，Agent端利用Flask开发，各取了几个大Python框架的优势。

之前为自己部署K8s，且需要保证集群高可用性，且要考虑从监控/日志/告警/CI/CD/调用链追逐/服务治理等，且需要提供分布式存储，为K8s集群提供存储类来工有状态引用持久化数据。	

## 1.1 云原生优势

更省：极大的资源利用效率， 最大限度榨取和共享物理资源，多项目更能体现出容器化多优势，节约部署IT成本。

更快：秒级启动，实现业务系统更快的开发迭代 和 交付部署。

弹性：可根据业务负载进行弹性容器伸缩，弹性扩展。

方便：容器化业务部署支持蓝绿/灰度/金丝雀等发布，回滚，更加灵活方便。

灵活：监控底层node节点健康状态，灵活调度至最优节点部署。

强一致性：容器将环境和代码打包在镜像内，保证了测试与生产环境的强一致性。

## 1.2 挑战

开发人员熟悉Docker虚拟化技术，熟练编写Dockerfile。

熟悉kubernetes容器化编排系统，  熟悉各组件资源清单编写。

开发需要考虑后期容器编排部署的需求来组织结构和编写代码。

部署人员需要熟悉kubernetes资源清单各参数含义，需要总体把控架构中到从上到下架构。

考虑高可用架构和rbac安全策略，外部流量引入及后期扩容伸缩。

## 1.3 希冀

希望可以有一套高可用开箱即用的容器管理平台，来使得我们更专注于业务的开发，不用过多的去将精力放在底层基建的部署维护上，

云容器引擎[（Cloud Container Engine，简称CCE）](https://www.huaweicloud.com/product/cce.html)提供高度可扩展的、高性能的企业级Kubernetes集群，支持运行Docker容器。借助云容器引擎，可以在华为云上轻松部署、管理和扩展容器化应用程序，来为企业释放更多精力，CCE提供一整套完整的最佳容器解决方案，赋能企业专注业务开发。

在此测试CCE功能，为更好将自己业务迁移到CCE，特此记录使用过程。

# 二 基础环境部署

由于是测试简单应用，基础环境为最小配置，且安全组等规则放行策略较为宽松，正式环境需要进行严格安全组规则配置，仅放行必要K8s组建端口及业务端口。

## 1.1 安全组配置

* 创建安全组

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211014131749.png)

* 针对安全组进行规则配置

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211014131800.png)

## 1.2 网络配置

配置Node节点网络

* 创建VPC

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211014131812.png)

* 创建子网

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211014131819.png)

# 三 CCE集群创建

云容器引擎(Cloud Container Engine)提供高性能可扩展的容器服务，基于云服务器快速构建高可靠的容器集群，深度整合网络和存储能力，兼容Kubernetes及Docker容器生态。帮助用户轻松创建和管理多样化的容器工作负载，并提供容器故障自愈，监控日志采集，自动弹性扩容等高效运维能力，用户无需过多关注基础设施高可用，将更多精力放在自身业务开发上。

## 2.1 选择K8s版本

* 版本选择

为了能够更好地方便您使用容器服务，确保您使用稳定又可靠的Kubernetes版本，云容器引擎CCE推出Kubernetes版本支持机制，将在每半年发布一个版本，每个版本的支持周期为一年，请您务必在维护周期结束之前升级您的Kubernetes集群。

* dev环境，集群环境可以先设置为50个

此为测试环境，选择集群较小，正式环境可根据业务来进行设置

* 高可用：
  * 高可用模式开启后将创建多个控制节点，在单个控制节点发生故障后集群可以继续使用，不影响业务功能。
  * 高可用模式开关在集群创建完成后不可变更
  * 商用场景建议选择高可用模式集群。
* 网络

网络是选择刚基础环境创建的VPC，K8s的Node节点网络就用的VPC内的子网，默认情况下，同一个VPC的所有子网内的弹性云服务器均可以进行通信，不同VPC的弹性云服务器不能进行通信。

* 网络模型

容器隧道网络下只能添加同一类型的节点，即全部为虚拟机节点或全部为裸金属节点。

基于底层VPC网络，另构建了独立的VXLAN隧道化容器网络，适用于一般场景。VXLAN是将以太网报文封装成UDP报文进行隧道传输。容器网络是承载于VPC网络之上的Overlay网络平面，具有付出少量隧道封装性能损耗，获得了通用性强、互通性强、高级特性支持全面（例如Network Policy网络隔离）的优势，可以满足大多数应用需求。

* RBAC

开启RBAC能力后，设置了细粒度权限的子用户使用集群下资源将受到权限控制。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211014131831.png)

## 2.2 Node云服务器选择

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211014131841.png)

* 安装插件

在插件着栏，非常关注volcano这块高性能调度器，Volcano是基于Kubernetes的批处理系统，源自于华为云AI容器，Volcano方便AI、大数据、基因、渲染等诸多行业通用计算框架接入，提供高性能任务调度引擎，高性能异构芯片管理，高性能任务运行管理等能力，真多高并发场景做了深入的优化和改进，内置多种算法，提供大数据，AI等应用场景解决方案。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211014131853.png)

## 2.3 提交购买

提交后等待集群部署完成。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211014131914.png)

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211014131924.png)

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211014131934.png)

如果为我们自己部署，考虑的东西非常多，而且可能部署异常，排错非常麻烦，CCE提供简单web界面配置，就可以部署出一个高可用的生成环境，非常的高效快捷。

# 四 访问

由于目前为配置弹性公务IP，使用web-terminal来访问，其也为一个pod，利用该pod可以通过浏览器打开web终端连接到集群中进行操作。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211014131947.png)![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211014131957.png)购买弹性IP

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211014132014.png)



* 配置本地kubectl

由于我本地之前已经安装贵哦kubectl，只需要下载创建的CCE技巧的kubeconfig.json移动到家目录.kube下面，重命名为config即可。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211014132027.png)

可以看到目前已经可以正常获取到两个node节点。

* 创建应用测试

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211014132037.png)

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211014132045.png)

* 暴露服务

创建svc

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211014132054.png)



* LB

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211014132105.png)

a

查看service

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211014132121.png)

在此就简单测试了从集群开通到单个应用的部署。后期可以将自己的应用迁移上来，统一管理。

# 五 反思

后期我们可以将自己的容器做好镜像，上传到SWR中，通过Devcloud中的部署，简单快捷的一键式部署应用至CCE中，同时可以web界面配置访问，非常快捷方便，华为云结合生成环境中经验，将最近解决方案作为服务，为用户提供开箱即用的应用解决方案。

其提供多种网络访问模式，支持四层/七层负载均衡使用不同场景，存储方便除了支持本地存储，提供丰富的存储类，灵魂的伸缩策略，可以应对业务高峰期的流量并发，深度集成K8s工具生态，提供Helm标准，将自己的应用制作成charts，轻松发布进K8s内，同时提供Istio服务治理能力，基础设施下沉到K8s内，提供流量控制，黑白名单，认证鉴权等功能，负载均衡/流量监控/链路追逐/熔断降级。

# 六 链接

* https://www.huaweicloud.com/product/cce.html