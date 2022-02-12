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

#### 3.1.2 队列实现

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

#### 3.1.3 通用队列方法

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

可以看到，在正常处理1元素时候，新添加进来的1不会立即被存入到queue中，而是待processing中1标记Done的时候，再将其插入到queue中，进行后续处理，注意dirty和processing都是Hash Map类型，其不用考虑无序，只保证去重即可。

### 3.2 延迟队列

#### 3.2.1 延迟队列接口

该接口涉及的目的就是可以避免某些失败的`items`陷入热循环. 因为某些`item`在出队列进行处理有可能失败, 失败了用户就有可能将该失败的`item`重新进队列, 如果短时间内又出队列有可能还是会失败(因为短时间内整个环境可能没有改变等等), 所以可能又重新进队列等等, 因此陷入了一种`hot-loop`.
所以就为用户提供了`AddAfter`方法, 可以允许用户告诉`DelayingInterface`过多长时间把该`item`加入队列中.

```go
type DelayingInterface interface {
	Interface
  // 在此刻过duration时间后再加入到queue中 
	AddAfter(item interface{}, duration time.Duration)
}

```

#### 3.2.2 延迟队列实现

延迟队列的实现为delayingType，其继承了通用接口，在其基础上主要新增了waitingForAddCh字段，来实现延迟的功能

waitingForAddCh：其默认初始大小为 1000，通过 AddAfter 方法插入元素时，是非阻塞状态的，只有当插入的元素大于或等于 1000 时，延迟队列才会处于阻塞状态。waitingForAddCh 字段中的数据通过 goroutine 运行的 waitingLoop 函数持久运行。

waitFor: 保存了包括数据`data`和该`item`什么时间起(`readyAt`)就可以进入队列了.
waitForPriorityQueue:waitForPriorityQueue 为 waitFor 项目实现优先级队列，是用于保存`waitFor`的优先队列, 按`readyAt`时间从早到晚排序，形成一个优先级队列， 先`ready`的`item`先出队列.

```go
type delayingType struct {
	Interface

	// clock tracks time for delayed firing
	clock clock.Clock

	// stopCh lets us signal a shutdown to the waiting loop
	stopCh chan struct{}
  // 保证仅仅发送一次关闭信号
	stopOnce sync.Once

  // 在触发之前确保等待的时间不超过maxWait
	heartbeat clock.Ticker

  // waitingForAddCh是一个缓冲channel
	waitingForAddCh chan *waitFor

	// 记录重试次数
	metrics retryMetrics
}

// 保存了包括数据`data`和该`item`什么时间起(`readyAt`)就可以进入队列了.
type waitFor struct {
	data    t
	readyAt time.Time
	// index in the priority queue (heap)
	index int
}
// waitForPriorityQueue 为 waitFor 项目实现优先级队列
// 是用于保存`waitFor`的优先队列, 按`readyAt`时间从早到晚排序. 先`ready`的`item`先出队列.
type waitForPriorityQueue []*waitFor
```

#### 3.2.3 延迟队列方法

* AddAfter

AddAfter方法为延迟队列添加元素，除了具体元素外，还需要传入延迟多长时间。

```go
func (q *delayingType) AddAfter(item interface{}, duration time.Duration) {
	// don't add if we're already shutting down
	if q.ShuttingDown() {
		return
	}

	q.metrics.retry()

	// 如果传入的时间小于等于0则立即添加
	if duration <= 0 {
		q.Add(item)
		return
	}


	select {
	case <-q.stopCh:
		// unblock if ShutDown() is called
    // 构造一个waitFor对象传入到waitingForAddCh通道中
	case q.waitingForAddCh <- &waitFor{data: item, readyAt: q.clock.Now().Add(duration)}:
	}
}
```

* waitingLoop

waitingLoop 方法一直运行到工作队列关闭，并检查要添加的项目列表，利用channel(q.waitingForAddCh)一直接受从AddAfter过来的waitFor，将已经ready的item添加到队列中。

```go
func (q *delayingType) waitingLoop() {
	defer utilruntime.HandleCrash()
  // 创建一个占位channel，当list没有元素的时候，利用其实现等待
	never := make(<-chan time.Time)

	// 构造一个定时器，当等待队列头部的元素准备好后，该定时器失效
  var nextReadyAtTimer clock.Timer

  // 构造一个优先队列
	waitingForQueue := &waitForPriorityQueue{}
	heap.Init(waitingForQueue)
	// 利用map实现去重，如果在此添加相同元素，仅实现更新时间
	waitingEntryByData := map[t]*waitFor{}

	for {
		if q.Interface.ShuttingDown() {
			return
		}

		now := q.clock.Now()
		// 如果优先队列有元素
		for waitingForQueue.Len() > 0 {
			entry := waitingForQueue.Peek().(*waitFor)
      // 如果队列头部的元素时间已经大于现在时间，则表明整个优先级队列中的元素都大于当前时间，还没到插入元素的时间，则直接break
      // 第一个元素时间是最小的
			if entry.readyAt.After(now) {
				break
			}
			// 否则说明以及改到了插入数据时间了，执行插入到通用队列中
			entry = heap.Pop(waitingForQueue).(*waitFor)
			q.Add(entry.data)
      // 并从优先级队列删除
			delete(waitingEntryByData, entry.data)
		}
		// 当优先级队列中没有元素，则在后面进行等待
		nextReadyAt := never
    // 如果优先级队列中有元素
		if waitingForQueue.Len() > 0 {
			if nextReadyAtTimer != nil {
				nextReadyAtTimer.Stop()
			}
      
      // 获取第一个元素
			entry := waitingForQueue.Peek().(*waitFor)
      // 构造一个nextReady时间等待器
			nextReadyAtTimer = q.clock.NewTimer(entry.readyAt.Sub(now))
			nextReadyAt = nextReadyAtTimer.C()
		}

		select {
		case <-q.stopCh:
			return
		// 定时器，如果过一段时间在没有任何数据，则进行一次大循环
		case <-q.heartbeat.C():
			// continue the loop, which will add ready items

		case <-nextReadyAt:
			// continue the loop, which will add ready items

    // addAfter放入的元素从该处获取
		case waitEntry := <-q.waitingForAddCh:
      // 如果元素时间大于当前时间，则添加到优先级队列中
			if waitEntry.readyAt.After(q.clock.Now()) {
				insert(waitingForQueue, waitingEntryByData, waitEntry)
			} else {
        // 时间到了则添加进通用队列
				q.Add(waitEntry.data)
			}
			// 将channel中的元素全部取出来
			drained := false
			for !drained {
				select {
				case waitEntry := <-q.waitingForAddCh:
					if waitEntry.readyAt.After(q.clock.Now()) {
						insert(waitingForQueue, waitingEntryByData, waitEntry)
					} else {
						q.Add(waitEntry.data)
					}
				default:
					drained = true
				}
			}
		}
	}
}

// 添加元素至优先级队列，如果元素相同则进行更新时间
func insert(q *waitForPriorityQueue, knownEntries map[t]*waitFor, entry *waitFor) {
	// 判断队列中是否已经存在该元素
	existing, exists := knownEntries[entry.data]
  // 如果存则，则直接更新元素
	if exists {
		if existing.readyAt.After(entry.readyAt) {
			existing.readyAt = entry.readyAt
			heap.Fix(q, existing.index)
		}

		return
	}
	// 不存在则添加进优先级队列
	heap.Push(q, entry)
	knownEntries[entry.data] = entry
}

```

至此就完成了限速队列的源码分析，其在通用队列之上构造优先级队列，如果时间没到的添加到优先级队列中，待时间到了添加到通用队列中进行正常处理，其中优先级队列实现利用了golang底层库heap，又兴趣的可以进一步阅读源码。

### 3.3 限速队列

#### 3.3.1 限速队列接口

限速队列，顾名思义就是在元素入对速度有一定限制，其利用延迟队列的特性实现对元素插入队列的速度控制，因此限速队列也是扩展的延迟队列，新增有AddRateLimited、Forget、NumRequeues 3个方法。

```go
type RateLimitingInterface interface {
	DelayingInterface

	// 增加一个元素大限速队列
	AddRateLimited(item interface{})

  // 表示完成该元素重试，丢弃掉该元素
	Forget(item interface{})

	// 查询元素入队列的次数
	NumRequeues(item interface{}) int
}
```

#### 3.3.2 限速队列实现

我们看其中主要还是调用rateLimiter 的接口方法，

```go
// rateLimitingType 是限速队列的具体实现
type rateLimitingType struct {
  // 继承延迟队列
	DelayingInterface
	// 新增限速接口
	rateLimiter RateLimiter
}

// 获取到限速器延迟时间，然后加入到延迟队列
func (q *rateLimitingType) AddRateLimited(item interface{}) {
	q.DelayingInterface.AddAfter(item, q.rateLimiter.When(item))
}

// 通过限速器获取元素加入队列的次数
func (q *rateLimitingType) NumRequeues(item interface{}) int {
	return q.rateLimiter.NumRequeues(item)
}

// 通过限速器丢弃元素
func (q *rateLimitingType) Forget(item interface{}) {
	q.rateLimiter.Forget(item)
}
```

#### 3.3.3 限速器方法



```go
// 限速器接口
type RateLimiter interface {
	// 获取元素等待时间
	When(item interface{}) time.Duration
	// 释放指定元素
	Forget(item interface{})
	// 返回某个对象重新入队列次数
	NumRequeues(item interface{}) int
}
```

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220206194921.png) 

* BucketRateLimiter

令牌桶算法是通过 Go 语言的第三方库 golang.org/x/time/rate 实现的。令牌桶算法内部实现了一个存放 token（令牌）的“桶”，初始时“桶”是空的，token 会以固定速率往“桶”里填充，直到将其填满为止，多余的 token 会被丢弃。每个元素都会从令牌桶得到一个 token，只有得到 token 的元素才允许通过（accept），而没有得到 token 的元素处于等待状态。令牌桶算法通过控制发放 token 来达到限速目的。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220206212915.png)

```go
type BucketRateLimiter struct {
	*rate.Limiter
}


func (r *BucketRateLimiter) When(item interface{}) time.Duration {
	return r.Limiter.Reserve().Delay()
}

func (r *BucketRateLimiter) NumRequeues(item interface{}) int {
	return 0
}

func (r *BucketRateLimiter) Forget(item interface{}) {
}
```

在实例化 rate.NewLimiter 后，传入 r 和 b 两个参数，其中 r 参数表示每秒往“桶”里填充的 token 数量，b 参数表示令牌桶的大小（即令牌桶最多存放的 token 数量）。我们假定 r 为 10，b 为 100。假设在一个限速周期内插入了 1000 个元素，通过 r.Limiter.Reserve().Delay 函数返回指定元素应该等待的时间，那么前 b（即 100）个元素会被立刻处理，而后面元素的延迟时间分别为 item100/100ms、item101/200ms、item102/300ms、item103/400ms，以此类推。

```go
func DefaultControllerRateLimiter() RateLimiter {
	return NewMaxOfRateLimiter(
		NewItemExponentialFailureRateLimiter(5*time.Millisecond, 1000*time.Second),
		// 10 qps, 100 bucket size.  This is only for retry speed and its only the overall factor (not per item)
    // 默认令牌桶限速器
		&BucketRateLimiter{Limiter: rate.NewLimiter(rate.Limit(10), 100)},
	)
}
```

* ItemExponentialFailureRateLimiter

排队指数算法将相同元素的排队数作为指数，排队数增大，速率限制呈指数级增长，但其最大值不会超过 maxDelay。元素的排队数统计是有限速周期的，一个限速周期是指从执行 AddRateLimited 方法到执行完 Forget 方法之间的时间。如果该元素被 Forget 方法处理完，则清空排队数。排队指数算法的核心实现代码示例如下：

```go

func (r *ItemExponentialFailureRateLimiter) When(item interface{}) time.Duration {
	r.failuresLock.Lock()
	defer r.failuresLock.Unlock()

	exp := r.failures[item]
	r.failures[item] = r.failures[item] + 1

	// The backoff is capped such that 'calculated' value never overflows.
	backoff := float64(r.baseDelay.Nanoseconds()) * math.Pow(2, float64(exp))
	if backoff > math.MaxInt64 {
		return r.maxDelay
	}

	calculated := time.Duration(backoff)
	if calculated > r.maxDelay {
		return r.maxDelay
	}

	return calculated
}

func (r *ItemExponentialFailureRateLimiter) NumRequeues(item interface{}) int {
	r.failuresLock.Lock()
	defer r.failuresLock.Unlock()

	return r.failures[item]
}

func (r *ItemExponentialFailureRateLimiter) Forget(item interface{}) {
	r.failuresLock.Lock()
	defer r.failuresLock.Unlock()

	delete(r.failures, item)
}
```

failures 字段用于统计元素排队数，每当 AddRateLimited 方法插入新元素时，会为该字段加 1；另外，baseDelay 字段是最初的限速单位（默认为 5ms），maxDelay 字段是最大限速单位（默认为 1000s）

限速队列利用延迟队列的特性，延迟多个相同元素的插入时间，达到限速目的。

我们假定 baseDelay 是 1 * time.Millisecond，maxDelay 是 1000 * time.Second。假设在一个限速周期内通过 AddRateLimited 方法插入 10 个相同元素，那么第 1 个元素会通过延迟队列的 AddAfter 方法插入并设置延迟时间为 1ms（即 baseDelay），第 2 个相同元素的延迟时间为 2ms，第 3 个相同元素的延迟时间为 4ms，第 4 个相同元素的延迟时间为 8ms，第 5 个相同元素的延迟时间为 16ms……第 10 个相同元素的延迟时间为 512ms，最长延迟时间不超过 1000s（即 maxDelay）。

* ItemFastSlowRateLimiter

计数器算法是限速算法中最简单的一种，其原理是：限制一段时间内允许通过的元素数量，例如在 1 分钟内只允许通过 100 个元素，每插入一个元素，计数器自增 1，当计数器数到 100 的阈值且还在限速周期内时，则不允许元素再通过。但 WorkQueue 在此基础上扩展了 fast 和 slow 速率。



计数器算法提供了 4 个主要字段：failures、fastDelay、slowDelay 及 maxFastAttempts。其中，failures 字段用于统计元素排队数，每当 AddRateLimited 方法插入新元素时，会为该字段加 1；而 fastDelay 和 slowDelay 字段是用于定义 fast、slow 速率的；另外，maxFastAttempts 字段用于控制从 fast 速率转换到 slow 速率。计数器算法核心实现的代码示例如下：

```go
type ItemFastSlowRateLimiter struct {
	failuresLock sync.Mutex
	failures     map[interface{}]int

	maxFastAttempts int
	fastDelay       time.Duration
	slowDelay       time.Duration
}

func (r *ItemFastSlowRateLimiter) When(item interface{}) time.Duration {
	r.failuresLock.Lock()
	defer r.failuresLock.Unlock()

	r.failures[item] = r.failures[item] + 1

	if r.failures[item] <= r.maxFastAttempts {
		return r.fastDelay
	}

	return r.slowDelay
}

func (r *ItemFastSlowRateLimiter) NumRequeues(item interface{}) int {
	r.failuresLock.Lock()
	defer r.failuresLock.Unlock()

	return r.failures[item]
}

func (r *ItemFastSlowRateLimiter) Forget(item interface{}) {
	r.failuresLock.Lock()
	defer r.failuresLock.Unlock()

	delete(r.failures, item)
}

```



假设 fastDelay 是 5 * time.Millisecond，slowDelay 是 10 * time.Second，maxFastAttempts 是 3。在一个限速周期内通过 AddRateLimited 方法插入 4 个相同的元素，那么前 3 个元素使用 fastDelay 定义的 fast 速率，当触发 maxFastAttempts 字段时，第 4 个元素使用 slowDelay 定义的 slow 速率。

* MaxOfRateLimiter

```go
type MaxOfRateLimiter struct {
	limiters []RateLimiter
}
func (r *MaxOfRateLimiter) When(item interface{}) time.Duration {
	ret := time.Duration(0)
	for _, limiter := range r.limiters {
		curr := limiter.When(item)
		if curr > ret {
			ret = curr
		}
	}

	return ret
}

func (r *MaxOfRateLimiter) NumRequeues(item interface{}) int {
	ret := 0
	for _, limiter := range r.limiters {
		curr := limiter.NumRequeues(item)
		if curr > ret {
			ret = curr
		}
	}

	return ret
}

func (r *MaxOfRateLimiter) Forget(item interface{}) {
	for _, limiter := range r.limiters {
		limiter.Forget(item)
	}
}

```

混合模式是将多种限速算法混合使用，即多种限速算法同时生效。例如，同时使用排队指数算法和令牌桶算法，代码示例如下：

```go
func DefaultControllerRateLimiter() RateLimiter {
	return NewMaxOfRateLimiter(
		NewItemExponentialFailureRateLimiter(5*time.Millisecond, 1000*time.Second),
		// 10 qps, 100 bucket size.  This is only for retry speed and its only the overall factor (not per item)
		&BucketRateLimiter{Limiter: rate.NewLimiter(rate.Limit(10), 100)},
	)
}
```



## 四 小试牛刀

### 4.1 通用队列



### 4.2 延迟队列



### 4.3 限速队列













## 参考链接

* https://mp.weixin.qq.com/s/-qiB1KilhwtcjI61m_x3jA
* https://xie.infoq.cn/article/63258ead84821bc3e276de1f7
* https://www.jianshu.com/p/d42da5d5f586

