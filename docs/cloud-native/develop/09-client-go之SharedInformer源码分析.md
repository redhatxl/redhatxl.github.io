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

  

