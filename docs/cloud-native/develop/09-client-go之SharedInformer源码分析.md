# 9.Client-go源码分析之SharedInformer

## 一 前言

上篇单独就Informer中的controller来看，processFunc以一个参数单独穿入NewInformer中，如果有另一个程序需要处理相同的资源，那么就需要另外再创建一个Informer对象，而队列也无法复用，队列不能被两个消费者同时消费，因此在Client-go中又设计有ShareInformer，后续的示例包括K8s的控制器中也都适用的是此类共享型的对象。

## 二 相关概念

### 2.1 资源Informer

- 每一种资源都实现了Informer机制，允许监控不同的资源事件
- 每一个Informer都会实现Informer和Lister方法

```javascript
type PodInformer interface {
  Informer() cache.SharedIndexInformer
  Lister() v1.PodLister
}
```

### 2.2 SharedInformer

若同一个资源的Informer被实例化了多次，每个Informer使用一个Reflector，那么会运行过多相同的ListAndWatch，太多重复的序列化和反序列化操作会导致api-server负载过重

SharedInformer可以使同一类资源Informer共享一个Reflector。内部定义了一个map字段，用于存放所有Infromer的字段。

通常会使用informerFactory来管理控制器需要的多个资源对象的informer实例，例如创建一个deployment的Informer

```go
// 创建一个informer factory
sharedInformerFactory := informers.NewSharedInformerFactory(clientSet, 0)
// factory已经为所有k8s的内置资源对象提供了创建对应informer实例的方法，调用具体informer实例的Lister或Informer方法
// 就完成了将informer注册到factory的过程
deploymentLister := sharedInformerFactory.Apps().V1().Deployments().Lister()
// 启动注册到factory的所有informer
kubeInformerFactory.Start(stopCh)
```

SharedInformer是一个接口，包含添加事件，当有资源变化时，会回掉通知使用者，启动函数及获取是否全利卿对象已经同步到本地存储中。

```go
type SharedInformer interface {
    // 添加资源事件处理器，当有资源变化时就会通过回调通知使用者
    AddEventHandler(handler ResourceEventHandler)
    AddEventHandlerWithResyncPeriod(handler ResourceEventHandler, resyncPeriod time.Duration)
    
    // 获取一个 Store 对象
    GetStore() Store
    // 主要是用来将 Reflector 和 DeltaFIFO 组合到一起工作
    GetController() Controller

    // SharedInformer 的核心实现，启动并运行这个 SharedInformer
    // 当 stopCh 关闭时候，informer 才会退出
    Run(stopCh <-chan struct{})
    
    // 告诉使用者全量的对象是否已经同步到了本地存储中
    HasSynced() bool
    // 最新同步资源的版本
    LastSyncResourceVersion() string
}

// SharedIndexInformer在SharedInformer基础上扩展了添加和获取Indexers的能力
type SharedIndexInformer interface {
    SharedInformer
    // 在启动之前添加 indexers 到 informer 中
    AddIndexers(indexers Indexers) error
    GetIndexer() Indexer
}
```

## 三 源码分析

### 3.1 SharedInformerFactory

SharedInformerFactory 为所有已知 API 组版本中的资源提供共享informer

```go
type SharedInformerFactory interface {
	internalinterfaces.SharedInformerFactory
	ForResource(resource schema.GroupVersionResource) (GenericInformer, error)
	WaitForCacheSync(stopCh <-chan struct{}) map[reflect.Type]bool

	Admissionregistration() admissionregistration.Interface
	Internal() apiserverinternal.Interface
	Apps() apps.Interface
	Autoscaling() autoscaling.Interface
	Batch() batch.Interface
	Certificates() certificates.Interface
	Coordination() coordination.Interface
	Core() core.Interface
	Discovery() discovery.Interface
	Events() events.Interface
	Extensions() extensions.Interface
	Flowcontrol() flowcontrol.Interface
	Networking() networking.Interface
	Node() node.Interface
	Policy() policy.Interface
	Rbac() rbac.Interface
	Scheduling() scheduling.Interface
	Storage() storage.Interface
}
```

这里的informer实现是shareIndexInformer NewSharedInformerFactory调用了NewSharedInformerFactoryWithOptions，将返回一个sharedInformerFactory对象

```go
type sharedInformerFactory struct {
	client           kubernetes.Interface
  //关注的namepace，可以通过WithNamespace Option配置
	namespace        string
	tweakListOptions internalinterfaces.TweakListOptionsFunc
	lock             sync.Mutex
	defaultResync    time.Duration
	customResync     map[reflect.Type]time.Duration
  //针对每种类型资源存储一个informer，informer的类型是ShareIndexInformer
	informers map[reflect.Type]cache.SharedIndexInformer
	// startedInformers is used for tracking which informers have been started.
	// This allows Start() to be called multiple times safely.
	startedInformers map[reflect.Type]bool
}

// SharedInformerFactory 构造方法
func NewSharedInformerFactoryWithOptions(client kubernetes.Interface, defaultResync time.Duration, options ...SharedInformerOption) SharedInformerFactory {
	factory := &sharedInformerFactory{
		client:           client,
		namespace:        v1.NamespaceAll,
		defaultResync:    defaultResync,
    // 初始化map
		informers:        make(map[reflect.Type]cache.SharedIndexInformer),
		startedInformers: make(map[reflect.Type]bool),
		customResync:     make(map[reflect.Type]time.Duration),
	}

	// Apply all options
	for _, opt := range options {
		factory = opt(factory)
	}

	return factory
}
```

### 3.2 Start

启动factory下的所有informer，其实就是启动每个informer中的Reflector

```go
// Start initializes all requested informers.
func (f *sharedInformerFactory) Start(stopCh <-chan struct{}) {
	f.lock.Lock()
	defer f.lock.Unlock()
  // 启动informers中所有informer
	for informerType, informer := range f.informers {
		if !f.startedInformers[informerType] {
			go informer.Run(stopCh)
			f.startedInformers[informerType] = true
		}
	}
}
```

### 3.3 WaitForCacheSync

- sharedInformerFactory的WaitForCacheSync将会不断调用factory持有的所有informer的HasSynced方法，直到返回true
- 而informer的HasSynced方法调用的自己持有的controller的HasSynced方法（informer结构持有controller对象，下文会分析informer的结构）
- informer中的controller的HasSynced方法则调用的是controller持有的deltaFIFO对象的HasSynced方法

也就说sharedInformerFactory的WaitForCacheSync方法判断informer的cache是否同步，最终看的是informer中的deltaFIFO是否同步了。

```go
func (f *sharedInformerFactory) WaitForCacheSync(stopCh <-chan struct{}) map[reflect.Type]bool {
	informers := func() map[reflect.Type]cache.SharedIndexInformer {
		f.lock.Lock()
		defer f.lock.Unlock()

		informers := map[reflect.Type]cache.SharedIndexInformer{}
		for informerType, informer := range f.informers {
			if f.startedInformers[informerType] {
				informers[informerType] = informer
			}
		}
		return informers
	}()

	res := map[reflect.Type]bool{}
	for informType, informer := range informers {
    // 调用Informer中的HasSynced
		res[informType] = cache.WaitForCacheSync(stopCh, informer.HasSynced)
	}
	return res
}
```

### 3.4 InformerFor

只有向factory中添加informer，factory才有意义，obj: informer关注的资源如deployment{} newFunc: 一个知道如何创建指定informer的方法，k8s为每一个内置的对象都实现了这个方法，比如创建deployment的ShareIndexInformer的方法

```go
func (f *sharedInformerFactory) InformerFor(obj runtime.Object, newFunc internalinterfaces.NewInformerFunc) cache.SharedIndexInformer {
	f.lock.Lock()
	defer f.lock.Unlock()
  // 获取对象类型
	informerType := reflect.TypeOf(obj)
	informer, exists := f.informers[informerType]
	if exists {
		return informer
	}

	resyncPeriod, exists := f.customResync[informerType]
	if !exists {
		resyncPeriod = f.defaultResync
	}
  // 根据用户传入的informer生成方法生成informer
	informer = newFunc(f.client, resyncPeriod)
  // 将informer添加进SharedIndexInformer
	f.informers[informerType] = informer

	return informer
}
```

shareIndexInformer对应的newFunc的实现，client-go中已经为所有内置对象都提供了NewInformerFunc，例如podinformer。

```go
// podinformer 接口，包含Informer/Lister两个方法
type PodInformer interface {
	Informer() cache.SharedIndexInformer
	Lister() v1.PodLister
}

// podInformer结构体，包含SharedInformerFactory接口和ns
type podInformer struct {
	factory          internalinterfaces.SharedInformerFactory
	tweakListOptions internalinterfaces.TweakListOptionsFunc
	namespace        string
}

func NewPodInformer(client kubernetes.Interface, namespace string, resyncPeriod time.Duration, indexers cache.Indexers) cache.SharedIndexInformer {
	return NewFilteredPodInformer(client, namespace, resyncPeriod, indexers, nil)
}

// 真正生成SharedIndexInformer，在其中添加了eventHandler
func NewFilteredPodInformer(client kubernetes.Interface, namespace string, resyncPeriod time.Duration, indexers cache.Indexers, tweakListOptions internalinterfaces.TweakListOptionsFunc) cache.SharedIndexInformer {
	return cache.NewSharedIndexInformer(
		&cache.ListWatch{
			ListFunc: func(options metav1.ListOptions) (runtime.Object, error) {
				if tweakListOptions != nil {
					tweakListOptions(&options)
				}
				return client.CoreV1().Pods(namespace).List(context.TODO(), options)
			},
			WatchFunc: func(options metav1.ListOptions) (watch.Interface, error) {
				if tweakListOptions != nil {
					tweakListOptions(&options)
				}
				return client.CoreV1().Pods(namespace).Watch(context.TODO(), options)
			},
		},
		&corev1.Pod{},
		resyncPeriod,
		indexers,
	)
}

func (f *podInformer) defaultInformer(client kubernetes.Interface, resyncPeriod time.Duration) cache.SharedIndexInformer {
	return NewFilteredPodInformer(client, f.namespace, resyncPeriod, cache.Indexers{cache.NamespaceIndex: cache.MetaNamespaceIndexFunc}, f.tweakListOptions)
}

func (f *podInformer) Informer() cache.SharedIndexInformer {
	return f.factory.InformerFor(&corev1.Pod{}, f.defaultInformer)
}

func (f *podInformer) Lister() v1.PodLister {
	return v1.NewPodLister(f.Informer().GetIndexer())
}
```

调用sharedInformerFactory.Core().V1().Pods()，就掉用了podInformer构造函数，生成Podinformer对象。

```
// Pods returns a PodInformer.
func (v *version) Pods() PodInformer {
   return &podInformer{factory: v.factory, namespace: v.namespace, tweakListOptions: v.tweakListOptions}
}
```

cache 包暴露的 Informer 创建方法有以下 5 个：

- New
- NewInformer
- NewIndexerInformer
- NewSharedInformer
- NewSharedIndexInformer

它们有着不同程度的抽象和封装，NewSharedIndexInformer 是其中抽象程度最低，封装程度最高的一个，但即使是 NewSharedIndexInformer，也没有封装具体的资源类型，需要接收 ListerWatcher 和 Indexers 作为参数：

```go
func NewSharedIndexInformer(
	lw ListerWatcher,
	exampleObject runtime.Object,
	defaultEventHandlerResyncPeriod time.Duration,
	indexers Indexers,
) SharedIndexInformer {}
```

### 3.5 sharedProcessor

sharedProcessor 有一个processorListener 的集合，可以将一个通知对象分发给它的监听器。有两种分发操作。同步分发到侦听器的子集，(a) 在偶尔调用 shouldResync 时重新计算，(b) 每个侦听器最初都被放入。非同步分发到每个侦听器。在SharedInformer中有非常重要的一个属性sharedProcessor，其包含了processorListener，来通知从sharedProcessor到ResourceEventHandler，其使用两个无缓冲chanel，两个goroutines和一个无界环形缓冲区，一个 goroutine 运行 `pop()`，它使用环形缓冲区中的存储将通知从 `addCh` 泵到 `nextCh`，而 `nextCh` 没有跟上。另一个 goroutine 运行 `run()`，它接收来自 `nextCh` 的通知并同步调用适当的处理程序方法。

```
type sharedProcessor struct {
  // 所有processor是否都已经启动
   listenersStarted bool
   listenersLock    sync.RWMutex
   // 通用处理列表
   listeners        []*processorListener
   // 定时同步的处理列表
   syncingListeners []*processorListener
   clock            clock.Clock
   wg               wait.Group
}

// processorListener 通知
type processorListener struct {
	nextCh chan interface{}
	addCh  chan interface{}  // 添加事件的通道

	handler ResourceEventHandler

	// pendingNotifications 是一个无边界的环形缓冲区，用于保存所有尚未分发的通知
	pendingNotifications buffer.RingGrowing

	requestedResyncPeriod time.Duration
	
	resyncPeriod time.Duration

	nextResync time.Time

	resyncLock sync.Mutex
}
```

* add：事件添加通过addCh通道接受，notification就是事件，也就是从DeltaFIFO弹出的Deltas，addCh是一个无缓冲通道，所以可以将其看作一个事件分发器，从DeltaFIFO弹出的对象需要逐一送到多个处理器，如果处理器没有及时处理addCh则会被阻塞。

```go
func (p *processorListener) add(notification interface{}) {
	p.addCh <- notification
}
```

* pop：利用golang select来同时处理多个channel，直到至少有一个case表达式满足条件为止。

```go
func (p *processorListener) pop() {
	defer utilruntime.HandleCrash()
  // 通知run停止函数运行
	defer close(p.nextCh) // Tell .run() to stop

	var nextCh chan<- interface{}
	var notification interface{}
	for {
		select {
    // 由于nextCh还没有进行初始化，在此会zuse
		case nextCh <- notification:
			// 通知分发，
			var ok bool
			notification, ok = p.pendingNotifications.ReadOne()
			if !ok { // 没有事件被Pop，则设置nextCh为nil
				nextCh = nil // Disable 这个 select的 case
			}
    // 从p.addCh 中读取一个事件，消费addCh
		case notificationToAdd, ok := <-p.addCh:
			if !ok { // 如果关闭则直接退出
				return
			}
      // pendingNotifications 为空，则说明没有notification 去pop
			if notification == nil { 
				// 把刚刚获取的事件通过 p.nextCh 发送给处理器
				notification = notificationToAdd
				nextCh = p.nextCh
			} else { 
        // 上一个事件还没发送完成（已经有一个通知等待发送），就先放到缓冲通道中
				p.pendingNotifications.WriteOne(notificationToAdd)
			}
		}
	}
}
```

run() 和 pop() 是 processorListener 的两个最核心的函数，processorListener 就是实现了事件的缓冲和处理，在没有事件的时候可以阻塞处理器，当事件较多是可以把事件缓冲起来，实现了事件分发器与处理器的异步处理。processorListener 的 run() 和 pop() 函数其实都是通过 sharedProcessor 启动的协程来调用的，所以下面我们再来对 sharedProcessor 进行分析了。首先看下如何添加一个 processorListener：

```go

// 添加处理器
func (p *sharedProcessor) addListener(listener *processorListener) {
	p.listenersLock.Lock()  // 加锁
	defer p.listenersLock.Unlock()
  // 调用 addListenerLocked 函数
	p.addListenerLocked(listener)
  // 如果事件处理列表中的处理器已经启动了，则手动启动下面的两个协程
  // 相当于启动后了
	if p.listenersStarted {  
    // 通过 wait.Group 启动两个协程，就是上面我们提到的 run 和 pop 函数
		p.wg.Start(listener.run)
		p.wg.Start(listener.pop)
	}
}

// 将处理器添加到处理器的列表中
func (p *sharedProcessor) addListenerLocked(listener *processorListener) {
  // 添加到通用处理器列表中
	p.listeners = append(p.listeners, listener)  
  // 添加到需要定时同步的处理器列表中
	p.syncingListeners = append(p.syncingListeners, listener)
}
```

这里添加处理器的函数 addListener 其实在 sharedIndexInformer 中的 AddEventHandler 函数中就会调用这个函数来添加处理器。然后就是事件分发的函数实现：

```go
func (p *sharedProcessor) distribute(obj interface{}, sync bool) {
	p.listenersLock.RLock()
	defer p.listenersLock.RUnlock()
  // sync 表示 obj 对象是否是同步事件对象
  // 将对象分发给每一个事件处理器列表中的处理器
	if sync {
		for _, listener := range p.syncingListeners {
			listener.add(obj)
		}
	} else {
		for _, listener := range p.listeners {
			listener.add(obj)
		}
	}
}
```

然后就是将 sharedProcessor 运行起来：

```go
func (p *sharedProcessor) run(stopCh <-chan struct{}) {
	func() {
		p.listenersLock.RLock()
		defer p.listenersLock.RUnlock()
    // 遍历所有的处理器，为处理器启动两个后台协程：run 和 pop 操作
    // 后续添加的处理器就是在上面的 addListener 中去启动的
		for _, listener := range p.listeners {
			p.wg.Start(listener.run)
			p.wg.Start(listener.pop)
		}
    // 标记为所有处理器都已启动
		p.listenersStarted = true
	}()
  // 等待退出信号
	<-stopCh
  // 接收到退出信号后，关闭所有的处理器
	p.listenersLock.RLock()
	defer p.listenersLock.RUnlock()
  // 遍历所有处理器
	for _, listener := range p.listeners {
    // 关闭 addCh，pop 会停止，pop 会通知 run 停止
		close(listener.addCh) 
	}
  // 等待所有协程退出，就是上面所有处理器中启动的两个协程 pop 与 run
	p.wg.Wait() 
}
```

到这里 sharedProcessor 就完成了对 ResourceEventHandler 的封装处理，当然最终 sharedProcessor 还是在 SharedInformer 中去调用的。

sharedIndexInformer的HandleDeltas处理从deltaFIFO pod出来的增量时，先尝试更新到本地缓存cache，更新成功的话会调用processor.distribute方法向processor中的listener添加notification，listener启动之后会不断获取notification回调用户的EventHandler方法。

在此就不在过度展开，更详细的内容还需要去看k8s源码。

## 四 小试牛刀

### 4.1 添加event事件

添加有AddEventHandler，首次list关注的资源后，后期通过watch判断资源状态执行相应操作，通过indexer来缓存中获取对象。

```go
func Muste(e error) {
	if e != nil {
		panic(e)
	}
}
func InitClientSet() *kubernetes.Clientset {
	kubeconfig := filepath.Join(homedir.HomeDir(), ".kube", "config")
	restConfig, err := clientcmd.BuildConfigFromFlags("", kubeconfig)
	Muste(err)
	return kubernetes.NewForConfigOrDie(restConfig)
}

func main() {
	clientSet := InitClientSet()
  // 初始化sharedInformerFactory
	sharedInformerFactory := informers.NewSharedInformerFactory(clientSet, 0)
  // 生成podInformer
	podInformer := sharedInformerFactory.Core().V1().Pods()
  // 生成具体informer/indexer
	informer := podInformer.Informer()
	indexer := podInformer.Lister()
  // 添加Event事件处理函数
	informer.AddEventHandler(cache.ResourceEventHandlerFuncs{
		func(obj interface{}) {
			podObj := obj.(*corev1.Pod)

			fmt.Printf("AddFunc: %s\n", podObj.GetName())

		},
		func(oldObj, newObj interface{}) {
			oldPodObj := oldObj.(*corev1.Pod)
			newPodObj := newObj.(*corev1.Pod)

			fmt.Printf("old: %s\n", oldPodObj.GetName())
			fmt.Printf("new: %s\n", newPodObj.GetName())

		},
		func(obj interface{}) {
			podObj := obj.(*corev1.Pod)
			fmt.Printf("deleteFunc: %s\n", podObj.GetName())
		},
	})

	stopCh := make(chan struct{})
	defer close(stopCh)
  // 启动informer
	sharedInformerFactory.Start(stopCh)
  // 等待同步完成
  sharedInformerFactory.WaitForCacheSync(stopCh)

 // 利用indexer获取资源
	pods, err := indexer.List(labels.Everything())
	Muste(err)
	for _, items := range pods {
		fmt.Printf("namespace: %s, name:%s\n", items.GetNamespace(), items.GetName())
	}
	<-stopCh
}

```

### 4.2 通过Gin实现K8s资源获取

#### 4.2.1 单个资源对象获取

通过生成deployinformer,通过gin框架暴露访问入口，通过informer中的indexer来获取deployment资源。

```go
func main() {

	clientSet, err := initClientSet()
	if err != nil {
		log.Fatal(err)
	}
	sharedInformerFactory := informers.NewSharedInformerFactory(clientSet, 0)
  // 获取deployinformeer
	deployInformer := sharedInformerFactory.Apps().V1().Deployments()
	_ = deployInformer.Informer()
	lister := deployInformer.Lister()
	stopCh := make(chan struct{})
	sharedInformerFactory.Start(stopCh)
	sharedInformerFactory.WaitForCacheSync(stopCh)

	// http://localhost:8888/deploy?labels[k8s-app]=kube-dns
	r := gin.Default()
	r.GET("/deploy", func(c *gin.Context) {
		var set map[string]string
    // 通过labels过滤deploy
		if content, ok := c.GetQueryMap("labels"); ok {
			set = content
		}
		selectQuery := labels.SelectorFromSet(set)

		podList, err := lister.List(selectQuery)
		if err != nil {
			c.JSON(400, gin.H{
				"msg": err.Error(),
			})
			return
		}
		c.JSON(200, gin.H{
			"data": podList,
		})

	})

	r.Run(":8888")

	//<-stopCh

}

```

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220129173201.png)

#### 4.2.2 根据路径参数获取不同资源

之前的示例可以获取制定资源，下面示例将gvr通过gin路径参数传递进来，进而实现一个API可以获取K8s所有资源。

```go
func main() {
	clientSet := InitClientSet()
	sharedInformerFactory := informers.NewSharedInformerFactory(clientSet, 0)
	stopCh := make(chan struct{})
	defer close(stopCh)

	engin := gin.Default()
	// 构造一个通用的根据不同group/version/kind 获取对象类型
	//http://localhost:8888/core/v1/pods
	engin.GET("/:group/:version/:resource", func(c *gin.Context) {
		g, v, r := c.Param("group"), c.Param("version"), c.Param("resource")
    // 如果是core组则为空字符串
		if g == "core" {
			g = ""
		}
		gvr := schema.GroupVersionResource{g, v, r}
		genericInformer, _ := sharedInformerFactory.ForResource(gvr)
		_ = genericInformer.Informer()
		sharedInformerFactory.Start(stopCh)
		sharedInformerFactory.WaitForCacheSync(stopCh)
		items, err := genericInformer.Lister().List(labels.Everything())
		if err != nil {
			c.JSON(500, gin.H{
				"msg": err,
			})
		}
		c.JSON(200, gin.H{
			"msg": items,
		})
	})

	engin.Run(":8888")
}
```

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220129173627.png)

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220129173704.png)

## 总结

在以上的内容中，我们分析了 informers 包的源代码，并简单测试了informer的原理，更详细的内容还是需要自己去看源码领会，后续看完indexer后，我们自己手动去编写一个Controller来加深对Informer整个流程的理解。

## 参考链接

* https://jimmysong.io/kubernetes-handbook/develop/client-go-informer-sourcecode-analyse.html

  

