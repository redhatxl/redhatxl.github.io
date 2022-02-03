# CronHPA初探

# 一 背景

在目前大部分互联网企业而言，应用的负载呈现明显的峰谷分布，有一定的规律可循，例如订餐应用，在中午12点，下午6点，以及晚上9点等呈现明显的峰值流量，其应用负载也达到最高，不通类型的业务更具长时间的监控数据以及业务用户画像，能够精确的预知应用的高峰期，一般对于应用资源的波峰和波谷之间相差3-4倍，而且波峰来临可能不是缓慢上升，而是瞬间负载拉满。

对于传统的IDC业务，可能在活动大促或者业务推广期间会提前增加服务器，以横向扩展，提升服务整体服务能力，在云计算下，云厂商基本都有弹性伸缩服务，对于业务的横向扩展，可以将业务抽象为无状态应用，进而根据不通的监控指标来达到动态伸缩，对于大量预知并发，也可以设置固定时间，对后台计算资源进行定点扩所容，以达到动态的最小化成本满足应用业务需求。

在云原生中，容器的出现，可以让我们更加标准、轻量、自动的动态扩所容，在Kubernetes中，只用简单的修复replicase数即可完成，对于弹性扩所容只用定义HPA即可，根据不同的metric来动态调整副本数量，但是对于尖刺形的业务峰值，由于HPA中后端POD的拉起延迟，显然在这种场景是不能很好的满足用户需求。但是如果只是人工手动的提前扩展POD，对于这种周期性的业务显然也是不合理。

# 二 方案

对于标准的 HPA 是基于指标阈值进行伸缩的，常见的指标主要是 CPU、内存，当然也可以通过自定义指标例如 QPS、连接数等进行伸缩。但是这里存在一个问题：基于资源的伸缩存在一定的时延，这个时延主要包含：采集时延(分钟级) + 判断时延(分钟级) + 伸缩时延(分钟级)。而对于上图中，我们可以发现负载的峰值毛刺还是非常尖锐的，这有可能会由于 HPA 分钟级别的伸缩时延造成负载数目无法及时变化，短时间内应用的整体负载飙高，响应时间变慢。特别是对于一些游戏业务而言，由于负载过高带来的业务抖动会造成玩家非常差的体验。

为了解决这个场景，目前阿里提供了 `kube-cronhpa-controller`，专门应对资源画像存在周期性的场景。开发者可以根据资源画像的周期性规律，定义 time schedule，提前扩容好资源，而在波谷到来后定时回收资源。底层再结合 `cluster-autoscaler` 的节点伸缩能力，提供资源成本的节约。

## 2.2 cronhpa

Cronhpa ，利用底层使用的是`go-cron`,因此支持更多的规则。

```
  Field name   | Mandatory? | Allowed values  | Allowed special characters
  ----------   | ---------- | --------------  | --------------------------
  Seconds      | Yes        | 0-59            | * / , -
  Minutes      | Yes        | 0-59            | * / , -
  Hours        | Yes        | 0-23            | * / , -
  Day of month | Yes        | 1-31            | * / , - ?
  Month        | Yes        | 1-12 or JAN-DEC | * / , -
  Day of week  | Yes        | 0-6 or SUN-SAT  | * / , - ?    
```

`kubernetes-cronhpa-controller` is a kubernetes cron horizontal pod autoscaler controller using `crontab` like scheme. You can use `CronHorizontalPodAutoscaler` with any kind object defined in kubernetes which support `scale` subresource(such as `Deployment` and `StatefulSet`).

# 三 安装部署

## 3.1 下载相关资源

```yaml
git clone https://github.com/AliyunContainerService/kubernetes-cronhpa-controller.git
```

## 3.2 安装

* install CRD

```
kubectl apply -f config/crds/autoscaling_v1beta1_cronhorizontalpodautoscaler.yaml
```

* install RBAC settings

```yaml
# create ClusterRole 
kubectl apply -f config/rbac/rbac_role.yaml

# create ClusterRolebinding and ServiceAccount 
kubectl apply -f config/rbac/rbac_role_binding.yaml

[root@master ~]# kubectl api-resources  |grep cronhpa
cronhorizontalpodautoscalers      cronhpa      autoscaling.alibabacloud.com   true         CronHorizontalPodAutoscaler
```

* deploy kubernetes-cronhpa-controller

```
kubectl apply -f config/deploy/deploy.yaml
```

* verify installation

```
kubectl get deploy kubernetes-cronhpa-controller -n kube-system -o wide 

➜  kubernetes-cronhpa-controller git:(master) ✗ kubectl get deploy kubernetes-cronhpa-controller -n kube-system
NAME                            DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
kubernetes-cronhpa-controller   1         1         1            1           49s
```

# 四 测试

进入examples folder目录

* 部署创建workload和cronhpa

```
kubectl apply -f examples/deployment_cronhpa.yaml 
```

* 查看deploy副本数

```
kubectl get deploy nginx-deployment-basic 

➜  kubernetes-cronhpa-controller git:(master) ✗ kubectl get deploy nginx-deployment-basic
NAME                     DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment-basic   2         2         2            2           9s
```

* 查看cronhpa status

```
[root@master kubernetes-cronhpa-controller]# kubectl describe cronhpa cronhpa-sample 
'Name:         cronhpa-sample
Namespace:    default
Labels:       controller-tools.k8s.io=1.0
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"autoscaling.alibabacloud.com/v1beta1","kind":"CronHorizontalPodAutoscaler","metadata":{"annotations":{},"labels":{"controll...
API Version:  autoscaling.alibabacloud.com/v1beta1
Kind:         CronHorizontalPodAutoscaler
Metadata:
  Creation Timestamp:  2020-07-31T07:27:12Z
  Generation:          5
  Resource Version:    67753389
  Self Link:           /apis/autoscaling.alibabacloud.com/v1beta1/namespaces/default/cronhorizontalpodautoscalers/cronhpa-sample
  UID:                 3be95264-9b08-4efc-b9f9-db1aca7a80d8
Spec:
  Exclude Dates:  <nil>
  Jobs:
    Name:         scale-down
    Run Once:     false
    Schedule:     30 */1 * * * *
    Target Size:  1
    Name:         scale-up
    Run Once:     false
    Schedule:     0 */1 * * * *
    Target Size:  3
  Scale Target Ref:
    API Version:  apps/v1beta2
    Kind:         Deployment
    Name:         nginx-deployment-basic
Status:
  Conditions:
    Job Id:           184b967e-a149-488d-b4ae-765f9a792f93
    Last Probe Time:  2020-07-31T07:28:31Z
    Message:          cron hpa job scale-down executed successfully. current replicas:3, desired replicas:1.
    Name:             scale-down
    Run Once:         false
    Schedule:         30 */1 * * * *
    State:            Succeed
    Target Size:      1
    Job Id:           870b014f-cfbc-4f60-a628-16f2995c288d
    Last Probe Time:  2020-07-31T07:28:01Z
    Message:          cron hpa job scale-up executed successfully. current replicas:1, desired replicas:3.
    Name:             scale-up
    Run Once:         false
    Schedule:         0 */1 * * * *
    State:            Succeed
    Target Size:      3
  Exclude Dates:      <nil>
  Scale Target Ref:
    API Version:  apps/v1beta2
    Kind:         Deployment
    Name:         nginx-deployment-basic
Events:
  Type    Reason   Age   From                            Message
  ----    ------   ----  ----                            -------
  Normal  Succeed  87s   cron-horizontal-pod-autoscaler  cron hpa job scale-down executed successfully. current replicas:2, desired replicas:1.
  Normal  Succeed  57s   cron-horizontal-pod-autoscaler  cron hpa job scale-up executed successfully. current replicas:1, desired replicas:3.
  Normal  Succeed  27s   cron-horizontal-pod-autoscaler  cron hpa job scale-down executed successfully. current replicas:3, desired replicas:1.
```

在这个例子中，设定的是每分钟的第 0 秒扩容到 3 个 Pod，每分钟的第 30s 缩容到 1 个 Pod。如果执行正常，我们可以在 30s 内看到负载数目的两次变化。可以通过查看pod的增加和缩减情况，可以使得我们可以更容易的来定时控制POD数量，从而在波段性业务中，最大效率利用资源。

# 五 反思

目前对于无状态应用可以配合HPA，或CronHPA来应对横向扩展的压力，对于有状态应用，可借助Vertical Pod Autoscaler（VPA）来进行实现，根据业务用户画像，精确的预估容量水位，最小化的成本来自动化的满足业务需求。

# 参考链接

  * https://www.lagou.com/lgeduarticle/8604.html
* https://github.com/AliyunContainerService/kubernetes-cronhpa-controller