# Kubernetes安全之NPD

节点问题检测器（Node Problem Detector）是一个守护程序，用于监视和报告节点的健康状况（包括内核死锁、OOM、系统线程数压力、系统文件描述符压力等指标）。 你可以将节点问题探测器以 DaemonSet 或独立守护程序运行。 节点问题检测器从各种守护进程收集节点问题，并以[NodeCondition](https://kubernetes.io/zh/docs/concepts/architecture/nodes/#condition)和[Event](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.22/#event-v1-core)的形式报告给 API Server。



## 参考链接

* http://dockone.io/article/2434546
* https://github.com/kubernetes/node-problem-detector

