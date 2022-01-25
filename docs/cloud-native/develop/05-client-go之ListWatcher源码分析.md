# 一 背景

kubernetes所有API对象都存储在etcd中，并只能通过apiserver访问。如果很多客户端频繁的列举全量对象（比如列举所有的Pod），这会造成apiserver不堪重负。

ListerWatcher是Lister和Watcher的结合体，ListerWatcher负责列举全量对象，Watcher负责监视（本文将watch翻译为监视）对象的增量变化。

通过客户端缓存，至在没有任何状态变化的情况下只需要读取本地缓存即可，减少对API-server的压力效率提升显而易见。通过列举全量对象完成本地缓存，而监视增量则是为了及时的将apiserver的状态变化更新到本地缓存。所以，在apiserver与客户端之间绝大部分传输的是对象的增量变化，当然在异常的情况下还是要重新列举一次全量对象。

本文值得客户端本地缓存就是Indexer，client-go不仅实现了缓存，同时还加了索引，进一步提升了检索效率。

# 二 ListerWatcher

Kubernetes 控制面 (control plane) 的核心是 **API 服务器 (API server)**。API 服务器负责提供 HTTP API，以供用户，集群中的不同部分和集群外部组件相互通信。控制器也不例外，所有控制器都通过 API 获取集群的当前状态，也通过 API 对集群状态进行修改。

list-watch，作为k8s系统中统一的异步消息传递方式，对系统的性能、数据一致性起到关键性的作用。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220120151429.png)

值得一提的是，Kubernetes 提供了 [watch](https://link.juejin.cn?target=https%3A%2F%2Fkubernetes.io%2Fdocs%2Freference%2Fusing-api%2Fapi-concepts%2F%23efficient-detection-of-changes) 机制方便客户端实时获取集群状态，有了这个接口，控制器才得以无延迟（准确地说是低延迟）地对状态变更作出响应。这里指的 "状态变更"，就是我们常说的**事件 (event)**。

## 2.1 EventType

```go
// EventType defines the possible types of events.
type EventType string

const (
	Added    EventType = "ADDED"
	Modified EventType = "MODIFIED"
	Deleted  EventType = "DELETED"
	Bookmark EventType = "BOOKMARK"
	Error    EventType = "ERROR"
)
```

## 2.2 ListerWatcher定义

```go
// 复制代码
// client-go/tools/cache/listwatch.go
// Lister is any object that knows how to perform an initial list.
type Lister interface {
	// List should return a list type object; the Items field will be extracted, and the
	// ResourceVersion field will be used to start the watch in the right place.
	List(options metav1.ListOptions) (runtime.Object, error)
}

// Watcher is any object that knows how to start a watch on a resource.
type Watcher interface {
	// Watch should begin a watch at the specified version.
	Watch(options metav1.ListOptions) (watch.Interface, error)
}

// ListerWatcher is any object that knows how to perform an initial list and start a watch on a resource.
type ListerWatcher interface {
	Lister
	Watcher
}

```

## 2.3 创建ListWatcher对象

```go

// NewListWatchFromClient creates a new ListWatch from the specified client, resource, namespace and field selector.
func NewListWatchFromClient(c Getter, resource string, namespace string, fieldSelector fields.Selector) *ListWatch {
	optionsModifier := func(options *metav1.ListOptions) {
		options.FieldSelector = fieldSelector.String()
	}
	return NewFilteredListWatchFromClient(c, resource, namespace, optionsModifier)
}
```

# 三 小试牛刀

```shell
$ kubectl proxy
# 进行listwatch default名称空间下的pods
$ curl "127.0.0.1:8001/api/v1/namespaces/default/pods?watch=1"
# 创建pod进行观察
$ kubectl run nginx --image=nginx

```

```shell 
{"type":"ADDED","object":{"kind":"Pod","apiVersion":"v1","metadata":{"name":"nginx","namespace":"default","uid":"8d0548ce-fb67-4b71-93ec-59ad67b429d9","resourceVersion":"2925331","creationTimestamp":"2022-01-20T07:32:22Z","labels":{"run":"nginx"},"managedFields":[{"manager":"kubectl-run","operation":"Update","apiVersion":"v1","time":"2022-01-20T07:32:22Z","fieldsType":"FieldsV1","fieldsV1":{"f:metadata":{"f:labels":{".":{},"f:run":{}}},"f:spec":{"f:containers":{"k:{\"name\":\"nginx\"}":{".":{},"f:image":{},"f:imagePullPolicy":{},"f:name":{},"f:resources":{},"f:terminationMessagePath":{},"f:terminationMessagePolicy":{}}},"f:dnsPolicy":{},"f:enableServiceLinks":{},"f:restartPolicy":{},"f:schedulerName":{},"f:securityContext":{},"f:terminationGracePeriodSeconds":{}}}}]},"spec":{"volumes":[{"name":"kube-api-access-nc5v8","projected":{"sources":[{"serviceAccountToken":{"expirationSeconds":3607,"path":"token"}},{"configMap":{"name":"kube-root-ca.crt","items":[{"key":"ca.crt","path":"ca.crt"}]}},{"downwardAPI":{"items":[{"path":"namespace","fieldRef":{"apiVersion":"v1","fieldPath":"metadata.namespace"}}]}}],"defaultMode":420}}],"containers":[{"name":"nginx","image":"nginx","resources":{},"volumeMounts":[{"name":"kube-api-access-nc5v8","readOnly":true,"mountPath":"/var/run/secrets/kubernetes.io/serviceaccount"}],"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File","imagePullPolicy":"Always"}],"restartPolicy":"Always","terminationGracePeriodSeconds":30,"dnsPolicy":"ClusterFirst","serviceAccountName":"default","serviceAccount":"default","securityContext":{},"schedulerName":"default-scheduler","tolerations":[{"key":"node.kubernetes.io/not-ready","operator":"Exists","effect":"NoExecute","tolerationSeconds":300},{"key":"node.kubernetes.io/unreachable","operator":"Exists","effect":"NoExecute","tolerationSeconds":300}],"priority":0,"enableServiceLinks":true,"preemptionPolicy":"PreemptLowerPriority"},"status":{"phase":"Pending","qosClass":"BestEffort"}}}
```

# 四 代码实现

编写代码对default名称空间下的configmap进行list watch。

```go
package main

import (
	"fmt"
	v1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/api/meta"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/fields"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/cache"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/client-go/util/homedir"
	"os"
	"os/signal"
	"path/filepath"
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

func InitListerWatcher(clientSet *kubernetes.Clientset, resource, namespace string, fieldSelector fields.Selector) cache.ListerWatcher {
	restClient := clientSet.CoreV1().RESTClient()
	return cache.NewListWatchFromClient(restClient, resource, namespace, fieldSelector)
}

func main() {
	clientSet, err := InitClientSet()
	if err != nil {
		panic(err)
	}

	// 什么常量
	resource := "configmaps"
	namespace := "default"

	configMapListerWatcher := InitListerWatcher(clientSet, resource, namespace, fields.Everything())

	// 1. list操作
	listObj, err := configMapListerWatcher.List(metav1.ListOptions{})

	// meta 包封装了一些处理 runtime.Object 对象的方法，屏蔽了反射和类型转换的过程，
	// 提取出的 items 类型为 []runtime.Object
	items, err := meta.ExtractList(listObj)
	if err != nil {
		Must(err)
	}
	fmt.Println("list result:")
	for _, item := range items {
		configmaps, ok := item.(*v1.ConfigMap)
		if !ok {
			return
		}
		fmt.Printf("namespace: %s, resource name:%s\n", configmaps.Namespace, configmaps.Name)
	}

	// 2. watch 操作
	listMetaInterface, err := meta.ListAccessor(listObj)
	if err != nil {
		Must(err)
	}
	resourceVersion := listMetaInterface.GetResourceVersion()

	watchObj, err := configMapListerWatcher.Watch(metav1.ListOptions{
		ResourceVersion: resourceVersion,
	})

	// 接收信号
	stopCh := make(chan os.Signal)
	signal.Notify(stopCh, os.Interrupt)
	fmt.Println("Start watching...")
	for {
		select {
		case <-stopCh:
			fmt.Println("exit")
			return
		case event, ok := <-watchObj.ResultChan():
			if !ok {
				fmt.Println("Broken channel")
				break
			}
			configmaps, ok := event.Object.(*v1.ConfigMap)
			if !ok {
				return
			}
			fmt.Printf("eventType: %s, watch obj:%s\n", event.Type, configmaps.Name)
		}
	}
}
```



进行创建configmap测试

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220120153657.png)

# 其他

* ListerWatcher就是为SharedIndexInformer列举全量对象、监视对象增量变化设计的接口，实现就是Clientset的List和Watch函数；
* SharedIndexInformer利用ListerWatcher实现了本地缓存与apiserver之间的状态一致性；
* 不仅可以提升客户端访问API对象的效率，同时可以将对象的增量变化回调给使用者；
* 从原理上讲，可以用etcd的clientv3.Client实现ListerWatcher，SharedIndexInformer同步etcd的对象，这样一些简单的醒目就可以复用SharedIndexInformer了，毕竟不是所有的项目都需要一个apiserver；

# 参考资料

* https://blog.csdn.net/weixin_42663840/article/details/114379569