# 3.Client-go 的使用

## 一 client-go简介

近期有需求要对k8s的一些数据进行自定义整合，利用client-go可以快速方便的实现需求，在K8s运维中，我们可以使用kubectl、客户端库或者REST请求来访问K8S API。而实际上，无论是kubectl还是客户端库，都是封装了REST请求的工具。client-go作为一个客户端库，能够调用K8S API，实现对K8S集群中资源对象（包括deployment、service、ingress、replicaSet、pod、namespace、node等）的增删改查等操作。

## 二 简介

### 2.1 结构图

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/517cccb9e9424b76975d18f45d68aa97~tplv-k3u1fbpfcp-zoom-1.image)

从上图可以看到client-go 中包含四种client，`ClientSet`，`DynamicClient`，`DiscoveryClient`都是`RestClient`的具体实现。

### 2.2 目录结构

```properties
# tree client-go/ -L 1
client-go/
  ├── discovery
  ├── dynamic
  ├── informers
  ├── kubernetes
  ├── listers
  ├── plugin
  ├── rest
  ├── scale
  ├── tools
  ├── transport
  └── util
```

- `discovery`：提供 `DiscoveryClient` 发现客户端
- `dynamic`：提供 `DynamicClient` 动态客户端
- `informers`：每种 kubernetes 资源的 Informer 实现
- `kubernetes`：提供 `ClientSet` 客户端
- `listers`：为每一个 Kubernetes 资源提供 Lister 功能，该功能对 Get 和 List 请求提供只读的缓存数据
- `plugin`：提供 OpenStack、GCP 和 Azure 等云服务商授权插件
- `rest`：提供 `RESTClient` 客户端，对 Kubernetes API Server 执行 RESTful 操作
- `scale`：提供 `ScaleClient` 客户端，用于扩容或缩容 Deployment、ReplicaSet、Relication Controller 等资源对象
- `tools`：提供常用工具，例如 SharedInformer、Reflector、DealtFIFO 及 Indexers。提供 Client 查询和缓存机制，以减少向 kube-apiserver 发起的请求数等
- `transport`：提供安全的 TCP 连接，支持 Http Stream，某些操作需要在客户端和容器之间传输二进制流，例如 exec、attach 等操作。该功能由内部的 spdy 包提供支持
- `util`：提供常用方法，例如 WorkQueue 功能队列、Certificate 证书管理等

## 三 认证

这里值得一提的是go mod通过go client 对接k8s的时候有个小坑
那么就是我们需要在go mod文件中指定k8s版本 不然会默认拉去最新的k8s版本的包 我们也知道不同的k8s版本的api会出现差异	

### 3.1 认证

#### 3.1.1 token

```shell
# APISERVER=$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')
# TOKEN=$(kubectl get secret $(kubectl get serviceaccount default -o jsonpath='{.secrets[0].name}') -o # jsonpath='{.data.token}' | base64 --decode )
# curl $APISERVER/api --header "Authorization: Bearer $TOKEN" --insecure
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/685a0c71d2c34786ab9dec06f95c2072~tplv-k3u1fbpfcp-zoom-1.image)

## 四 client-go四种类型

### 4.1 RestClient

RestClient是最基础的客户端，RestClient基于http request进行了封装，实现了restful的api，可以直接通过 是RESTClient提供的RESTful方法如Get()，Put()，Post()，Delete()进行交互,同时支持Json 和 protobuf,支持所有原生资源和CRDs，但是，一般而言，为了更为优雅的处理，需要进一步封装，通过Clientset封装RESTClient，然后再对外提供接口和服务。

```go
package main

import (
	"fmt"
	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes/scheme"
	"k8s.io/client-go/rest"
	"k8s.io/client-go/tools/clientcmd"
)

var configFile = "../config"
var ApiPath = "api"
var nameSpace = "kube-system"
var resouce = "pods"

func main() {
	// 生成config
	config, err := clientcmd.BuildConfigFromFlags("", configFile)
	if err != nil {
		panic(err)
	}
	config.APIPath = ApiPath
	config.GroupVersion = &corev1.SchemeGroupVersion
	config.NegotiatedSerializer = scheme.Codecs

	// 生成restClient
	restClient, err := rest.RESTClientFor(config)
	if err != nil {
		panic(err)
	}
	// 声明空结构体
	rest := &corev1.PodList{}
	if err = restClient.Get().Namespace(nameSpace).Resource("pods").VersionedParams(&metav1.ListOptions{Limit: 500},
		scheme.ParameterCodec).Do().Into(rest); err != nil {
		panic(err)
	}
	for _, v := range rest.Items {
		fmt.Printf("NameSpace: %v  Name: %v  Status: %v \n", v.Namespace, v.Name, v.Status.Phase)
	}
}
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/994b7f372d1a4381ab66a1edcb369784~tplv-k3u1fbpfcp-zoom-1.image)

### 4.2 ClientSet

ClientSet在RestClient的基础上封装了对Resouorce和Version的管理方法一个Resource可以理解为一个客户端，而ClientSet是多个客户端的集合

其操作资源对象时需要指定Group、指定Version，然后根据Resource获取，但是clientset不支持自定义crd。

```go
package main

import (
	"flag"
	"fmt"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/client-go/util/homedir"
	"path/filepath"
)

// ~/.kube/config
func ParseConfig(configPath string) (*kubernetes.Clientset, error) {
	var kubeconfigPath *string
	if home := homedir.HomeDir(); home != "" {
		kubeconfigPath = flag.String("kubeconfig", filepath.Join(home, ".kube", "config"), "(optional) absolute path to the kubeconfig file")
	} else {
		kubeconfigPath = flag.String("kubeconfig", configPath, "absolute path to the kubeconfig file")
	}
	flag.Parse()
	config, err := clientcmd.BuildConfigFromFlags("", *kubeconfigPath)
	if err != nil {
		return nil, err
	}
	// 生成clientSet
	clientSet, err := kubernetes.NewForConfig(config)
	if err != nil {
		return clientSet, err
	}
	return clientSet, nil
}

func ListCm(c *kubernetes.Clientset, ns string) error {
	configMaps, err := c.CoreV1().ConfigMaps(ns).List(metav1.ListOptions{})
	if err != nil {
		return err
	}
	for _, cm := range configMaps.Items {
		fmt.Printf("configName: %v, configData: %v \n", cm.Name, cm.Data)
	}
	return nil
}

func ListNodes(c *kubernetes.Clientset) error {
	nodeList, err := c.CoreV1().Nodes().List(metav1.ListOptions{})
	if err != nil {
		return err
	}
	for _, node := range nodeList.Items {
		fmt.Printf("nodeName: %v, status: %v", node.GetName(), node.GetCreationTimestamp())
	}
	return nil
}

func ListPods(c *kubernetes.Clientset, ns string) {
	pods, err := c.CoreV1().Pods(ns).List(metav1.ListOptions{})
	if err != nil {
		panic(err)
	}
	for _, v := range pods.Items {
		fmt.Printf("namespace: %v podname: %v podstatus: %v \n", v.Namespace, v.Name, v.Status.Phase)
	}
}

func ListDeployment(c *kubernetes.Clientset, ns string) error {
	deployments, err := c.AppsV1().Deployments(ns).List(metav1.ListOptions{})
	if err != nil {
		return err
	}
	for _, v := range deployments.Items {
		fmt.Printf("deploymentname: %v, available: %v, ready: %v", v.GetName(), v.Status.AvailableReplicas, v.Status.ReadyReplicas)
	}
	return nil
}

func main() {
	var namespace = "kube-system"
	configPath := "../config"
	config, err := ParseConfig(configPath)
	if err != nil {
		fmt.Printf("load config error: %v\n", err)
	}
	fmt.Println("list pods")
	ListPods(config, namespace)
	fmt.Println("list cm")
	if err = ListCm(config, namespace); err != nil {
		fmt.Printf("list cm error: %v", err)
	}
	fmt.Println("list nodes")
	if err = ListNodes(config); err != nil {
		fmt.Printf("list nodes error: %v", err)
	}
	fmt.Println("list deployment")
	if err = ListDeployment(config, namespace); err != nil {
		fmt.Printf("list deployment error: %v", err)
	}
}
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8b900341d56a496784d6dd9576180d6e~tplv-k3u1fbpfcp-zoom-1.image)

### 4.3 DynamicClient

DynamicClient是一种动态客户端它可以对任何资源进行restful操作包括crd自定义资源，不同于 clientset，dynamic client 返回的对象是一个 map[string]interface{}，如果一个 controller 中需要控制所有的 API，可以使用dynamic client，目前它在 garbage collector 和 namespace controller中被使用。
DynamicClient的处理过程将Resource例如podlist转换为unstructured类型，k8s的所有resource都可以转换为这个结构类型，处理完之后在转换为podlist，整个转换过程类似于接口转换就是通过interface{}的断言。

Dynamic client 是一种动态的 client，它能处理 kubernetes 所有的资源，只支持JSON。

```go
package main

import (
	"fmt"
	apiv1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/apimachinery/pkg/runtime/schema"
	"k8s.io/client-go/dynamic"
	"k8s.io/client-go/tools/clientcmd"
)

var namespace = "kube-system"

func main() {

	config, err := clientcmd.BuildConfigFromFlags("", "../config")
	if err != nil {
		panic(err)
	}

	dynamicClient, err := dynamic.NewForConfig(config)
	if err != nil {
		panic(err)
	}
	// 定义组版本资源
	gvr := schema.GroupVersionResource{Version: "v1", Resource: "pods"}
	unStructObj, err := dynamicClient.Resource(gvr).Namespace(namespace).List(metav1.ListOptions{})
	if err != nil {
		panic(err)
	}
	podList := &apiv1.PodList{}

	if err = runtime.DefaultUnstructuredConverter.FromUnstructured(unStructObj.UnstructuredContent(), podList); err != nil {
		panic(err)
	}

	for _, v := range podList.Items {
		fmt.Printf("namespaces:%v  name:%v status:%v \n", v.Namespace, v.Name, v.Status.Phase)
	}
}
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/af0748ccd38f4051ac8913ace79295cf~tplv-k3u1fbpfcp-zoom-1.image)

### 4.4 DiscoveryClient

DiscoveryClient是发现客户端，主要用于发现api server支持的资源组 资源版本 资源信息，k8s api server 支持很多资源组 资源版本，资源信息，此时可以通过DiscoveryClient来查看

kubectl的api-version和api-resource也是通过DiscoveryClient来实现的，还可以将信息缓存在本地cache，以减轻api的访问压力，默认在./kube/cache和./kube/http-cache下。

```go
package main

import (
	"fmt"
	"k8s.io/apimachinery/pkg/runtime/schema"
	"k8s.io/client-go/discovery"
	"k8s.io/client-go/tools/clientcmd"
)

func main() {
	config, err := clientcmd.BuildConfigFromFlags("", "../config")
	if err != nil {
		panic(err)
	}
	discoverClient, err := discovery.NewDiscoveryClientForConfig(config)
	if err != nil {
		panic(err)
	}
	_, apiResourceList, err := discoverClient.ServerGroupsAndResources()
	for _, v := range apiResourceList {
		gv, err := schema.ParseGroupVersion(v.GroupVersion)
		if err != nil {
			panic(err)
		}
		for _, resource := range v.APIResources {
			fmt.Println("name:", resource.Name, "    ", "group:", gv.Group, "    ", "version:", gv.Version)
		}
	}

}

```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2ae077286c584204b840b4d01c8cf37a~tplv-k3u1fbpfcp-zoom-1.image)

## 五 其他

学习client-go，可以非常方便的利用其对k8s集群资源进行操作，kubeconfig→rest.config→clientset→具体的client(CoreV1Client)→具体的资源对象(pod)→RESTClient→http.Client→HTTP请求的发送及响应

通过clientset中不同的client和client中不同资源对象的方法实现对kubernetes中资源对象的增删改查等操作，常用的client有CoreV1Client、AppsV1beta1Client、ExtensionsV1beta1Client等。

本篇为简单利用client-go 实现简单的k8s资源操作，后期利用例如 kubebuilder 和 operator-SDK 编写operator也需要深入的理解和学习client-go。后期继续继续深入学习。

## 参考链接

* https://www.voidking.com/dev-k8s-client-go/
* https://godoc.org/k8s.io/client-go/kubernetes
* https://github.com/kubernetes/client-go
* https://pkg.go.dev/k8s.io/client-go/kubernetes
* https://zhuanlan.zhihu.com/p/202611841?utm_source=wechat_session