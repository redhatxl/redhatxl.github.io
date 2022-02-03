# Inside Kubernetes controller

# 前言



# 一 高层架构

## 1.1 K8s架构

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210924163658.png)

* Api-server：api-server 负责接收来自客户端的对资源对象的CRUD请求，并去执行。

* controller-manager：
  * 控制管理资源（deployment，service，replicaset，daemonset，job，cronjob，endpoint，statefulset）
  * 是控制器的集合组，打包到同一个二进制文件中。

## 1.2 Controller 相关

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926104335.png)

### 1.2.1 控制循环

A. 控制循环概念：也称为调谐，通过资源的期望状态和真实状态通过一些了的观测、分析、执行对应动作，使得真实状态往期望状态逼近。

B. 控制循环流程：

* 获取当前资源的真实状态。
* 改变资源状态到期望状态。
* 更新资源状态。

### 1.2.2 ReplicaSet Control 循环示例

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926104949.png)

如图replicaset 控制器查看期望为3个副本，目前2个，所以执行动作增加一个副本。

#### 1.2.2.1 POD 创建

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926105252.png)

1. 用户通过kubect应用了一个replicaset 资源清单，api-server将其存储到etcd中，replicaset 资源创建。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926105358.png)

2. Replicates 控制器检测到replicaset资源创建，没有spec.nodeName 的空pod开始创建，kubelet检测到pod但是目前还不知道该pod被调度到那个node上面执行。通过schedule开始调度。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926105552.png)

3. 调度器检测到pod被创建，由于pod.spec.nodeName为空，调度器将其添加到调度队列中，后期进行调度算法进行调度。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926110032.png)

4. kubelet检测到pod创建事件，由于pod.specc.nodeName为空，先跳过处理。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926110143.png)

5. 调度器从调度队列中弹出pod，开始为pod分配node节点，之前更新pod的pod.Spec.nodeName字段

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926110313.png)

6. Kubelet 检测到pod更新，选择到podName 的node节点，开始启动容器。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926110537.png)

7. kubelet 发送请求给api-server, api-server 更新pod的字段 pod.status到etcd中。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926110636.png)

#### 1.2.2.1 POD 销毁

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926110754.png)

现在由于某种情况POD开始销毁。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926111723.png)

8. kubelet 发送请求到api-server 上，api-server 更新etcd中pod.Status 字段。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926111806.png)

9. ReplicaSet Controller 检测到Pod更新，开始执行Pod删除。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926111857.png)

10. replicaset controller 调谐Pod到期望状态，开始循环2-10步骤。

api-server负责资源crud，schedule负责资源调度，kubelet负责容器创建，controller负责调谐资源，每个组件都专注于自己的职责，组成一个有机的统一体。



## 1.3 事件

什么时候控制器开始执行控制循环呢，Event是非常关键的因素，例如：Added、Modifie、Deleted，Error...

对于Kubeernetes，这种有分布式组件组成的系统，Event是非常的重要，通过他去联系其他的组件。

事件触发在k8s中被分为两类，一个是Edge-driven Triggers(边缘触发)，Level-driven Triggers(水平触发)。

### 1.3.1 边缘触发

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926113200.png)

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926113652.png)

边缘触发就是当资源副本数量的增加或缩减。

但是仅仅有边缘触发会存在一定问题，假设由于网络原因或程序bug，边缘事件发生丢失，那么调谐就没有发生，资源副本数量也就未发生变化。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926114129.png)

### 1.3.2 水平触发

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926113216.png)

垂直触发会遇到这种丢失的问题，控制器在每次重新同步间隔时进行协调，这样控制器可以使状态更接近期望状态。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926135939.png)

## 1.4 控制器的组件

### 1.4.1 术语表

* Kind：API 对象的种类，例如deployment，service。
* Resource：kind中资源的实体，被作为http endpoint，以小写或复数形式表示。
* Object：API 对象的实体，被持久化存储在etcd中。

### 1.4.2 控制器的库

* Client-go：
  * informer
  * lister
  * workQueue
* api-machinery：
  * runtime.Object
  * scheme
* Code-generator

### 1.4.3 自定义控制器SDK

* 框架：kubebuilder、Operator SDK
* 高级库：controller-runtime、controller-tools
* 低级库：client-go、api-machinery
* 组件：informer、lister、workqueue、scheme、runtime.Object

### 1.4.4 控制器的库

* Client-go：
  * kubernetes 官方client库。
  * 这个是kubernetes开发工具。
* Api-machinery：
  * kubernetes API 对象 & kubernetes API 对象库
  * 控制器管理API 对象使用。
* Code-generator：
  * informer，lister，clientset，deepcopy 源码生成，使用这个进行控制器开发。

### 1.4.5 控制器下的组件

* Informer：watch 对象事件和超难吃数据在内存缓存中。
* Lister：吸收对象数据在内存缓存中。
* WorkQueue：队列存储控制循环item。
* runtime.Object：API 对象接口。
* Scheme：联系go type 和kubernetes API。

# 二 低层架构-client-go

## 2.1 Informer

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926163457.png)

Informer watches 对象事件（added，updated，deleted）

当控制器每次向api服务器查询对象状态时，监视对象更改，api服务器高负载。

informer watch api，通过controller设置关联cache。

```go
func main() {
 ...
 clientset, err := kubernetes.NewForConfig(config)
 // Create InformerFactory
 informerFactory := informers.NewSharedInformerFactory(clientset, time.Second*30)
 // Create pod informer by informerFactory
 podInformer := informerFactory.Core().V1().Pods()
 // Add EventHandler to informer
 podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
 AddFunc: func(new interface{}) { log.Println("Added") },
 UpdateFunc: func(old, new interface{}) { log.Println("Updated") },
 DeleteFunc: func(old interface{}) { log.Println("Deleted") },
 })
 // Start Go routines
 informerFactory.Start(wait.NeverStop)
 // Wait until finish caching with List API
 informerFactory.WaitForCacheSync(wait.NeverStop)
 // Create Pod Lister
 podLister := podInformer.Lister()
 // Get List of pods
 _, err = podLister.List(labels.Nothing())
 …
}
```

### 2.1.1 Shared Informer

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926165524.png)

当我们使用Informer，我们不直接使用informer自己，而是使用共享informer。

共享informer 共享相同的资源在一个单独的二进制文件中。

### 2.1.2 Informer 和workqueue

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926171048.png)

1. Reflector list & watch api-server
2. Reflector将对象添加到Delta FIFO 队列中
3. informer 从Delta FIFO中弹出对象
4. 将对象添加到Indexer
5. Indexer 存储对象到安全线程存储中
6. informer 设置事件reference
7. 用户自定义的Controller，事件Handlers 将对象添加至workqueue，
8. 进程获取key，process item
9. 获取对象的key从indexer reference中。

### 2.1.3 Informer 详解

Informer 整体架构

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926172230.png)

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926181905.png)

1. reflector 开始listAndWatch api对象中的用户创建的心资源pod对象。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926181956.png)

2. informer 从DeltaFIFO 队列中Pop初事件。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926182330.png)

3. HandleDelta

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926182417.png)

4. indexer.Add 增加事件到内存存储中

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926182519.png)

5. 开始循环弹出事件并进行处理

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926185156.png)

6. 后期get or list 请求都从indexer中获取。



 Informer 和其组件

* Informer：watch 一个对象事件并且存储在内存缓存中
* Reflector：listAndWatch api-server
* DeltaFIFO：FIFO 队列存储对象临时
* Indexer：设置或者获取对象
* Store：存储对象
* Lister：获取对象在内存通过index

## 2.2 WorkQueue

WorkQueue不同于DeltaFIFO队列，

WorkQueue是使用存储Contrl Loop 的item。

调谐将执行WorkQWueue中的内容。

感觉的控制存储enqueues item 到工作队列中，当事件发生时。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926193053.png)

### 2.2.1 WorkQueue 简单代码

```go
func main() {
 ...
 clientset, err := kubernetes.NewForConfig(config)
 // Create InformerFactory
 informerFactory := informers.NewSharedInformerFactory(clientset, time.Second*30)
 // Create pod informer by informerFactory
 podInformer := informerFactory.Core().V1().Pods()
 // Create RateLimitQueue
 queue := workqueue.NewRateLimitingQueue(workqueue.DefaultControllerRateLimiter())
 // shutdown when process ends
 defer queue.ShutDown()
 // Add EventHandler to informer
 podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
 AddFunc: func(old interface{}) {
var key string
var err error
if key, err = cache.MetaNamespaceKeyFunc(old); err != nil {
runtime.HandleError(err)
return
}
queue.Add(key)
log.Println("Added: " + key)
},
 UpdateFunc: func(old, new interface{}) { … },
 DeleteFunc: func(old interface{}) { … },
 })
 …
}
```

### 2.2.2 WorkQueue 详解

整体示意图

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926193202.png)

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926193234.png)

1. Reflector watch k8s-api 将其中资源发送到DeltaFIFO中

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926194623.png)

2. Informer EventHandler 从DeltaFIFO中pop出数据。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926194815.png)

3. Informer EventHandler 调用资源事件函数，例如：AddFunc、UpdateFunc、DeleteFunc来将事件对象放进WorkQueue中。

注意：再次添加的key为资源的唯一标示，后期使用的时候从index中之间获取，从而减轻api-server 的压力。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926194955.png)

4. workqueue.Add 进入WorkQueue中，例如namespace/name、default/nginx

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926195208.png)

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926195239.png)

循环3-4，处理事件放入队列中。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926195404.png)

5. 用户自定义的controller 逻辑从WorkQueue中获取对象进行处理。

### 2.2.3 Informer Resync Period

Informer 重新同步周期

重新同步周期对于Informer是可选的。

Informer监视api服务器上的对象事件。

再同步期过后，无论发生什么事件，UpdateFunc被回调。结果，调谐将再次执行。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926200009.png)

## 2.3 Controller Cycle Main Logic

完整示意图：

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926200942.png)

### 2.3.1 Controller Cycle

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926201017.png)

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926201106.png)

1. 假设用户创建了两个pod，生成了两个update 事件

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926201129.png)

2. informer 执行UpdateFunc

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926201208.png)![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926201402.png)

3. Informer 调用workQueue.Add 函数 增加事件到WorkQueue中。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926201416.png)![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926201557.png)

4. 程序处理调用workqueue.Get 获取事件

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926201611.png)

5. 调谐处理具体更新业务逻辑。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926201653.png)

6. 当调谐成功后，处理程序进行workQueue.Forget/workQueue.Done，这个item从workQueue中移除。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926201932.png)![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926202359.png)

当调谐是错误结束后，workQueue 增加速率限制，并且控制器重新入队item，并且开始新一轮调谐。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926203307.png)

每次事件发生时，项都会继续存储在工作队列中,控制器处理工作队列中的项并执行协调。该循环持续不断，直到控制器停止。

### 2.3.2 控制器基本策略

从内存中读取，写向api-server。

但是，如果我们直接更新缓存中的对象，很难保证其一致性。

所以，我们在更新对象时使用DeepCopy（获取克隆数据）。

例如代码：kubernetes/pkg/controller/replicaset/replica_set.go

```go
rs = rs.DeepCopy()
newStatus := calculateStatus(rs, filteredPods, manageReplicasErr)
// Always updates status as pods come up or die.
updatedRS, err := updateReplicaSetStatus(rsc.kubeClient.AppsV1(). ⇨
ReplicaSets(rs.Namespace), rs, newStatus)
```



### 2.3.3 控制器主逻辑

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210926203743.png)

* worker：无限循环 processNextWorkItem。
* processNet WorkItem：操作WorkQueue(Get, Add)并且调用调谐逻辑。
* syncHandler：这就是调谐具体逻辑。

### 2.3.4 ReplicaSet Controller 源码

基于K8s v1.16

* worker：https://github.com/kubernetes/kubernetes/blob/release-1.16/pkg/controller/replicaset/replica_set.go#L432
* processNextWorkItem：https://github.com/kubernetes/kubernetes/blob/release-1.16/pkg/controller/replicaset/replica_set.go#L437
* syncReplicaSet：https://github.com/kubernetes/kubernetes/blob/release-1.16/pkg/controller/replicaset/replica_set.go#L562

# 其他

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210927111215.png)

Informer 从etcd 同步对象数据到内存中。

再次需要考虑内存数据是否和etcd数据不相同呢。

这是没有问题的，对象有resourceVersion，如果etcd的resourceVersion 于内存缓存中的resourceVersion不同，Controller在更新对象状态时出错，控制器请求调谐，直到迁移完成。

# review

* Informer：通过eventandler将Control Loop的项添加到WorkQueue中
* Lister：通过Indexer从内存缓存中获取对象数据
* WorkQueue：存储控制循环项的队列。该项是Reconcile Logic的目标。如果在调解结束时发生错误，控制器将项请求到工作队列。控制器再次执行Reconcile。

