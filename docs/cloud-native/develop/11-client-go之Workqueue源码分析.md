# 11.Client-go源码分析之WorkQueue

## 一 前言

在上文SharedInformer中，最后将资源对象已经写入到事件回调函数中，此后我们直接处理这些数据即可，但是我们使用golang中的chanel来处理会存在处理效率低，存在数据大并发量，数据积压等其他异常情况，为此client-go单独将workqueue提出来，作为公共组件，不仅可以在Kubernetes内部使用，还可以供Client-go使用，用户侧可以通过Workqueue相关方法进行更加灵活的队列处理，如失败重试，延迟入队，限速控制等，实现非阻塞异步逻辑处理。

## 二 WorkQueue相关概念及构成

### 2.1 分类

WorkQueue 支持 3 种队列，并提供了 3 种接口，不同队列实现可应对不同的使用场景，分别介绍如下。

- **Interface**：通用队FIFO 队列接口，先进先出队列，并支持去重机制。
- **DelayingInterface**：延迟队列接口，基于 Interface 接口封装，延迟一段时间后再将元素存入队列。
- **RateLimitingInterface**：限速队列接口，基于 DelayingInterface 接口封装，支持元素存入队列时进行速率限制。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220202134307.png)

从图中可以看到，workqueue.RateLimitingInterface 集成了 DelayingInterface，DelayingInterface 集成了 Interface，最终由 rateLimitingType 进行实现，提供了 rateLimit 限速、delay 延时入队(由优先级队列通过小顶堆实现)、queue 队列处理 三大核心能力，另外，在代码中可看到 K8s 实现了三种 RateLimiter：BucketRateLimiter, ItemExponentialFailureRateLimiter, ItemFastSlowRateLimiter。

### 2.2 特征

WorkQueue 称为工作队列，Kubernetes 的 WorkQueue 队列与普通 FIFO（先进先出，First-In, First-Out）队列相比，实现略显复杂，它的主要功能在于标记和去重，并支持如下特性。

- **有序**：按照添加顺序处理元素（item）。
- **去重**：相同元素在同一时间不会被重复处理，例如一个元素在处理之前被添加了多次，它只会被处理一次。
- **并发性**：多生产者和多消费者。
- **标记机制**：支持标记功能，标记一个元素是否被处理，也允许元素在处理时重新排队。
- **通知机制**：ShutDown 方法通过信号量通知队列不再接收新的元素，并通知 metric goroutine 退出。
- **延迟**：支持延迟队列，延迟一段时间后再将元素存入队列。
- **限速**：支持限速队列，元素存入队列时进行速率限制。限制一个元素被重新排队（Reenqueued）的次数。
- **Metric**：支持 metric 监控指标，可用于 Prometheus 监控。

## 三 源码分析

### 3.1 通用队列

#### 3.1.1 队列接口

通用队列FIFO 队列支持最基本的队列方法，例如插入元素、获取元素、获取队列长度等。另外，WorkQueue 中的限速及延迟队列都基于 Interface 接口实现，其提供如下方法：

```go
// k8s.io/client-go/util/workqueue/queue.go
type Interface interface {
  // 为队列添添加一个元素
	Add(item interface{})
  // 获取队列长度
	Len() int
  // 从头部获取一个元素
	Get() (item interface{}, shutdown bool)
  // 标记一个元素处理完成
	Done(item interface{})
  // 关闭队列
	ShutDown()
  // 获取队列是否正在关闭
	ShuttingDown() bool
}
```

#### 3.1.2 队列具体实现

通用队列具体实现为Type，其中有非常重要的三个属性，queue为一个任意类型元素的slice，是具体存储元素的地方，其实一个；dirty表示元素在内存中，可以对元素进行去重；processing表示当前正在处理中的元素。

```go
// 具体queue的实现
type Type struct {
	// 存储具体元素，类型为slice
	queue []t
	// 定义所有的元素被处理，是以map类型，可以对元素去重
	dirty set
	// 标记元素是否正在处理。
  // 有些元素可能同时存在processing和dirty中，当处理完这个元素，从集合删除的时候，如果发现元素还在dirty集合中，在此将其添加至queue队列中
	processing set

	cond *sync.Cond

	shuttingDown bool

	metrics queueMetrics

	unfinishedWorkUpdatePeriod time.Duration
	clock                      clock.Clock
}

// 空结构体
type empty struct{}
// t为空接口类型，表示任意类型
type t interface{}  
// set 集合，用 map 来模拟 set 集合（元素不重复），更快效率的查找元素
type set map[t]empty
```

#### 3.1.3 具体方法

* Add

Add方法为添加一个元素到item中，首先会判断队列是否关闭，其次会判断元素是否以及存在dirty中，如果在则返回实现去重，同时判断元素是否正在处理中，如果存在元素处理中，则不将其添加进queue中，而是等后期标记Done时候，将dirty中的元素添加进入queue中，如果没有正在处理，则元素成功添加进queue中。

```go
func (q *Type) Add(item interface{}) {
	q.cond.L.Lock()
	defer q.cond.L.Unlock()
  // 如果队列已经关闭则直接返回
	if q.shuttingDown {
		return
	}
  // 如果dirty中已经有该元素，则直接返回，目的是实现去重
	if q.dirty.has(item) {
		return
	}

	q.metrics.add(item)
  // 添加元素到dirty中
	q.dirty.insert(item)
  // 如果元素正在处理，则直接返回
	if q.processing.has(item) {
		return
	}
  // 添加元素到queue中
	q.queue = append(q.queue, item)
  // 通知其他协程接触阻塞
	q.cond.Signal()
}
```

* Get

Get方法，从队列头部弹出一个元素将放到proocessing集合中，从dirty集合中移除。

```go
func (q *Type) Get() (item interface{}, shutdown bool) {
	q.cond.L.Lock()
	defer q.cond.L.Unlock()
  // 如果队列为关闭，且队列长度为0，则进行阻塞，待元素添加进队列发送解除阻塞通知执行
	for len(q.queue) == 0 && !q.shuttingDown {
		q.cond.Wait()
	}
  // 如果队列长度为0，则关闭队列
	if len(q.queue) == 0 {
		// We must be shutting down.
		return nil, true
	}
  // 从队列头部填出第一个元素
	item, q.queue = q.queue[0], q.queue[1:]

	q.metrics.get(item)
  // 将元素插入到processing中，标记为正在处理
	q.processing.insert(item)
  // 从dirty中删除队列
	q.dirty.delete(item)

	return item, false
}
```

* Done

Done方法标示元素以处理完成，该方法为了保证同一时刻不能有两个相同的元素被处理，如果正在处理一个元素，由添加进来同样的一个，先将其添加进dirty中，在Done方法中，再检查dirty将新元素重新入队列。

```go
func (q *Type) Done(item interface{}) {
	q.cond.L.Lock()
	defer q.cond.L.Unlock()

	q.metrics.done(item)
	// 将processing中的元素移除
	q.processing.delete(item)
  // 判断元素是否还在dirty中，如果在则将其重新添加至queue中
	if q.dirty.has(item) {
		q.queue = append(q.queue, item)
		q.cond.Signal()
	}
}
```

比较重要的就这三个方法，其他的Len/ShutDown比较简单，可以自己去参考源码。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220204193947.png)





### 3.2 延迟队列





### 3.3 限速队列

## 



## 四 小试牛刀

### 4.1 通用队列



### 4.2 延迟队列



### 4.3 限速队列













## 参考链接

* https://mp.weixin.qq.com/s/-qiB1KilhwtcjI61m_x3jA
* https://xie.infoq.cn/article/63258ead84821bc3e276de1f7
* https://www.jianshu.com/p/d42da5d5f586

