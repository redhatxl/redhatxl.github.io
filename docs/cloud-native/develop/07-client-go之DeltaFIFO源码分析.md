# 7.Client-go源码分析之DeltaFIFO

## 一 前言

在之前的 `Reflector` 学习中，可以看到在 `ListAndWatch` 方法中，对资源的全量 `List` 最后调用的其实是 `Reflector` 传入的 `Store` 中的 `Replace`  方法，在定时同步中，调用的是`Store`的`Resync`方法，那么都是如何实现的呢，本文即将从源码展开分析。

`DeltaFIFO`是K8s中用来存储处理数据的`Queue`，相较于传统的`FIFO`,它不仅仅存储了数据保证了先进先出，而且存储有K8s 资源对象的类型。其是连接`Reflector`(生产者)和`indexer`(消费者)的重要通道。

## 二 源码

在这里我们着重看几个比较重要的方法及函数，理解DeltaFIFO的重要几个属性及方法，具体更细的Queue，FIFO等方法需要去查看更详细的源码。

### 2.1 DeltaFIFO结构

```go
// Delta 结构体
type Delta struct {
	Type   DeltaType
	Object interface{}
}

type DeltaType string

// Change type definition
const (
	Added   DeltaType = "Added"
	Updated DeltaType = "Updated"
	Deleted DeltaType = "Deleted"
 // 当遇到 watch 错误，不得不进行重新list时，就会触发 Replaced。
  // 我们不知道被替换的对象是否发生了变化。
	Replaced DeltaType = "Replaced"
	// Sync 是针对周期性重新同步期间的合成事件
	Sync DeltaType = "Sync"
)

type DeltaFIFO struct {
	// lock/cond 保护对“项目”和“队列”的访问,确保线程安全
	lock sync.RWMutex
  // cond实现一个条件变量，一个集合点 ，用于等待或宣布发生的goroutine事件。
  // 每个Cond都有一个锁L（通常是* Mutex或* RWMutex），
  // 更改条件时必须保留 调用Wait方法时。第一次使用后不得复制条件。
	cond sync.Cond

	// `items` maps a key to a Deltas.
	// items为同一类对象的变化delta变化列表
	items map[string]Deltas

	// `queue` maintains FIFO order of keys for consumption in Pop().
	// There are no duplicates in `queue`.
	// 为了保证顺序
	queue []string

  // 如果调用Replace()第一次填充完成，则populated设置为true
	populated bool
	// 第一次调用Replace插入的项目数
	initialPopulationCount int

	// 计算item的key
	keyFunc KeyFunc

	// 是后面的indexer
	knownObjects KeyListerGetter

	closed bool

	// emitDeltaTypeReplaced is whether to emit the Replaced or Sync
	// DeltaType when Replace() is called (to preserve backwards compat).
	emitDeltaTypeReplaced bool
}
```

从源码中我们可以看出Delta有5种类型，前面增删改顾名思义都是在watch中监控对象的变化，对于Replaced和Sync用于首次和异常情况确保indexer中的数据和etcd中的保持一致。

我们可以看出DeltaFIFO中重要的几个属性。

* queue：存储资源对象的key。

* items：存储某个对象的一类行为，顺序的保存同一个对象的一系列变化行为，key为keyFunc计算的值，value为delta的列表。
* keyFunc:用于计算items的key。

DeltaFIFO 他会保留关于资源对象obj的操作类型，队列中会存在不同操作类型的同一个资源对象，queue的key通过Keyof函数计算得到 ，items字段通过map数据结构方式存储，valuse存储的是对象的deltas数组。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220127194012.png)

例如用户创建一个Pod，那么该Delat就说一个带有Added类型的Pod，带上类型为了后续跟进不同的操作，控制器执行不同的业务逻辑。

### 2.2 queueActionLocked

我们查看DeltaFIFO的方法，例如Add/Update/Delete都调用了`queueActionLocked`方法，具体对该方法进行分析。

```
// DeltaFIFO 的Add方法
func (f *DeltaFIFO) Add(obj interface{}) error {
   f.lock.Lock()
   defer f.lock.Unlock()
   f.populated = true
   return f.queueActionLocked(Added, obj)
}
```

* queueActionLocked

大体步骤可以分为：获取对象键，将新对象添加至对象列表中，对delete类型进行去重，如果对象不在queue中，将其key存储到queue中，最终完成添加对象到items中和更新queue。

```go
// 
func (f *DeltaFIFO) queueActionLocked(actionType DeltaType, obj interface{}) error {
  //  获取对象键
	id, err := f.KeyOf(obj)
	if err != nil {
		return KeyError{obj, err}
	}
  // 临时存储旧对象列表
	oldDeltas := f.items[id]
  // 将对象添加至新对象列表中
	newDeltas := append(oldDeltas, Delta{actionType, obj})
  // 对于delete 类型进行去重，因为更新类型可能更新某个字段存在异常。
	newDeltas = dedupDeltas(newDeltas)
  // 判断queue中是否存储有对象key，如果不存在之前将其添加至queue中
	if len(newDeltas) > 0 {
		if _, exists := f.items[id]; !exists {
			f.queue = append(f.queue, id)
		}
		f.items[id] = newDeltas
    // 通知所有的消费者解除阻塞
		f.cond.Broadcast()
	} else {
		// This never happens, because dedupDeltas never returns an empty list
		// when given a non-empty list (as it is here).
		// If somehow it happens anyway, deal with it but complain.
		if oldDeltas == nil {
			klog.Errorf("Impossible dedupDeltas for id=%q: oldDeltas=%#+v, obj=%#+v; ignoring", id, oldDeltas, obj)
			return nil
		}
		klog.Errorf("Impossible dedupDeltas for id=%q: oldDeltas=%#+v, obj=%#+v; breaking invariant by storing empty Deltas", id, oldDeltas, obj)
    // 生成最终的对象items
		f.items[id] = newDeltas
		return fmt.Errorf("Impossible dedupDeltas for id=%q: oldDeltas=%#+v, obj=%#+v; broke DeltaFIFO invariant by storing empty Deltas", id, oldDeltas, obj)
	}
	return nil
}
```

* KeyOf

  在queueActionLocked中获取DeltaFIFO最新的对象键使用的是KeyOf方法，首先判读对象是否为delta切片，利用最新的delta对象进行操作，直接调用的DeltaFIFO中的keyFunc，在K8s中如果不传入，默认使用的是MetaNamespaceKeyFunc，该函数返回的是<namespace>/<name>，除非<namespace>为空，则直接使用对象名称作为key。

```go
// DeltaFIFO 的对象键计算函数
func (f *DeltaFIFO) KeyOf(obj interface{}) (string, error) {
 // 判断是否是 Delta 切片
  if d, ok := obj.(Deltas); ok {
  if len(d) == 0 {
   return "", KeyError{obj, ErrZeroLengthDeltasObject}
  }
  // 使用最新版本的对象进行计算
  obj = d.Newest().Object
 }
 if d, ok := obj.(DeletedFinalStateUnknown); ok {
  return d.Key, nil
 }
  // 具体计算还是要看初始化 DeltaFIFO 传入的 KeyFunc 函数
 return f.keyFunc(obj)
}

// Newest 返回最新的 Delta，如果没有则返回 nil。
func (d Deltas) Newest() *Delta {
 if n := len(d); n > 0 {
  return &d[n-1]
 }
 return nil
}
```

### 2.3 Replace

在之前的 `Reflector` 学习中，可以看到在 `ListAndWatch` 方法中，对资源的全量 `List` 最后调用的其实是 `Reflector` 传入的 `Store` 中的 `Replace`  方法。

Replace方法主要用于对象的全量更新上，由于 DeltaFIFO 对外输出的就是所有目标的增量变化，所以每次全量更新都要判断对象是否已经删除，因为在全量更新前可能没有收到目标删除的请求。这一点与 cache 不同，cache 的Replace() 相当于重建，因为 cache 就是对象全量的一种内存映射，所以Replace() 就等于重建。

```go
func (f *DeltaFIFO) Replace(list []interface{}, resourceVersion string) error {
	f.lock.Lock()
	defer f.lock.Unlock()
  // 构造一个set
	keys := make(sets.String, len(list))

	// 旧版本客户端List操作Sync，再此为了向后兼容
	action := Sync
	if f.emitDeltaTypeReplaced {
		action = Replaced
	}

	// 对传入的对象列切片进行遍历并添加至DeltaFIFO中。
	for _, item := range list {
    // 获取对象key
		key, err := f.KeyOf(item)
		if err != nil {
			return KeyError{item, err}
		}
    // 利用Set集合保存处理过的对象键
		keys.Insert(key)
		if err := f.queueActionLocked(action, item); err != nil {
			return fmt.Errorf("couldn't enqueue object: %v", err)
		}
	}

  // 判断是否有Indexer存储，如果没有Indexer，那么就维护自己的Queue
  // 如果老对象不在Queue中，则删除对象，否则更新为最新的对象
	if f.knownObjects == nil {
		// Do deletion detection against our own list.
		queuedDeletions := 0
    
    // 对对象执行更新
		for k, oldItem := range f.items {
			if keys.Has(k) {
				continue
			}
			// Delete pre-existing items not in the new list.
			// This could happen if watch deletion event was missed while
			// disconnected from apiserver.
			var deletedObj interface{}
			if n := oldItem.Newest(); n != nil {
				deletedObj = n.Object
			}
			queuedDeletions++
      // 因为可能队列中已经存在 Deleted 类型的元素了，避免重复，所以采用 DeletedFinalStateUnknown 
			if err := f.queueActionLocked(Deleted, DeletedFinalStateUnknown{k, deletedObj}); err != nil {
				return err
			}
		}
   // 如果populated为false，则说明第一次入的队列对象为操作完成
		if !f.populated {
			f.populated = true
			// 记录第一次设置的对象数量
			f.initialPopulationCount = keys.Len() + queuedDeletions
		}

		return nil
	}

	// 检测不在队列中的删除对象
	knownKeys := f.knownObjects.ListKeys()
	queuedDeletions := 0
	for _, k := range knownKeys {
		if keys.Has(k) {
			continue
		}
    // 根据key从indexer中进行获取
		deletedObj, exists, err := f.knownObjects.GetByKey(k)
		if err != nil {
			deletedObj = nil
			klog.Errorf("Unexpected error %v during lookup of key %v, placing DeleteFinalStateUnknown marker without object", err, k)
		} else if !exists {
			deletedObj = nil
			klog.Infof("Key %v does not exist in known objects store, placing DeleteFinalStateUnknown marker without object", k)
		}
		queuedDeletions++
    // 将对象删除的delta放入队列
		if err := f.queueActionLocked(Deleted, DeletedFinalStateUnknown{k, deletedObj}); err != nil {
			return err
		}
	}

	if !f.populated {
		f.populated = true
		f.initialPopulationCount = keys.Len() + queuedDeletions
	}

	return nil
}
```



### 2.4 Resync

在定时同步中，调用的是`Store`的`Resync`方法，Resync 重新同步，带有 Sync 类型的 Delta 对象，如果f.knownObjects也就是Indexer不存在，则不执行Resync操作。

```go
func (f *DeltaFIFO) Resync() error {
	f.lock.Lock()
	defer f.lock.Unlock()
	// 如果没有indexer，则不执行
	if f.knownObjects == nil {
		return nil
	}
  // 获取indexer的key列表
	keys := f.knownObjects.ListKeys()
	for _, k := range keys {
		if err := f.syncKeyLocked(k); err != nil {
			return err
		}
	}
	return nil
}
// 具体的resync操作
func (f *DeltaFIFO) syncKeyLocked(key string) error {
  // 获取key
	obj, exists, err := f.knownObjects.GetByKey(key)
  // 如果错误或不存在则直接返回
	if err != nil {
		klog.Errorf("Unexpected error %v during lookup of key %v, unable to queue object for sync", err, key)
		return nil
	} else if !exists {
		klog.Infof("Key %v does not exist in known objects store, unable to queue object for sync", key)
		return nil
	}

	// If we are doing Resync() and there is already an event queued for that object,
	// we ignore the Resync for it. This is to avoid the race, in which the resync
	// comes with the previous value of object (since queueing an event for the object
	// doesn't trigger changing the underlying store <knownObjects>.
  // 根据KeyOf获取到key
	id, err := f.KeyOf(obj)
	if err != nil {
		return KeyError{obj, err}
	}
  // 如果DeltaFIFO中该key有值，则暂时先不更新，等最后一个对象操作完成后执行最后状态的更新
	if len(f.items[id]) > 0 {
		return nil
	}
  // 添加对象同步的这个 Delta
	if err := f.queueActionLocked(Sync, obj); err != nil {
		return fmt.Errorf("couldn't queue object: %v", err)
	}
	return nil
}
```

### 2.5 Pop

最后要看下对DeltaFIFO中对象的消费，实际是利用Pop函数，具体对于数据流向的处理是通过PopProcessFunc实现。Pop 会等到一个元素准备好后再进行处理，如果有多个元素准备好了，则按照它们被添加或更新的顺序返回。在处理之前，元素会从队列（和存储）中移除，所以如果没有成功处理，应该用 AddIfNotPresent() 函数把它添加回来。
处理函数是在有锁的情况下调用的，所以更新其中需要和队列同步的数据结构是安全的。

```
func (f *DeltaFIFO) Pop(process PopProcessFunc) (interface{}, error) {
   f.lock.Lock()
   defer f.lock.Unlock()
   for {
      for len(f.queue) == 0 {
         // 当队列为空时，Pop() 的调用会被阻塞住，直到新的元素插入队列后
         // 当调用 Close() 时，设置 f.closed 并广播条件。
         if f.closed {
            return nil, ErrFIFOClosed
         }

         f.cond.Wait()
      }
      // 获取最先进入的元素进行处理
      id := f.queue[0]
      // 从队列删除第一元素
      f.queue = f.queue[1:]
      if f.initialPopulationCount > 0 {
         f.initialPopulationCount--
      }
      // 获取被弹出的对象
      item, ok := f.items[id]
      if !ok {
         // This should never happen
         klog.Errorf("Inconceivable! %q was in f.queue but not f.items; ignoring.", id)
         continue
      }
      // 从items中删除弹出的元素
      delete(f.items, id)
      // 利用process对item进行处理
      err := process(item)
      if e, ok := err.(ErrRequeue); ok {
         // 如果处理未成功，需要调用 addIfNotPresent 加回队列
         f.addIfNotPresent(id, item)
         err = e.Err
      }
      // Don't need to copyDeltas here, because we're transferring
      // ownership to the caller.
      return item, err
   }
}
```

## 三 小试牛刀



```go
package main

import (
	"fmt"
	corev1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/fields"
	"k8s.io/apimachinery/pkg/util/wait"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/cache"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/client-go/util/homedir"
	"path/filepath"
	"time"
)

func Must(e interface{}) {
	if e != nil {
		panic(e)
	}
}

func InitClientSet() (*kubernetes.Clientset, error) {
	kubeconfig := filepath.Join(homedir.HomeDir(), ".kube", "config")
	restConfig, err := clientcmd.BuildConfigFromFlags("", kubeconfig)
	if err != nil {
		return nil, err
	}
	return kubernetes.NewForConfig(restConfig)
}

// 生成listwatcher
func InitListerWatcher(clientSet *kubernetes.Clientset, resource, namespace string, fieldSelector fields.Selector) cache.ListerWatcher {
	restClient := clientSet.CoreV1().RESTClient()
	return cache.NewListWatchFromClient(restClient, resource, namespace, fieldSelector)
}

// 生成pods reflector
func InitPodsReflector(clientSet *kubernetes.Clientset, store cache.Store) *cache.Reflector {
	resource := "pods"
	namespace := "default"
	resyncPeriod := 0 * time.Second
	expectedType := &corev1.Pod{}
	lw := InitListerWatcher(clientSet, resource, namespace, fields.Everything())

	return cache.NewReflector(lw, expectedType, store, resyncPeriod)
}

// 生成 DeltaFIFO
func InitDeltaQueue(store cache.Store) cache.Queue {
	return cache.NewDeltaFIFOWithOptions(cache.DeltaFIFOOptions{
		// store 实现了 KeyListerGetter
		KnownObjects: store,
		// EmitDeltaTypeReplaced 表示队列消费者理解 Replaced DeltaType。
		// 在添加 `Replaced` 事件类型之前，对 Replace() 的调用的处理方式与 Sync() 相同。
		// 出于向后兼容的目的，默认情况下为 false。
		// 当为 true 时，将为传递给 Replace() 调用的项目发送“替换”事件。当为 false 时，将发送 `Sync` 事件。
		EmitDeltaTypeReplaced: true,
	})

}
func InitStore() cache.Store {
	return cache.NewStore(cache.MetaNamespaceKeyFunc)
}

func main() {
	clientSet, err := InitClientSet()
	Must(err)
	// 用于在processfunc中获取
	store := InitStore()
	// queue
	DeleteFIFOQueue := InitDeltaQueue(store)
	// 生成podReflector
	podReflector := InitPodsReflector(clientSet, DeleteFIFOQueue)

	stopCh := make(chan struct{})
	defer close(stopCh)
	go podReflector.Run(stopCh)
	// 对单个元素进行处理，元素ke为 namespace/name，value 为delta列表
  // delta对象为DeltaType和runtimeobject
	ProcessFunc := func(obj interface{}) error {
		// 最先收到的事件会被最先处理
		for _, d := range obj.(cache.Deltas) {
			switch d.Type {
			case cache.Sync, cache.Replaced, cache.Added, cache.Updated:
				if _, exists, err := store.Get(d.Object); err == nil && exists {
					if err := store.Update(d.Object); err != nil {
						return err
					}
				} else {
					if err := store.Add(d.Object); err != nil {
						return err
					}
				}
			case cache.Deleted:
				if err := store.Delete(d.Object); err != nil {
					return err
				}
			}
			pods, ok := d.Object.(*corev1.Pod)
			if !ok {
				return fmt.Errorf("not config: %T", d.Object)
			}

			fmt.Printf("Type:%s: Name:%s\n", d.Type, pods.Name)
		}
		return nil
	}

	fmt.Println("Start syncing...")

	wait.Until(func() {
		for {
			_, err := DeleteFIFOQueue.Pop(ProcessFunc)
			Must(err)
		}
	}, time.Second, stopCh)
}
```

首先创建store，在创建DeltaFIFO，之后通过初始化Reflector，将DeltaFIFO作为store穿入，Reflector运行起来后，对K8s APIserver进行ListWatch操作，将List的数据存储到DeltaFIFO中，通过自定义ProcessFunc 来对DeltaFIFO的元素进行处理。

## 四 流程总结

Reflector 通过 ListAndWatch 首先获取全量的资源对象数据，然后调用 DeltaFIFO 的 Replace() 方法全量插入队列，如果设置了定时同步，则定时更新Indexer中的内容，后续通过 Watch 操作根据资源对象的操作类型调用 DeltaFIFO 的 Add、Update、Delete 方法。对于Pop 出来的元素如何处理，就要看 Pop 的回调函数 `PopProcessFunc` 了。 

## 参考链接

* https://cloud.tencent.com/developer/article/1692474

