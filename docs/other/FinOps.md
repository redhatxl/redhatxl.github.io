# FinOps

## 一 前言

根据调研机构 Gartner 公司的调查，到 2022 年底，全球各地的企业在云计算基础设施方面的支出约为 3330 亿美元。麦肯锡公司在调查报告中指出，每家企业的云计算预算平均超过了 23 %，并且浪费了30 %的支出。这些数字令人震惊，同时也引发企业更关注大量云成本支出的回报。云计算最终是增加了企业的成本还是物有所值？在业务同质化竞争的形势下，云基础设施的成本及投资、运营，也成为影响企业云业务市场竞争力的关键。

云成本并不一定意味着只有 IT 成本，还包括某些运营和管理成本。那么，企业如何进行上云成本优化？在这里我们就需要引入一个FinOps（云成本优化）的概念了。

## 二 FinOps相关概念

### 2.1 FinOps定义

> FinOps is an evolving cloud financial management discipline and cultural practice that enables organizations to get maximum business value by helping engineering, finance, technology and business teams to collaborate on data-driven spending decisions.

FinOps 是一个不断发展的云财务管理纪律和文化实践，它使组织能够通过帮助工程、金融、技术和业务团队在数据驱动的支出决策上进行协作，从而获得最大的业务价值。

在其核心，FinOps 是一种文化实践。这是团队管理云成本的方式，每个人都拥有一个中央最佳实践小组支持的云使用。工程、财务、产品等领域的跨职能团队共同努力，以加快产品交付速度，同时获得更多的财务控制和可预测性。

### 2.2 什么是FinOps？

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220417182351.png)

* FinOps(FinancialOperations)基金会隶属于Linux基金会，致力推进企业的云资产管理
* FinOps核心是确保企业在云中花费的每一分钱都获得最大价值
* 辅助企业做成本优化，降本增效，契合腾讯科技向善的核心价值观

### 2.3 FinOpsCommunity

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220417182605.png)

### 2.4 FinOps Slogan

看起来FinOps像是在指导关于如何省钱的方法，实际上FinOps是关于挣钱的方案。

* 保证高优核心业务资源的供应量
* 可以有效衡量业务的投入产出比，帮助决策
* 减少工程师对设备选型/运维/弹性的顾虑，增加研发/发布进度，增加系统稳态腾讯云原生

### 2.5 成熟度模型

FinOps 的实践本质上是迭代的，任何给定的过程、功能活动、能力或领域的成熟度都会随着重复而提高。典型的“爬行”阶段组织是高度反应性的，并且在问题发生后着重于解决问题，而运行阶段实践是主动地将成本因素考虑到他们的架构设计选择和正在进行的工程流程中。

执行 FinOps 的成熟度方法“爬行、行走、运行”使组织能够从小规模开始，并随着业务价值保证功能活动的成熟而在规模、范围和复杂性方面不断增长。在小范围和有限的范围内快速采取行动，使 FinOps 团队能够评估他们行动的结果，并获得以更大、更快、更细粒度的方式采取进一步行动的价值的洞察力。

## 三 企业痛点

### 3.1 IT资源利用率低

#### 3.1.1 行业整体

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220417181726.png)

#### 3.1.2 局部业务

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220417181806.png)

### 3.2 真实痛点

#### 阶段一

* 业务增长迅速，IT团队全力保障业务，资源申请来者不拒

#### 阶段二

* IT资源成本支出迅速提升，业务增长放缓，老板责令成本优化
* IT团队向老板要人，没人；业务团队向老板要人，没人；
* IT团队百忙之中建设资源运营平台、弹性平台、推动业务团队改造
* 业务团队在百忙之中挤出人力，优化架构、提升弹性、减少冗余

#### 阶段三

* 某次生产环境重大故障，影响客户
* 业务团队复盘认为是IT团队推动成本优化，架构冗余减少，挤占研发人力造成
* IT团队认为是业务技术能力差、提出不合理接入需求、团队人力少造成

#### 阶段四

* 业务与IT团队关系极度恶化
* 所有团队觉得成本优化吃力不讨好结局

#### 结局

* 业务再次出现故障->企业IT资源优化失败
* IT团队减少推动力度->企业IT资源优化失败
* 老板信心丧失->企业IT资源优化失败

## 四 最佳实践

### 4.1 企业上云初衷

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220417182936.png)





## 五 工具

### 5.1 腾讯Crane

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220417183139.png)

#### 5.1.1 Crane简介

Crane（CloudResourceAnalyticsandEconomics）由腾讯主导的，以FinOps理念为指导的，面向云原生的，一站式云成本优化开源项目。

#### 5.1.2 Crane能力简介

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220417183248.png)

#### 5.1.3 Crane架构

* 预测为王
  * 可扩展的预测算法
* 优化为本
  * 基于预测的资源再分配
  * 基于预测的成本可视化
  * 基于预测的横多维扩缩容
* 质保为根
  * 基于业务优先级的增强QoS
  * 干扰检测和主动回避

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220417183345.png)

#### 5.1.4 信息展示

* 成本可视化
  * 用量
  * 基于预测的趋势分析
* 成本和浪费识别
  * 与计费API整合的费用展示
* 灵活的汇聚维度
  * 按部门
  * 按项目
  * 按应用类型
  * 按自定义标签

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220417183856.png)

#### 5.1.5 成本优化

* 资源优化
  * 预测结果驱动
  * 为容器应用推荐资源请求
* EffectiveHPA
  * 预测结果驱动
  * 推荐指标
  * 以自定义指标驱动HPA
* InstanceVPA
  * 实时资源回收
  * 基于Linux隔离技术的动态资源调整

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220417184007.png)

#### 5.1.6 权衡降本增效和业务稳定

1. 调度手段

* 动态调度器DynamicScheduler

应用场景:集群负载不均原生调度器基于PodRequest来进行调度，没有考虑据Node的当前的真实以及历史负载情况，容易导致集群负载不均，

特点：

* 避免调度热点问题底层依赖Prometheus采集真实负载，代替request值
* 避免业务高峰节点负载不均引入节点历史负载指标(1h内最大利用率，1天内最大利用率)，感知业务波峰
* 原生无侵入基于原生KubeScheduler-Extender机制，版本高兼容

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220417185553.png)

* 重调度器Descheduler

应用场景:

a. 集群存量节点负载不均，需要进行资源再均衡

b. 集群加入新节点时，新节点利用率较低

特点

* 资源再均衡自动识别高负载节点上，驱逐pod，保证节点负载在目标水位。

* 安全保障Pod驱逐时执行筛选逻辑（重要pod不可被驱逐）用户手动标记的业务才会被驱逐

* 高可用兜底措施保障：只在workloadready的pod比例>50%时执行驱逐

2. QoS质量保证

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220417185712.png)

* 声明式的增强质量保证
  * 基于PriorityClass的业务定级
  * 节点资源水位定义
  * 业务SLO定义
* 干扰检测
  * 节点指标探测
  * 业务指标探测
* 灵活的主动回避
  * 节点调度降级与禁止
  * 基于业务优先级的资源压制
  * 基于业务优先级的主动驱逐

## 参考链接

* https://www.finops.org/introduction/what-is-finops/
* https://github.com/gocrane/crane