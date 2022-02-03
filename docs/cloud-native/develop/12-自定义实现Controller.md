# 12.自定义实现Controller

##一 前言

通过之前的文章，到目前可以对client-go中的Informer机制有了全局的认识，并针对其中每一个组件功能也有了深入的理解，本文将从一个实际例子，完整实现一个自定义控制器，加深对controller的认识。

##二 回顾Informer流程

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211224125637.png)

![图片](https://mmbiz.qpic.cn/mmbiz_png/gBXSicjuwMvEs91yTP3jb352WZL6FygfEBfC1ptz2NibqmKX0K4OCsNJeRxiamoriaIYCTm1FwsFk8YwEsEBsibEnXg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



* 首先在控制器启动时，通过运行Informer的run函数，执行Reflector中的run，全量的list kubernetes中的资源对象到deltaFIFO中。
* Reflector 通过 ListAndWatch 首先获取全量的资源对象数据，然后通过 syncWith 函数全量替换(Replace) 到 DeltaFIFO queue/items 中，，如果设置了定时同步，则定时更新Indexer中的内容，之后通过持续监听 Watch(目标资源类型) 增量事件，并去重更新到 DeltaFIFO queue/items 中，等待被消费。
* sharedIndexInformer的HandleDeltas处理从deltaFIFO pod出来的增量时，先尝试更新到本地缓存cache，更新成功的话会调用processor.distribute方法向processor中的listener添加notification，listener启动之后会不断获取notification回调用户的EventHandler方法。
* 当用户注册的EventHandler收到事件后，将其添加进workqueue中。
* 之后运行用户自定义的processNextItem，不断从workqueue中获取对象，并对其进行业务逻辑处理，如果处理发生异常，则重新入队列，如果处理成功，则forget调改对象。

## 三 控制器逻辑

K8s控制器是一主动的调协过程，watch一些资源对象期望状态，也会watch实际状态，之后通过控制器发送执行动作指令，让对象的当前状态往期望状态逼近。

```go
for {
  desired := getDesiredState()
  current := getCurrentState()
  makeChanges(desired, current)
}
```

更多的规则，可以去看k8s兴趣小组的控制器指南：https://github.com/kubernetes/community/blob/master/contributors/devel/sig-api-machinery/controllers.md

##四 编程实现 

* 编码

```go
// kubernetes/staging/src/k8s.io/client-go/examples/workqueue

// 定义控制器结构体，包含indexer/queue/informer
type Controller struct {
	indexer  cache.Indexer
  // 使用限速队列
	queue    workqueue.RateLimitingInterface
	informer cache.Controller
}

// 控制器构造方法
func NewController(queue workqueue.RateLimitingInterface, indexer cache.Indexer, informer cache.Controller) *Controller {
	return &Controller{
		informer: informer,
		indexer:  indexer,
		queue:    queue,
	}
}

// 从限速队列中获取一个执行syncToStdout业务路径进行处理
func (c *Controller) processNextItem() bool {
	// 从队列中获取一个元素进行处理
	key, quit := c.queue.Get()
	if quit {
		return false
	}
  // 告诉队列我们处理完这个键。这为其他worker解锁该key
  // 这允许安全的并行处理，因为具有相同key的两个 pod 永远不会并行处理。
	defer c.queue.Done(key)

	// 具体的业务逻辑方法处理key
	err := c.syncToStdout(key.(string))
	// 错误处理
	c.handleErr(err, key)
	return true
}

// syncToStdout 是控制器的业务逻辑。在这个控制器中，它只是将有关 pod 的信息打印到标准输出。如果发生错误，它必须简单地返回错误。重试逻辑不应该是业务逻辑的一部分。
func (c *Controller) syncToStdout(key string) error {
  // 从刚indexer中获取key
	obj, exists, err := c.indexer.GetByKey(key)
	if err != nil {
		klog.Errorf("Fetching object with key %s from store failed with %v", key, err)
		return err
	}

	if !exists {
		fmt.Printf("Pod %s does not exist anymore\n", key)
	} else {
		// 大家pod名称到控制台
		fmt.Printf("Sync/Add/Update for Pod %s\n", obj.(*v1.Pod).GetName())
	}
	return nil
}

// 检查是否发生错误并确保重试
func (c *Controller) handleErr(err error, key interface{}) {
	if err == nil {

    // 处理成功则Forget该key
		c.queue.Forget(key)
		return
	}

	// 如果出现问题，此控制器会重试 5 次。之后，它停止尝试。
	if c.queue.NumRequeues(key) < 5 {
		klog.Infof("Error syncing pod %v: %v", key, err)

		// 将key 从新入队列
		c.queue.AddRateLimited(key)
		return
	}
  // 如果还是异常，则forget该key
	c.queue.Forget(key)
	// 向外部实体报告，即使经过多次重试，我们也无法成功处理此密钥，丢弃了该key
	runtime.HandleError(err)
	klog.Infof("Dropping pod %q out of the queue: %v", key, err)
}

// controller的run函数
func (c *Controller) Run(threadiness int, stopCh chan struct{}) {
	defer runtime.HandleCrash()

	// 如果发生异常停止队列
	defer c.queue.ShutDown()
	klog.Info("Starting Pod controller")
  // 在goroutine中运行informer.Run()
	go c.informer.Run(stopCh)

	// 等待所有注册的informer都启动完成，开始运行队列
	if !cache.WaitForCacheSync(stopCh, c.informer.HasSynced) {
		runtime.HandleError(fmt.Errorf("Timed out waiting for caches to sync"))
		return
	}
  // 启动多个携程运行runWorker
	for i := 0; i < threadiness; i++ {
		go wait.Until(c.runWorker, time.Second, stopCh)
	}

	<-stopCh
	klog.Info("Stopping Pod controller")
}

// 不断的运行processNextItem
func (c *Controller) runWorker() {
	for c.processNextItem() {
	}
}

func main() {
	var kubeconfig string
	var master string

	flag.StringVar(&kubeconfig, "kubeconfig", "", "absolute path to the kubeconfig file")
	flag.StringVar(&master, "master", "", "master url")
	flag.Parse()

	// creates the connection
	config, err := clientcmd.BuildConfigFromFlags(master, kubeconfig)
	if err != nil {
		klog.Fatal(err)
	}

	// 获取clientSet
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		klog.Fatal(err)
	}

	// 创建podListWatcher
	podListWatcher := cache.NewListWatchFromClient(clientset.CoreV1().RESTClient(), "pods", v1.NamespaceDefault, fields.Everything())

	// 创建限速队列
	queue := workqueue.NewRateLimitingQueue(workqueue.DefaultControllerRateLimiter())

  // 创建IndexerInformer，获取indexer和informer
	indexer, informer := cache.NewIndexerInformer(podListWatcher, &v1.Pod{}, 0, cache.ResourceEventHandlerFuncs{
		AddFunc: func(obj interface{}) {
			key, err := cache.MetaNamespaceKeyFunc(obj)
			if err == nil {
				queue.Add(key)
			}
		},
		UpdateFunc: func(old interface{}, new interface{}) {
			key, err := cache.MetaNamespaceKeyFunc(new)
			if err == nil {
				queue.Add(key)
			}
		},
		DeleteFunc: func(obj interface{}) {
			// IndexerInformer uses a delta queue, therefore for deletes we have to use this
			// key function.
			key, err := cache.DeletionHandlingMetaNamespaceKeyFunc(obj)
			if err == nil {
				queue.Add(key)
			}
		},
	}, cache.Indexers{})
  // 构造controller
	controller := NewController(queue, indexer, informer)

	// 添加哥测试pod
	indexer.Add(&v1.Pod{
		ObjectMeta: meta_v1.ObjectMeta{
			Name:      "mypod",
			Namespace: v1.NamespaceDefault,
		},
	})

	// 启动controller
	stop := make(chan struct{})
	defer close(stop)
	go controller.Run(1, stop)

	// Wait forever
	select {}
}

```

* 运行测试

```go
add/update/delete pod:smartkm-api-k8s-exampledeploy-7cddf8cbf4-hn7f9
add/update/delete pod:etcd-1
add/update/delete pod:etcd-2
add/update/delete pod:etcd-0
add/update/delete pod:podname1
add/update/delete pod:smartkm-api-k8s-exampledeploy-7cddf8cbf4-ztv2k
add/update/delete pod:po
add/update/delete pod:po
Pod default/po does not exist anymore

```

当我们创建一个pod后，可以看到在控制台已经打印出我们的pod名称，删除pod后打印出不存在改对象。

## 参考链接

* https://mp.weixin.qq.com/s/-qiB1KilhwtcjI61m_x3jA

