# 8.Client-go源码分析之Informer

## 一 前言

Informer 是 Client-go 中的一个核心工具包，其实就是一个带有本地缓存和索引机制的、可以注册 EventHandler 的 client，本地缓存被称为 Store，索引被称为 Index。Informer 中主要包含 Controller、Reflector、DeltaFIFO、LocalStore、Lister 和 Processor 六个组件，这篇文章主要从Controller来讲，单独拿Controller来将，注意Informer中的Controller和我们K8s内部传统的controller不是一个概念。

[源图地址](https://www.processon.com/view/link/5f55f3f3e401fd60bde48d31)，有兴趣可以打开原图学习。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220129144007.png)

## 二 处理流程

我们今天展开的controller struct，不断从DeltaFIFO中弹出对象，并和内存存储对象Indexer同步，随后调用用户注册的回调函数ResourceEventHandlers。

- Informer 首先会 list/watch apiserver，Informer 所使用的 Reflector 包负责与 apiserver 建立连接，Reflector 使用 ListAndWatch 的方法，会先从 apiserver 中 list 该资源的所有实例，list 会拿到该对象最新的 resourceVersion，然后使用 watch 方法监听该 resourceVersion 之后的所有变化，若中途出现异常，reflector 则会从断开的 resourceVersion 处重现尝试监听所有变化，一旦该对象的实例有创建、删除、更新动作，Reflector 都会收到”事件通知”，这时，该事件及它对应的 API 对象这个组合，被称为增量（Delta），它会被放进 DeltaFIFO 中。
- Informer 会不断地从这个 DeltaFIFO 中读取增量，每拿出一个对象，Informer 就会判断这个增量的时间类型，然后创建或更新本地的缓存，也就是 store。
- 如果事件类型是 Added（添加对象），那么 Informer 会通过 Indexer 的库把这个增量里的 API 对象保存到本地的缓存中，并为它创建索引，若为删除操作，则在本地缓存中删除该对象。
- DeltaFIFO 再 pop 这个事件到 controller 中，controller 会调用事先注册的 ResourceEventHandler 回调函数进行处理。
- 在 ResourceEventHandler 回调函数中，其实只是做了一些很简单的过滤，然后将关心变更的 Object 放到 workqueue 里面。
- Controller 从 workqueue 里面取出 Object，启动一个 worker 来执行自己的业务逻辑，业务逻辑通常是计算目前集群的状态和用户希望达到的状态有多大的区别，然后让 apiserver 将状态演化到用户希望达到的状态，比如为 deployment 创建新的 pods，或者是扩容/缩容 deployment。
- 在worker中就可以使用 lister 来获取 resource，而不用频繁的访问 apiserver，因为 apiserver 中 resource 的变更都会反映到本地的 cache 中。

## 三 源码分析

我们从源码中看到controller包含，config配置，reflecotr，及锁，其中config很重要，包含了controller的设置。

```go
type controller struct {
	config         Config
	reflector      *Reflector
	reflectorMutex sync.RWMutex
	clock          clock.Clock
}

// Config contains all the settings for one of these low-level controllers.
type Config struct {
	// DeltaFIFO
	Queue

	// ListerWatcher
	ListerWatcher

	// 处理pop出来的元素方法
	Process ProcessFunc

	// 具体资源对象
	ObjectType runtime.Object

	// 全量resync周期间隔
	FullResyncPeriod time.Duration

	// ShouldResync is periodically used by the reflector to determine
	// whether to Resync the Queue. If ShouldResync is `nil` or
	// returns true, it means the reflector should proceed with the
	// resync.
	ShouldResync ShouldResyncFunc

	// If true, when Process() returns an error, re-enqueue the object.
	// TODO: add interface to let you inject a delay/backoff or drop
	//       the object completely if desired. Pass the object in
	//       question to this interface as a parameter.  This is probably moot
	//       now that this functionality appears at a higher level.
	RetryOnError bool

	// Called whenever the ListAndWatch drops the connection with an error.
	WatchErrorHandler WatchErrorHandler

	// WatchListPageSize is the requested chunk size of initial and relist watch lists.
	WatchListPageSize int64
}

type Controller interface {
	// 运行controller
	Run(stopCh <-chan struct{})
  // 判断是否同步完成
	HasSynced() bool
	// LastSyncResourceVersion delegates to the Reflector when there
	// is one, otherwise returns the empty string
  // 获取最后一次同步的resourceversion
	LastSyncResourceVersion() string
}
```

### 3.1 newInformer创建

我们可以在源码中先单独看看controller的实现，创建一个Informer需要传入ListerWatcher，资源对象，及ResourceEventHandler，以及Store。

```go
// client-go/tools/cache/controller.go
func newInformer(
	lw ListerWatcher,
	objType runtime.Object,
	resyncPeriod time.Duration,
	h ResourceEventHandler,
	clientState Store,
) Controller {
	// 存储资源DeltaFIFO
	fifo := NewDeltaFIFOWithOptions(DeltaFIFOOptions{
		KnownObjects:          clientState,
		EmitDeltaTypeReplaced: true,
	})

	cfg := &Config{
		Queue:            fifo,
		ListerWatcher:    lw,
		ObjectType:       objType,
		FullResyncPeriod: resyncPeriod,
		RetryOnError:     false,
    
    // 对你pop出去的资源进行操作
		Process: func(obj interface{}) error {
			// from oldest to newest
			for _, d := range obj.(Deltas) {
				switch d.Type {
				case Sync, Replaced, Added, Updated:
					if old, exists, err := clientState.Get(d.Object); err == nil && exists {
            // store 中更新资源
						if err := clientState.Update(d.Object); err != nil {
							return err
						}
						h.OnUpdate(old, d.Object)
					} else {
						if err := clientState.Add(d.Object); err != nil {
							return err
						}
						h.OnAdd(d.Object)
					}
				case Deleted:
					if err := clientState.Delete(d.Object); err != nil {
						return err
					}
					h.OnDelete(d.Object)
				}
			}
			return nil
		},
	}
	return New(cfg)
}

```

### 3.2 Run

controller的run函数实际上创建reflector之后并reflector中的run。

```go
func (c *controller) Run(stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()
	go func() {
		<-stopCh
		c.config.Queue.Close()
	}()
	r := NewReflector(
		c.config.ListerWatcher,
		c.config.ObjectType,
		c.config.Queue,
		c.config.FullResyncPeriod,
	)
	r.ShouldResync = c.config.ShouldResync
	r.WatchListPageSize = c.config.WatchListPageSize
	r.clock = c.clock
	if c.config.WatchErrorHandler != nil {
		r.watchErrorHandler = c.config.WatchErrorHandler
	}

	c.reflectorMutex.Lock()
	c.reflector = r
	c.reflectorMutex.Unlock()

	var wg wait.Group
  // 运行reflector的run函数
	wg.StartWithChannel(stopCh, r.Run)
  // 从队列中pop对象资源并进行处理
	wait.Until(c.processLoop, time.Second, stopCh)
	wg.Wait()
}
// 从DeltaFIFO中弹出元素，通过用户穿入的PopProcessFunc进行处理
func (c *controller) processLoop() {
	for {
		obj, err := c.config.Queue.Pop(PopProcessFunc(c.config.Process))
		if err != nil {
			if err == ErrFIFOClosed {
				return
			}
			if c.config.RetryOnError {
				// This is the safe way to re-enqueue.
				c.config.Queue.AddIfNotPresent(obj)
			}
		}
	}
}
```

### 3.3 HasSynced

其实是调用FIFO中的，判断被插入的资源被pop处理完成，返回true

```go
// Returns true once this controller has completed an initial resource listing
func (c *controller) HasSynced() bool {
	return c.config.Queue.HasSynced()
}
```

## 四 小试牛刀

```go
func Must(e error) {
	if e != nil {
		panic(e)
	}
}
// 生成clientset
func InitClientSet() *kubernetes.Clientset {
	configFlags := genericclioptions.NewConfigFlags(false)
	restConfig, err := configFlags.ToRESTConfig()
	Must(err)
	return kubernetes.NewForConfigOrDie(restConfig)
}

func main() {
	clientSet := InitClientSet()
  // 构造pod listwatcher
	podlw := cache.NewListWatchFromClient(clientSet.CoreV1().RESTClient(), "pods", "default", fields.Everything())
  // 生成PodInformer，添加事件，在此事件仅仅是打印出pod对象名称
	_, controller := cache.NewInformer(podlw, &corev1.Pod{}, 0, cache.ResourceEventHandlerFuncs{
		AddFunc: func(obj interface{}) {
			podObj, _ := obj.(*corev1.Pod)
			fmt.Println("addfunc", podObj.GetName())
		},
		UpdateFunc: func(oldObj, newObj interface{}) {
			oldpodObj, _ := oldObj.(*corev1.Pod)
			newpodObj, _ := oldObj.(*corev1.Pod)
			fmt.Println("addfunc", newpodObj.GetName())
			fmt.Println("addfunc", oldpodObj.GetName())
		},
		DeleteFunc: func(obj interface{}) {
			podObj, _ := obj.(*corev1.Pod)
			fmt.Println("deletefunc", podObj.GetName())
		},
	})

	stopCh := make(chan struct{})
	defer close(stopCh)
	go controller.Run(stopCh)

	cache.WaitForCacheSync(stopCh, controller.HasSynced)

	<-stopCh
}
```

Informer帮我们做了针对不同类型的对象操作，我们只需要关系Handler，在同步周期中如果设置非0，则定时同步中会产生SYNC事件，触发updateFunc操作。





