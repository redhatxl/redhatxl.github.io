# 11.Client-go源码分析之WorkQueue

## 一 



![image-20220202130145196](/Users/xuel/Library/Application Support/typora-user-images/image-20220202130145196.png)

通过上一步 processorListener 回调函数，交给内部 ResourceEventHandler 进行真正的增删改(CUD) 处理，分别调用 OnAdd/OnUpdate/OnDelete 注册函数进行处理。

为了**快速处理而不阻塞 processorListener 回调函数，一般使用 workqueue 进行异步化解耦合处理**，其实现如下：

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220202134307.png)

从图中可以看到，workqueue.RateLimitingInterface 集成了 DelayingInterface，DelayingInterface 集成了 Interface，最终由 rateLimitingType 进行实现，提供了 rateLimit 限速、delay 延时入队(由优先级队列通过小顶堆实现)、queue 队列处理 三大核心能力。

另外，在代码中可看到 K8s 实现了三种 RateLimiter：BucketRateLimiter, ItemExponentialFailureRateLimiter, ItemFastSlowRateLimiter，Controller 默认采用了前两种如下：

这样，在用户侧可以通过调用 workqueue 相关方法进行灵活的队列处理，比如失败多少次就不再重试，失败了延时入队的时间控制，队列的限速控制(QPS)等，实现非阻塞异步化逻辑处理。

## 参考链接

* https://mp.weixin.qq.com/s/-qiB1KilhwtcjI61m_x3jA

