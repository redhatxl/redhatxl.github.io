# Istio笔记之Gateway

# 一 Gateway简介

在Istio中，gateway控制着网络边缘的服务暴露，可以将其看作网格的负载均衡器，提供一下功能

1. L4-L6的负载均衡
2. 对外的双向TLS

Istio服务网格中，gateway 可以部署多个，也可以公用一个，可以每个租户，namespace隔离

# 二 Istio的gateway与kubernetes Ingress的区别

kubernetes ingress集群边缘负载均衡，提供集群内部服务的访问入口，提供L7负载均衡，功能单一

Istio 1.0之前，利用kubernetes ingress实现网格内服务暴露，但是功能较单一

1. L4-L6负载均衡
2. 对外双向TLS
3. SNI支持
4. 其他Istio中以及实现内部网络功能，Falut Injection（故障注入），Traffic Shifting（流量转移），Circuit Breaking（断路器），Mirroring（镜像）

为了解决这些问题，istio在新版本设计了api来解决。

1. gateway允许管理员指定L4-L6的设置：端口及TLS设置
2. 对于ingress的设置，istio允许virtualservice 与gateway绑定起来
3. 分离的好处：用户可以像使用传统负载均衡器设备一样管理进入网格的流量，绑定虚拟IP到虚拟服务器上面，便于传统技术无缝迁移到微服务。

# 三 gateway原理与实现



![image-20191019190438706](/Users/xuel/Library/Application Support/typora-user-images/image-20191019190438706.png)

![image-20191020103320085](/Users/xuel/Library/Application Support/typora-user-images/image-20191020103320085.png)

gateway与普通sidecar均使用envoy作为proxy实行流量控制，pilot为不同类型proxy生成相应配置，gateway的类型为route，sidecar类型为sidecar。

* Pilit如何得知proxy类型

envoy发现服务使用xDS协议，envoy向server端的pilot发起请求discoverrequest时会携带自身的node信息，node信息有一个ID标识，pilot会解析node标示获取proxy类型。

![image-20191020103824973](/Users/xuel/Library/Application Support/typora-user-images/image-20191020103824973.png)

enovy的节点标示可以通过静态配置文件指定，也可以通过启动参数—service-node指定

* gateway对象在istio中是使用CRD声明，可以通

# 四 gateway配置下发

遵循make-before-break原则，杜绝规则更新过程中出现503

![image-20191020104347288](/Users/xuel/Library/Application Support/typora-user-images/image-20191020104347288.png)

route类型与sidecar类型的proxy本质没有区别，inbound cluster，endpoint，listener

这里也可以说明，gateway不是流量的终点，而只是充当一个代理转发



# 五 外部请求到达应用

* client发起请求到特定端口
* load banlancer监听在此端口，并转发到后端
* 在istio中，lb将请求转发到ingress gateway服务
* service将请求转发到ingress gateway pod
* pod获取gateway和virtual service配置，获取端口，协议，证书，创建监听器
* gateway pod根据路由将请求转发到应用pod（不是service）



控制ingress http流量

![image-20191019194404601](/Users/xuel/Library/Application Support/typora-user-images/image-20191019194404601.png)