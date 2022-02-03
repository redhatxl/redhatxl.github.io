# 10.Client-go源码分析之Indexer

## 一 前言

从上文我们可以看到，在SharedInformer在HandleDeltas对pop出来的数据进行处理，一部分进行维护indexer，一部分进行事件的分发，那么在Informer为什么需要Indexer呢，

## 二 Indexer功能

Indexer字面上看是索引器，就是Informer的LocalStore。可以类比数据库的索引，索引是为加快数据库查询而做的数据映射，索引是对存储的扩展和包装，使得按照某些条件查询速度会非常快。

## 三 源码分析

Indexer就是一种接口，而在Informer中，实现Indexer接口的struct就是`threadSafeStore`。我们从两个方面——成员及方法来认识这个struct。

### 3.1 Indexer接口

```go
type Indexer interface {
  // 继承Store接口 
	Store

  // 检索 匹配命名索引函数的对象列表
  // indexName索引函数名，obj是对象
  // 计算obj在indexName索引函数中的所有索引键，通过索引键把所有的对象取出来
	Index(indexName string, obj interface{}) ([]interface{}, error)

  // 获取 indexName 下的 索引值为 indexedValue 集合的所有 key
	IndexKeys(indexName, indexedValue string) ([]string, error)
  
  // 索引函数indexName  中索引键为indexKey 的对象
	ListIndexFuncValues(indexName string) []string

	// 返回值不是对象键，而是所有对象
	ByIndex(indexName, indexedValue string) ([]interface{}, error)
	// GetIndexer return the indexers
	GetIndexers() Indexers

  // 在存储中添加更多的indexers，增加更多的索引函数
	AddIndexers(newIndexers Indexers) error
}
```

indexer在 store存储的基础之上，拓展了对对象查找的索引能力。说白了，其实就是通过map结构实现不通维度的资源查找。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220202133603.png)

```go
// 映射索引键和对象键
type Index map[string]sets.String

// 索引器名称于索引函数映射，存储索引分类
type Indexers map[string]IndexFunc

// 索引器名称于index的索引映射
type Indices map[string]Index
```

### 3.2 cache

Indexer 接口具体实现类型为cache，其中包含了线程安全的store即ThreadSafeStore和计算key值的函数KeyFunc，其中继承了ThreadSafeStore的相关方法，我们可以看到cache中实现ThreadSafeStore接口方法，只要是调用的包含的 ThreadSafeStore 操作接口，所以我们着重看下ThreadSafeStore接口threadSafeMap方法的具体的实现。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220201224730.png)

```go
// tools/cache/store.go

// cache 是indexer的实现，其中
type cache struct {
	// 线程安全的索引
	cacheStorage ThreadSafeStore
	// keyFunc is used to make the key for objects stored in and retrieved from items, and
	// keyFunc 计算对象键
	keyFunc KeyFunc
}

type KeyFunc func(obj interface{}) (string, error)

// ThreadSafeStore也是一接口，其中包含了对store的相关增删改查，及获取key相关方法
type ThreadSafeStore interface {
	Add(key string, obj interface{})
	Update(key string, obj interface{})
	Delete(key string)
	Get(key string) (item interface{}, exists bool)
	List() []interface{}
	ListKeys() []string
	Replace(map[string]interface{}, string)
	Index(indexName string, obj interface{}) ([]interface{}, error)
	IndexKeys(indexName, indexKey string) ([]string, error)
	ListIndexFuncValues(name string) []string
	ByIndex(indexName, indexKey string) ([]interface{}, error)
	GetIndexers() Indexers

	// AddIndexers adds more indexers to this store.  If you call this after you already have data
	// in the store, the results are undefined.
	AddIndexers(newIndexers Indexers) error
	// Resync is a no-op and is deprecated
	Resync() error
}

// Add 插入一个元素到 cache 中
func (c *cache) Add(obj interface{}) error {
	key, err := c.keyFunc(obj)  // 生成对象键
	if err != nil {
		return KeyError{obj, err}
	}
  // 将对象添加到底层的 ThreadSafeStore 中
	c.cacheStorage.Add(key, obj)
	return nil
}

// 更新cache中的对象
func (c *cache) Update(obj interface{}) error {
	key, err := c.keyFunc(obj)
	if err != nil {
		return KeyError{obj, err}
	}
	c.cacheStorage.Update(key, obj)
	return nil
}
```

### 3.3 threadSafeMap

threadSafeMap是具体ThreadSafeStore接口的实现,其内部items为最底层存储具体的资源对象，indexers为计算索引键的方法，indeces为存储通过indexers中索引键计算出不同类型的索引键。

```
// threadSafeMap 实现了 ThreadSafeStore
type threadSafeMap struct {
   lock  sync.RWMutex
   // 具体存储底层元素的内容
   items map[string]interface{}

   // 索引器名称于索引键函数映射
   indexers Indexers
   // type Indices map[string]Index
   indices Indices
}
```

threadSafeMap 的Add方法，通过索引key计算对象将其添加到items中，可以看到，先去判断获取老的对象，之后将其更新到items，然后调用updateIndices方法，实现更新indices。

```go
func (c *threadSafeMap) Add(key string, obj interface{}) {
	c.lock.Lock()
	defer c.lock.Unlock()
	oldObject := c.items[key]
	c.items[key] = obj
	c.updateIndices(oldObject, obj, key)
}

// updateIndices modifies the objects location in the managed indexes, if this is an update, you must provide an oldObj
// updateIndices must be called from a function that already has a lock on the cache
func (c *threadSafeMap) updateIndices(oldObj interface{}, newObj interface{}, key string) {
	// 如果有旧对象，先从索引中删除该对象
	if oldObj != nil {
		c.deleteFromIndices(oldObj, key)
	}
  // 开始循环索引器，利用索引器计算对象的索引键
	for name, indexFunc := range c.indexers {
		indexValues, err := indexFunc(newObj)
		if err != nil {
			panic(fmt.Errorf("unable to calculate an index entry for key %q on index %q: %v", key, name, err))
		}
    // 获取当前索引器的索引
		index := c.indices[name]
    // 初始化一个索引
		if index == nil {
			index = Index{}
			c.indices[name] = index
		}

		for _, indexValue := range indexValues {
			set := index[indexValue]
			if set == nil {
				set = sets.String{}
				index[indexValue] = set
			}
			set.Insert(key)
		}
	}
}

//  删除对象索引
func (c *threadSafeMap) deleteFromIndices(obj interface{}, key string) {
  // 循环所有的索引器
	for name, indexFunc := range c.indexers {
    // 获取删除对象的索引键列表
		indexValues, err := indexFunc(obj)
		if err != nil {
			panic(fmt.Errorf("unable to calculate an index entry for key %q on index %q: %v", key, name, err))
		}
    // 获取当前索引器的索引
		index := c.indices[name]
		if index == nil {
			continue
		}
		for _, indexValue := range indexValues {
      // 获取索引键对应的对象键列表
			set := index[indexValue]
			if set != nil {
        // 从对象键列表中删除当前要删除的对象键
				set.Delete(key)

				// 如果当集合为空的时候不删除set，会导致内存跑满
        // `kubernetes/kubernetes/issues/84959`.
        if len(set) == 0 {
					delete(index, indexValue)
				}
			}
		}
	}
}
```

看了threadSafeMap的add方法，其他的update/delete也类似，接下来看下它的索引方法。

```go
// ByIndex 根据制定的indexName和indexedValue，返回给定索引中的索引值列表
func (c *threadSafeMap) ByIndex(indexName, indexedValue string) ([]interface{}, error) {
	c.lock.RLock()
	defer c.lock.RUnlock()
  // 获取索引键计算函数
	indexFunc := c.indexers[indexName]
	if indexFunc == nil {
		return nil, fmt.Errorf("Index with name %s does not exist", indexName)
	}
  // 获取indexName的所有索引
	index := c.indices[indexName]
  // 获取指定索引键的所有对象键
	set := index[indexedValue]
	list := make([]interface{}, 0, set.Len())
  // 遍历对象键获取对象
	for key := range set {
		list = append(list, c.items[key])
	}

	return list, nil
}
```

除了ByIndex方法外还有Index/IndexKeys等其他，更详细的内容，可以自行查看源码。这里我们就将 ThreadSafeMap 的实现进行了分析说明。就是对map 中对象数据进行存储，之后就是就是维护索引，方便根据索引来查找到对应的对象。

## 四 小试牛刀

为了更直观的理解index，进行实际编码测试。

### 4.1 自定义数据测试

```go
const (
	namespaceIndex = "namespaceIndex"
	nodeNameIndex  = "nodeNameIndex"
)

func namespaceFunc(obj interface{}) ([]string, error) {
	o, err := meta.Accessor(obj)
	if err != nil {
		return []string{""}, err
	}
	return []string{o.GetNamespace()}, nil
}

func nodeNameFunc(obj interface{}) ([]string, error) {
	o, ok := obj.(*corev1.Pod)
	if !ok {
		return []string{""}, nil
	}
	return []string{o.Spec.NodeName}, nil
}

func main() {

	indexer := cache.NewIndexer(cache.MetaNamespaceKeyFunc, cache.Indexers{
		namespaceIndex: namespaceFunc,
		nodeNameIndex:  nodeNameFunc,
	})

	podList := []corev1.Pod{
		corev1.Pod{
			ObjectMeta: metav1.ObjectMeta{
				Name:      "pod1",
				Namespace: "default",
			},
			Spec: corev1.PodSpec{
				NodeName: "node1",
			},
		},
		corev1.Pod{
			ObjectMeta: metav1.ObjectMeta{
				Name:      "pod2",
				Namespace: "default",
			},
			Spec: corev1.PodSpec{
				NodeName: "node2",
			},
		},
		corev1.Pod{
			ObjectMeta: metav1.ObjectMeta{
				Name:      "pod3",
				Namespace: "podns",
			},
			Spec: corev1.PodSpec{
				NodeName: "node2",
			},
		},
	}

	for idx, item := range podList {

		err := indexer.Add(&podList[idx])
		if err != nil {
			panic(err)
		}
		fmt.Printf("add %d,%s to indexer\n", idx, item.GetName())
	}

	// 开始利用不同索引键对indexer中的对象进行索引获取

	if items, err := indexer.ByIndex(namespaceIndex, "default"); err != nil {
		fmt.Println("get index err:%s", err)
	} else {
		for _, podobj := range items {
			if pod, ok := podobj.(*corev1.Pod); ok {
				fmt.Printf("use indexkey:%s,value:%s, podName:%s,namespace:%s,nodeName:%s\n",
					namespaceIndex, "default", pod.GetName(), pod.GetNamespace(), pod.Spec.NodeName)
			}
		}
	}

	if items, err := indexer.ByIndex(nodeNameIndex, "node2"); err != nil {
		fmt.Println("get index err:%s", err)
	} else {
		for _, podobj := range items {
			if pod, ok := podobj.(*corev1.Pod); ok {
				fmt.Printf("use indexkey:%s,value:%s, podName:%s,namespace:%s,nodeName:%s\n",
					nodeNameIndex, "node2", pod.GetName(), pod.GetNamespace(), pod.Spec.NodeName)
			}
		}
	}

	fmt.Println("-----indexkeys---")

	if list, err := indexer.IndexKeys(namespaceIndex, "default"); err != nil {
		fmt.Println(err)
	} else {
		fmt.Print(list)
	}
}
```

输出示例

```go
add 0,pod1 to indexer
add 1,pod2 to indexer
add 2,pod3 to indexer
use indexkey:namespaceIndex,value:default, podName:pod1,namespace:default,nodeName:node1
use indexkey:namespaceIndex,value:default, podName:pod2,namespace:default,nodeName:node2
use indexkey:nodeNameIndex,value:node2, podName:pod2,namespace:default,nodeName:node2
use indexkey:nodeNameIndex,value:node2, podName:pod3,namespace:podns,nodeName:node2
-----indexkeys
[default/pod1 default/pod2]
```

debug查看数据结构：

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220201161749.png)

```go
/*
计算所有key
indexers
{
	namespaceIndex: namespaceFunc,
	nodeNameIndex: nodeNameFunc,
}

indices
{
	namespaceIndex: {
		"default":["pod1","pod2"],
     "kube-system":["pod2","pod3"]
	},
	nodeNameIndex: {
		"node1":["pod1","pod2"],
        "node2":["pod2","pod3"]
	},
}

index {
	"default":["pod1","pod2"]
}
*/
```

这里我们定义的了两个索引键生成函数： namespaceFunc 与 nodeNameFunc，一个根据资源对象的命名空间来进行索引，一个根据资源对象所在的节点进行索引。然后定义了3个 Pod，前两个在 default 命名空间下面，另外一个在 kube-system 命名空间下面，然后通过 index.Add 函数添加 Pod 资源对象。然后通过 index.ByIndex 函数查询在名为 namespace 的索引器下面匹配索引键为 default 的 Pod 列表。也就是查询 default 这个命名空间下面的所有 Pod，这里就是前两个定义的 Pod。Indexers 是存储索引的，Indices 里面是存储的真正的数据（对象键）。

### 4.2 通过SharedInformer中获取

```go
// 生成clientset
func InitClientSet() *kubernetes.Clientset {
	configFlags := genericclioptions.NewConfigFlags(false)
	restConfig, err := configFlags.ToRESTConfig()
	Must(err)
	return kubernetes.NewForConfigOrDie(restConfig)
}

func main() {

	clientSet := InitClientSet()

	shareInformerFactory := informers.NewSharedInformerFactory(clientSet, 0)

	podInformer := shareInformerFactory.Core().V1().Pods()

	informer := podInformer.Informer()
	indexer := informer.GetIndexer()

	stopCh := make(chan struct{})
	shareInformerFactory.Start(stopCh)
	shareInformerFactory.WaitForCacheSync(stopCh)

	// 从indexer中获取

	items, err := indexer.ByIndex(cachetools.NamespaceIndex, metav1.NamespaceDefault)
	Must(err)

	// 通过indexer的ByIndex中获取内容，遍历pod对象
	for _, item := range items {
		pod, ok := item.(*corev1.Pod)
		if ok {
			fmt.Printf("indexName:%s, namespace:%s, podname:%s\n",
				cachetools.NamespaceIndex, metav1.NamespaceDefault, pod.GetName())
		}
	}
}
```

运行结果:

```go
indexName:namespace, namespace:default, podname:etcd-0
indexName:namespace, namespace:default, podname:podname1
indexName:namespace, namespace:default, podname:smartkm-api-k8s-exampledeploy-7cddf8cbf4-hn7f9
indexName:namespace, namespace:default, podname:etcd-1
indexName:namespace, namespace:default, podname:etcd-2
indexName:namespace, namespace:default, podname:smartkm-api-k8s-exampledeploy-7cddf8cbf4-ztv2k
```

使用shareInformerFactory创建一个podinformer，之后获取到indexer，通过indexer的ByIndex来获取indexName为namespace，value为default下的pod，在此为抛砖引玉，还可以通过informer.AddIndexers() 来添加自定义indexer来达到扩展性，根据实际业务来获取不同的对象。

## 参考链接

* https://www.jianshu.com/p/76e7b1a57d2c
* https://mp.weixin.qq.com/s/-qiB1KilhwtcjI61m_x3jA

