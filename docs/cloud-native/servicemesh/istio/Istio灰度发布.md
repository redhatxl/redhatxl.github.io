# Istio灰度发布

# 一 典型发布类型对比

* 蓝绿发布

![image-20191020172816421](/Users/xuel/Library/Application Support/typora-user-images/image-20191020172816421.png)

* 灰度发布（金丝雀发布）

![image-20191020172922080](/Users/xuel/Library/Application Support/typora-user-images/image-20191020172922080.png)

* A/B Test

![image-20191020172938789](/Users/xuel/Library/Application Support/typora-user-images/image-20191020172938789.png)

A/B Test主要对特定用户采样后，对收集的反馈数据做相关对比，然后根据对比结果作出决策，用来测试应用功能表现的方法，侧重应用的可用性，受欢迎程度等。

# 二 Istio

## 2.1 Virtualservice

定义一系列针对指定服务的流量路由规则

* hosts：流量的目标主机
* gateways：gateway名称列表
* http：http流量规则列表
* tcp：tcp流量规则
* tls：tls和https流量规则

![image-20191020174011021](/Users/xuel/Library/Application Support/typora-user-images/image-20191020174011021.png)

![image-20191020174017988](/Users/xuel/Library/Application Support/typora-user-images/image-20191020174017988.png)

# 三 发布流程

![image-20191020174359544](/Users/xuel/Library/Application Support/typora-user-images/image-20191020174359544.png)

# 四 智能灰度发布

## 4.1 目标：细粒度控制的自动化的持续交付

特点：

* 用户细分
* 流量管理
* 关机指标可以观测
* 发布流程自动化

## 4.2 智能灰度发布流程图

![image-20191020174737293](/Users/xuel/Library/Application Support/typora-user-images/image-20191020174737293.png)