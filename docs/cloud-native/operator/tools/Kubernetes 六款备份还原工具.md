# Kubernetes 六款备份还原工具

# 一 背景

传统的备份和灾难恢复应用程序往往不能很好地在容器化环境中发挥作用。这些类型的数据保护产品常常侧重于保护单个服务器以及运行在上面的应用程序。但是在Kubernetes环境中，应用程序通常是广泛分布的，有时需要启动多云和多个数据中心。此外，容器往往具有高度瞬时性，这也给备份应用带来重大挑战。因此，充分保护这些应用的唯一方法就是使用专门为Kubernetes打造的备份工具。

虽然不像传统的备份和灾难恢复的产品那样普遍，但如今也出现了许多Kubernetes备份工具可供使用。Veeam在宣布与Kasten合作提供Kubernetes原生备份产品几个月后，于9月25日收购了Kasten，这加速了两家公司产品的整合。同样，Zerto也宣布了其Zerto for Kubernetes产品，该产品将于2021年上市。Cohesity最近也推出了自己的保护Kubernetes应用。很明显，Kubernetes在备份领域正在获得广泛的应用。

在本文中，我们将研究6款知名的Kubernetes备份工具的特点和功能。在了解具体的备份工具之前，我们先来看看选择Kubernetes备份工具需要考虑的主要因素：

- **数据保护需求：**一个企业应该首先考虑其Kubernetes架构和备份需求。例如，如果它使用的是基于云的Kubernetes集群，它将需要选择一个能在该企业使用的云中工作的产品。同样，一个拥有多个集群的企业可能需要选择一个能够进行跨集群恢复的产品。
- **主要特性：**功能特性是主要的考虑因素。请记住，不是每个Kubernetes备份产品都有GUI界面或者都能够创建调度的备份。一家企业在确定一个产品之前，必须考虑哪些功能是最重要的。
- **安全和合规性：**企业选择一个与安全和合规性相匹配的产品非常重要。一些Kubernete备份工具高度重视安全，而其他产品则比较实用，没有提供安全方面的功能。
- **总成本：**价格当然也是一个十分重要的考虑因素，但企业需要了解到总成本也许不仅仅只包括订阅的价格。厂商也许还会要求企业购买技术支持。
- **先试后买：**一旦企业缩小了其潜在选项的列表，它应该了解这些Kubernetes备份工具是否提供免费试用。这将使企业有机会在承诺购买之前充分评估该产品是否满足其需要。



# 二 工具

## 2.1 Rancher

Rancher Labs并不是一家备份产品厂商，而是一家为使用容器的企业提供完整解决方案堆栈的公司。其旗舰产品Rancher是目前业界应用最为广泛的Kubernetes管理平台，该产品提供Kubernetes备份功能。



Rancher是为了与各种容器软件一起使用而构建的，因此其备份的工作方式也有所不同，这取决于企业使用的是哪种类型的容器。例如，使用Docker容器的企业需要使用一些系列Docker命令来创建一个tarball，作为Rancher server数据的备份。



Rancher目前支持对其K3s发行版、RKE和Docker部署进行备份。进行备份之后，如果你误删了某些节点或集群，可以根据以下教程操作恢复：

[误删节点或集群怎么办？这里有一颗后悔药](http://mp.weixin.qq.com/s?__biz=MzIyMTUwMDMyOQ==&mid=2247494297&idx=1&sn=e6a5dc5d73577306c2473455dec7676a&chksm=e8396c5fdf4ee549c265d90baecfcff1c71aed869b2f9535b12090853424eb17d23084139992&scene=21#wechat_redirect)

## 2.2 KubeDR

KubeDR是Catalogic Software公司的开源Kubernetes备份工具，这个工具是为了简化复杂的Kubernetes部署备份任务而开发的。KubeDR不需要管理员标记他们需要保护的资源，而是自动备份集群配置、其元数据和证书。

KubeDR也简化了备份流程。由于它备份了所有的集群组件，它可以执行自动集群恢复，并轻松重建一个故障的集群。

KubeDR备份Kubernetes数据到AWS S3并且支持使用保留政策。该软件还提供了自动清理、根据需要暂停和恢复备份等功能。

* https://github.com/catalogicsoftware/kubedr
* https://catalogicsoftware.com/clab-docs/kubedr/userguide/



## 2.3 Kasten K10

Kasten Inc.将其Kubernetes数据保护产品K10推向市场，但它不仅仅是一个Kubernetes备份产品，而是一个数据管理平台。K10是构建在企业的Kubernetes集群上自己的命名空间内运行的，并支持所有主要的Kubernetes发行版。它带有一个直观的、基于GUI仪表板的管理工具，支持命令行。该软件通过自动发现应用程序，包括那些跨卷或数据库的应用程序，简化了备份过程。此外，K10还包括许多安全功能，如基于角色的访问控制（RBAC）和对静止和活动的数据进行加密。

最值得一提的是，K10对各种行业标准的支持。除了支持众多Kubernetes发行版，K10还支持PostgreSQL、MongoDB、MySQL、Cassandra和AWS关系型数据库服务等数据服务。此外，它还支持众多厂商和云提供商的存储，包括NetApp、微软等。

## 2.4 Portworx PX-Backup

PX-Backup是由Portworx推出的一款企业级应用，专为Kubernetes提供数据保护。它的构建是为了在命名空间、Pod或标签级别上备份任何Kubernetes应用，并与多个命名空间一起工作。



PX-Backup的可扩展性使管理员能够按需创建备份，或一键轻松调度数百个应用程序的备份。PX-Backup支持跨越多个数据库的应用，并能将应用还原到微软、亚马逊或谷歌云端。

## 2.5 TrilioVault

Trilio的云原生Kubernetes数据保护产品TrilioVault是一款可无限扩展的备份产品，管理员可以从红帽OperatorHub安装。该软件可以执行基于快照的备份和还原操作到云端。然而，TrilioVault是没有任何云锁定的，这意味着管理员可以从任何云（甚至是混合云）上运行的Kubernetes或OpenShift集群中备份应用程序或将其还原。



TrilioVault支持按需或计划增量备份。它提供容器和应用程序的时间点恢复。管理员可以将应用程序恢复到其原始命名空间或不同的命名空间。此外，TrilioVault还能让授权用户执行自助式应用恢复。

## 2.6 Velero

Velero是一个开源的Kubernetes集群的备份和灾难恢复工具。它使管理员能够运行整个集群或单个命名空间或者是标签的计划备份。此外，它的Backup Hooks功能让管理员可以选择在备份之前或之后执行自定义操作，以及对备份应用保留计划。



除了作为Kubernetes备份和灾难恢复工具外，企业还可以将Velero作为迁移工具。Velero允许管理员将Kubernetes资源从一个集群移动到另一个集群。





# 参考链接

* https://mp.weixin.qq.com/s/rezN7FHBCtSGyBko4ZLjJg