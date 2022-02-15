# x-DeltaFIFO 与 Indexer关系

## 一 前言

在刚开始看DeltaFIFO的时候，不太清楚具体的数据存储在哪里，具体数据流向何处，本文通过程序debug可以更直观的看出DeltaFIFO中的属性与Indexer关系。

## 二 DeltaFIFO数据结构

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220215132954.png)

```go
type DeltaFIFO struct {
	// lock/cond protects access to 'items' and 'queue'.
	lock sync.RWMutex
	cond sync.Cond

	// `items` maps a key to a Deltas.
	// Each such Deltas has at least one Delta.
	items map[string]Deltas

	// `queue` maintains FIFO order of keys for consumption in Pop().
	// There are no duplicates in `queue`.
	// A key is in `queue` if and only if it is in `items`.
	queue []string

	// populated is true if the first batch of items inserted by Replace() has been populated
	// or Delete/Add/Update/AddIfNotPresent was called first.
	populated bool
	// initialPopulationCount is the number of items inserted by the first call of Replace()
	initialPopulationCount int

	// keyFunc is used to make the key used for queued item
	// insertion and retrieval, and should be deterministic.
	keyFunc KeyFunc

	// knownObjects list keys that are "known" --- affecting Delete(),
	// Replace(), and Resync()
	knownObjects KeyListerGetter

	// Used to indicate a queue is closed so a control loop can exit when a queue is empty.
	// Currently, not used to gate any of CRED operations.
	closed bool

	// emitDeltaTypeReplaced is whether to emit the Replaced or Sync
	// DeltaType when Replace() is called (to preserve backwards compat).
	emitDeltaTypeReplaced bool
}

// state of the object before it was deleted.
type Delta struct {
	Type   DeltaType
	Object interface{}
}

// Deltas is a list of one or more 'Delta's to an individual object.
// The oldest delta is at index 0, the newest delta is the last one.
type Deltas []Delta
```

DeltaFIFO中存储的内容

key：是通过keyFunc计算获取的，默认使用cache.MetaNamespaceKeyFunc来获取，为namespace/resourcename，例如下图中的default/etcd-2

value：是具体的对象Deltas切片，内部包含Type和Object，其中Type为DeltaType，为关注资源的操作类型。

object为具体对象的在k8s资源中的元数据。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220215133036.png)

## 三 Indexer数据结构图

```go
type cache struct {
	// cacheStorage bears the burden of thread safety for the cache
	cacheStorage ThreadSafeStore
	// keyFunc is used to make the key for objects stored in and retrieved from items, and
	// should be deterministic.
	keyFunc KeyFunc
}

// threadSafeMap implements ThreadSafeStore
type threadSafeMap struct {
	lock  sync.RWMutex
	items map[string]interface{}

	// indexers maps a name to an IndexFunc
	indexers Indexers
	// indices maps a name to an Index
	indices Indices
}
```

Indexer为DeltaFIFO中的knownObjects，包含keyFunc和cacheStorage

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220215133141.png)



## 四 对比

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220215133516.png)

可以对比来看，在DeltaFIFO中的item的value中是一个deltas切片，内部的值包含type和具体的object对象

indexer中item的值直接是对象本身。

## 总结

DeltaFIFO中的Item存储了具体的元素，为了后期便于索引，在内部还包含了一个knownObjects，其内部同样存储了一份数据，deltaFIFO最后Pop出来数据后添加至indexer中，和调用用户注册的resourceEventHandler。