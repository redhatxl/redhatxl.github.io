## ElasticSearch



## 一 基础概念

Elaticsearch，简称为 ES， ES 是一个**开源的高扩展的分布式全文搜索引擎**， 是整个 ElasticStack 技术栈的核心。



## 二 环境部署

### 2.1 单节点

```shell
# 拉取镜像
docker pull elasticsearch:latest
# 启动
docker run --name es01 -d -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" -e  "ES_JAVA_OPTS=-Xms1g -Xmx1g" elasticsearch:latest

# 使用head客户端链接
docker pull mobz/elasticsearch-head:5
# 启动header 容器
docker run -d --name my-es_admin -p 9100:9100 mobz/elasticsearch-head:5

# curl测试访问
curl http://127.0.0.1:9200/
```

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210916150444.png)

第一次打开浏览器header访问，连接的服务地址是localhost:9200，修改为docker所在的ip。此时出现连接失败，需要修改镜像的elasticsearch.yml文件，添加

```
http.cors.enabled: true
http.cors.allow-origin: "*"

# 重启es
docker restart es01
docker restart my-es-head
```

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210916151130.png)

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211001203642.png)

### 2.2 集群

```shell
# 拉取镜像
docker pull elasticsearch:latest
# 启动
docker run --name es01 -d -p 9200:9200 -p 9300:9300 --restart=always  -e  "ES_JAVA_OPTS=-Xms1g -Xmx1g" elasticsearch:7.4.0

docker run --name es02 -d -p 9201:9201 -p 9301:9301 --restart=always  -e  "ES_JAVA_OPTS=-Xms1g -Xmx1g" elasticsearch:7.4.0

# 使用head客户端链接
docker pull mobz/elasticsearch-head:5
# 启动header 容器
docker run -d --name my-es_admin -p 9100:9100 mobz/elasticsearch-head:5

# curl测试访问
curl http://127.0.0.1:9200/
```



## 三 测试

### 3.1 创建

* 创建一个索引(PUT)

```shell
http://127.0.0.1:9200/shopping
```

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220325213030.png)

查看服务日志

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220325213131.png)

### 3.2 查看索引

* 查看所有索引

```shell
http://127.0.0.1:9200/_cat/indices?v
```

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220325213427.png)

|  表头   | 含义 |
|  ----  | ----  |
| health | green，yellow，red |
| status | open/close |
| index | 索引名称 |
| uuid | 索引统一编号 |
| pri | 主分片数量 |
| rep | 副本数量 |
| docs.count | 可用文档数量 |
| docs.deleted | 文档删除状态(逻辑删除) |
| store.size | 主分片和副本分片整体占用空间大小 |
| pri.store.size | 主要分片占用大小 |

* 查看单个索引

```shelll
http://127.0.0.1:9200/shopping
```

### 3.4 文档操作

向 ES 服务器发 POST 请求 ： http://127.0.0.1:9200/shopping/_doc/1，请求体JSON内容为：

```shell
{
    "title":"小米手机",
    "category":"小米",
    "images":"http://www.gulixueyuan.com/xm.jpg",
    "price":3999.00
}
```

* 查看索引下所有内容

向 ES 服务器发 GET 请求 ： http://127.0.0.1:9200/shopping/_search。

* 局部修改

局部修改
修改数据时，也可以只修改某一给条数据的局部信息

在 Postman 中，向 ES 服务器发 POST 请求 ： http://127.0.0.1:9200/shopping/_update/1。

请求体JSON内容为:

```shell
{
	"doc": {
		"title":"小米手机",
		"category":"小米"
	}
}
```

* 删除

在 Postman 中，向 ES 服务器发 DELETE 请求 ： http://127.0.0.1:9200/shopping/_doc/1

### 3.5 查询

* URL带参查询

 http://127.0.0.1:9200/shopping/_search?q=category:小米

* 请求体带参查询

接下带JSON请求体，还是**查找category为小米的文档**，在 Postman 中，向 ES 服务器发 GET请求 ： http://127.0.0.1:9200/shopping/_search

```shelll
{
	"query":{
		"match":{
			"category":"小米"
		}
	}
}
```

* 制定字段查询

如果你想查询指定字段，在 Postman 中，向 ES 服务器发 GET请求 ： http://127.0.0.1:9200/shopping/_search，附带JSON体如下：

```shelll
{
	"query":{
		"match_all":{}
	},
	"_source":["title"]
}
```

* 分页查询

在 Postman 中，向 ES 服务器发 GET请求 ： http://127.0.0.1:9200/shopping/_search，附带JSON体如下：

```json
{
	"query":{
		"match_all":{}
	},
	"from":0,
	"size":2
}
```

* 查询排序

如果你想通过排序查出价格最高的手机，在 Postman 中，向 ES 服务器发 GET请求 ： http://127.0.0.1:9200/shopping/_search，附带JSON体如下：

```shell
{
	"query":{
		"match_all":{}
	},
	"sort":{
		"price":{
			"order":"desc"
		}
	}
}

```

## 四 集群信息

### 单机 & 集群

单台 Elasticsearch 服务器提供服务，往往都有最大的负载能力，超过这个阈值，服务器
性能就会大大降低甚至不可用，所以生产环境中，一般都是运行在指定服务器集群中。
除了负载能力，单点服务器也存在其他问题：

单台机器存储容量有限
单服务器容易出现单点故障，无法实现高可用
单服务的并发处理能力有限

### 集群 Cluster

一个集群就是由一个或多个服务器节点组织在一起，共同持有整个的数据，并一起提供索引和搜索功能。**一个 Elasticsearch 集群有一个唯一的名字标识，这个名字默认就是”elasticsearch”。这个名字是重要的，因为一个节点只能通过指定某个集群的名字，来加入这个集群。

### 节点 Node

集群中包含很多服务器， 一个节点就是其中的一个服务器。 作为集群的一部分，它存储数据，参与集群的索引和搜索功能。

一个节点也是由一个名字来标识的，默认情况下，这个名字是一个随机的漫威漫画角色的名字，这个名字会在启动的时候赋予节点。这个名字对于管理工作来说挺重要的，因为在这个管理过程中，你会去确定网络中的哪些服务器对应于 Elasticsearch 集群中的哪些节点。

一个节点可以通过配置集群名称的方式来加入一个指定的集群。默认情况下，每个节点都会被安排加入到一个叫做“elasticsearch”的集群中，这意味着，如果你在你的网络中启动了若干个节点，并假定它们能够相互发现彼此，它们将会自动地形成并加入到一个叫做“elasticsearch”的集群中。

在一个集群里，只要你想，可以拥有任意多个节点。而且，如果当前你的网络中没有运
行任何 Elasticsearch 节点，这时启动一个节点，会默认创建并加入一个叫做“elasticsearch”的
集群。

### 分片Shards

一个索引可以存储超出单个节点硬件限制的大量数据。比如，一个具有 10 亿文档数据
的索引占据 1TB 的磁盘空间，而任一节点都可能没有这样大的磁盘空间。 或者单个节点处理搜索请求，响应太慢。为了解决这个问题，**Elasticsearch 提供了将索引划分成多份的能力，每一份就称之为分片。**当你创建一个索引的时候，你可以指定你想要的分片的数量。每个分片本身也是一个功能完善并且独立的“索引”，这个“索引”可以被放置到集群中的任何节点上。

分片很重要，主要有两方面的原因：

允许你水平分割 / 扩展你的内容容量。
允许你在分片之上进行分布式的、并行的操作，进而提高性能/吞吐量。
至于一个分片怎样分布，它的文档怎样聚合和搜索请求，是完全由 Elasticsearch 管理的，对于作为用户的你来说，这些都是透明的，无需过分关心。

被混淆的概念是，一个 Lucene 索引 我们在 Elasticsearch 称作 分片 。 一个Elasticsearch 索引 是分片的集合。 当 Elasticsearch 在索引中搜索的时候， 他发送查询到每一个属于索引的分片（Lucene 索引），然后合并每个分片的结果到一个全局的结果集。

Lucene 是 Apache 软件基金会 Jakarta 项目组的一个子项目，提供了一个简单却强大的应用程式接口，能够做全文索引和搜寻。在 Java 开发环境里 Lucene 是一个成熟的免费开源工具。就其本身而言， Lucene 是当前以及最近几年最受欢迎的免费 Java 信息检索程序库。但 Lucene 只是一个提供全文搜索功能类库的核心工具包，而真正使用它还需要一个完善的服务框架搭建起来进行应用。

目前市面上流行的搜索引擎软件，主流的就两款： Elasticsearch 和 Solr,这两款都是基于 Lucene 搭建的，可以独立部署启动的搜索引擎服务软件。由于内核相同，所以两者除了服务器安装、部署、管理、集群以外，对于数据的操作 修改、添加、保存、查询等等都十分类似。


### 副本Replicas

在一个网络 / 云的环境里，失败随时都可能发生，在某个分片/节点不知怎么的就处于
离线状态，或者由于任何原因消失了，这种情况下，有一个故障转移机制是非常有用并且是强烈推荐的。为此目的， Elasticsearch 允许你创建分片的一份或多份拷贝，这些拷贝叫做复制分片(副本)。

复制分片之所以重要，有两个主要原因：

在分片/节点失败的情况下，提供了高可用性。因为这个原因，注意到复制分片从不与原/主要（original/primary）分片置于同一节点上是非常重要的。
扩展你的搜索量/吞吐量，因为搜索可以在所有的副本上并行运行。
总之，每个索引可以被分成多个分片。一个索引也可以被复制 0 次（意思是没有复制）或多次。一旦复制了，每个索引就有了主分片（作为复制源的原来的分片）和复制分片（主分片的拷贝）之别。

分片和复制的数量可以在索引创建的时候指定。在索引创建之后，你可以在任何时候动态地改变复制的数量，但是你事后不能改变分片的数量。

默认情况下，Elasticsearch 中的每个索引被分片 1 个主分片和 1 个复制，这意味着，如果你的集群中至少有两个节点，你的索引将会有 1 个主分片和另外 1 个复制分片（1 个完全拷贝），这样的话每个索引总共就有 2 个分片， 我们需要根据索引需要确定分片个数。


### 分配Allocation

将分片分配给某个节点的过程，包括分配主分片或者副本。如果是副本，还包含从主分片复制数据的过程。这个过程是由 master 节点完成的。

## 五 系统架构-简介

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220329132818.png)

一个运行中的 Elasticsearch 实例称为一个节点，而集群是由一个或者多个拥有相同
cluster.name 配置的节点组成， 它们共同承担数据和负载的压力。当有节点加入集群中或者从集群中移除节点时，集群将会重新平均分布所有的数据。

当一个节点被选举成为主节点时， 它将负责管理集群范围内的所有变更，例如增加、
删除索引，或者增加、删除节点等。 而主节点并不需要涉及到文档级别的变更和搜索等操作，所以当集群只拥有一个主节点的情况下，即使流量的增加它也不会成为瓶颈。 任何节点都可以成为主节点。我们的示例集群就只有一个节点，所以它同时也成为了主节点。

作为用户，我们可以将请求发送到集群中的任何节点 ，包括主节点。 每个节点都知道
任意文档所处的位置，并且能够将我们的请求直接转发到存储我们所需文档的节点。 无论我们将请求发送到哪个节点，它都能负责从各个包含我们所需文档的节点收集回数据，并将最终结果返回給客户端。 Elasticsearch 对这一切的管理都是透明的。





##参考链接

* https://blog.csdn.net/u011863024/article/details/115721328