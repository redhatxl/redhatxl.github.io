# x-Informer实战之持久化K8s事件至ElasticSearch

## 一 前言

在系列文章中详细讲解了Informer的相关知识，本届番外获取K8s的事件，将其存储到Elasticsearch，可以利用inforrmer机制回去到应用的时间进行外部持久化存储，或者进行过滤分类展示，或进行数据应用分析告警等。

## 二 ES部署

为了测试简单，采用Docker启动es。

```shell
docker run --name es01 -d -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" -e  "ES_JAVA_OPTS=-Xms512m -Xmx512m" elasticsearch:latest

# 使用head客户端链接
docker pull mobz/elasticsearch-head:5
# 启动header 容器
docker run -d --name my-es_admin -p 9100:9100 mobz/elasticsearch-head:5


```

启动后，es正常，为了更方便操作es，使用head插件来，插件链接异常，需要修改es配置，并重启容器

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210927142029.png)

```shell
# 进入容器
$ docker exec -it es01 /bin/bash

# 进入容器后设置参数
# http.cors.enabled: true
# http.cors.allow-origin: "*"
echo 'http.cors.enabled: true' >> config/elasticsearch.yml
echo 'http.cors.allow-origin: "*"' >> config/elasticsearch.yml
# 设置完成，退出后重启容器
docker restart es01
```

修改配置重启后，可以已经可以通过head组件正常链接es集群

![image-20210927143317543](/Users/xuel/Library/Application Support/typora-user-images/image-20210927143317543.png)

创建索引：

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210927144438.png)

## 三 代码

```go
var client *elastic.Client
var host = "http://127.0.0.1:9200/"

//初始化
func init() {
	errorlog := log.New(os.Stdout, "APP", log.LstdFlags)
	var err error
	// 这个地方有个小坑 不加上elastic.SetSniff(false) 会连接不上
	client, err = elastic.NewClient(elastic.SetSniff(false), elastic.SetErrorLog(errorlog), elastic.SetURL(host))
	if err != nil {
		panic(err)
	}
	info, code, err := client.Ping(host).Do(context.Background())
	if err != nil {
		panic(err)
	}
	fmt.Printf("Elasticsearch returned with code %d and version %s\n", code, info.Version.Number)

	esversion, err := client.ElasticsearchVersion(host)
	if err != nil {
		panic(err)
	}
	fmt.Printf("Elasticsearch version %s\n", esversion)

}

func Must(err error) {
	if err != nil {
		panic(err)
	}
}

func main() {
	rand.Seed(time.Now().UnixNano())
	config, err := clientcmd.BuildConfigFromFlags("", "/Users/xuel/.kube/config")
	Must(err)

	clientset, err := kubernetes.NewForConfig(config)
	Must(err)
	sharedInformers := informers.NewSharedInformerFactory(clientset, 0)
	stopChan := make(chan struct{})
	defer close(stopChan)

  // 在此使用event informer，
	eventInformer := sharedInformers.Events().V1beta1().Events().Informer()
	addChan := make(chan v1beta1.Event)
	deleteChan := make(chan v1beta1.Event)
	eventInformer.AddEventHandlerWithResyncPeriod(cache.ResourceEventHandlerFuncs{
		AddFunc: func(obj interface{}) {
			unstructObj, err := runtime.DefaultUnstructuredConverter.ToUnstructured(obj)
			Must(err)
			event := &v1beta1.Event{}
			err = runtime.DefaultUnstructuredConverter.FromUnstructured(unstructObj, event)
			Must(err)
			addChan <- *event
		},
		UpdateFunc: func(oldObj, newObj interface{}) {
		},
		DeleteFunc: func(obj interface{}) {
		},
	}, 0)

	go func() {
		for {
			select {
			case event := <-addChan:
				str, err := json.Marshal(&event)
				Must(err)
				fmt.Printf("插入k8s事件内容：%s", string(str))
				esinsert(str)
				break
			case <-deleteChan:
				break
			}
		}
	}()
	eventInformer.Run(stopChan)
}

func esinsert(str []byte) {
	index := "k8s_informer"
	dbtype := "doc"

	put1, err := client.Index().
		Index(index).
		Type(dbtype).
		Id("1").BodyString(string(str)).
		Do(context.Background())
	if err != nil {
		fmt.Println("Insert es error: %s", err)
	}
	fmt.Println("insert success", put1)
}

```

## 四 测试

插入数据后使用es查询：

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210927165517.png)

```shell
curl  -H "Content-Type: application/json" -XGET 'http://127.0.0.1:9200/k8s_informer/doc/_search?pretty' -d '{"query":{"match_all":{}}}'
```



![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210927165304.png)

触发k8s事件，会自动记录下来

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210927165844.png)

## 其他 

可以利用inforrmer机制回去到应用的时间进行外部持久化存储，或者进行过滤分类进行图像话展示，或进行数据应用分析告警等。

在本示例中仅仅使用了event 事件，当然你也可以使用其他事件，且仅关注了addfunc，你也可以关注update/delete等操作。