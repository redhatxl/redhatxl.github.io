# vitess

# 一 概述

## 1.1 vitess是什么

Vitess是一个用于部署、扩展和管理大型MySQL实例集群的数据库解决方案。Vitess集Mysql数据库的很多重要特性和NoSQL数据库的可扩展性于一体。它的架构设计使得您可以像在物理机上一样在公共云或私有云架构中有效运行。它结合并扩展了许多重要的MySQL功能，同时兼具NoSQL数据库的可扩展性。 Vitess可以帮助您解决以下问题：

1. 支持您对MySQL数据库进行分片来扩展MySQL数据库，应用程序无需做太多更改。
2. 从物理机迁移到私有云或公共云。
3. 部署和管理大量的MySQL实例。

Vitess包括使用与本机查询协议兼容的JDBC和Go数据库驱动。此外，它还实现了mysql服务器协议，该协议几乎与任何其他语言都兼容。

自2011年以来，Vitess一直为YouTube所有的数据库提供服务，现在已被许多企业采用并应用于实际生产。

## 1.2 特性

### 1.2.1 性能提升

- 连接池 - 将前端应用程序以多路复用的方式映射到MySQL连接池以优化性能。
- 查询结果重用 – 对于相同结果集的查询，多个查询并发查询时，vttablet会识别和管理相同查询，等待第一个查询结果完成，并发送给所有的调用者。
- 事务管理 – 限制并发事务数、管理事务超时时间以优化总体吞吐量。

### 1.2.2 保护机制

- 查询重写和清理 – 避免漫无目的的更新，对大查询添加limits。
- 查询黑名单 – 可通过自定义规则以防止可能存在问题的查询命中数据库。
- 查询超时 – 可自定义查询超时时间值，Vitess将干掉超时的查询。
- 表别访问权限控制定义 – 可以针对不同的接入用户指定表的访问控制权限 (ACLs)。

### 1.2.3 监控

- 性能分析: Vitess提供工具可让您监控，诊断和分析数据库性能。
- 流式查询 – 使用传入查询列表来提供OLAP工作。
- 更新流 – 服务器流式传输数据库中更改的行列表，可用作将更改传播到其他数据存储的机制。

### 1.2.4 拓扑管理工具

- Master管理工具（用于reparent处理）
- 基于Web GUI的管理端
- 可工作于多个数据中心/区域的设计

### 1.2.5 拆分

- 几乎无缝的动态分片拆分
- 支持垂直和水平分片拆分
- 多种分片方案，支持自定义分片方案



## 1.3 与其他数据库对比

以下部分将Vitess与两种常见的替代方案进行比较，即vanilla MySQL实现和NoSQL实现。

### 1.3.1 Vitess vs. Vanilla MySQL

| Vanilla MySQL                                                | Vitess                                                       |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| 每个MySQL连接的内存开销介于256KB和3MB之间，具体取决于您使用的MySQL版本。随着用户群的增长，您需要添加RAM以支持其他连接，但RAM不会有助于更快的查询。此外，获得连接时的CPU成本也很高。 | Vitess创建非常轻量级的连接。 Vitess的连接池功能使用Go语言的并发特性，将这些轻量级连接映射到一小部分MySQL连接中。因此，Vitess可以轻松处理数千个连接。 |
| 写的很烂的查询（例如未设置LIMIT的查询）会对拖慢数据库影响所有用户。 | Vitess实现了SQL解析器，并使用一组可配置的规则来重写可能会损害数据库性能的查询。 |
| 分片是对数据进行分区以提高可伸缩性和性能的手段。 MySQL本身不具备分片功能，要求您编写分片代码并在应用程序中嵌入分片逻辑。 | Vitess支持各种分片方案。它还可以将表迁移到不同的数据库中，并可以扩容或缩容扩展分片数。这些功能皆是非侵入式执行的，只需几秒钟的只读停机时间即可完成大多数数据转换。 |
| 使用复制实现高可用的MySQL集群具有主数据库和一些副本。如果主服务器出现故障，则副本应成为新主服务器。这要求您自己管理数据库生命周期并将当前系统状态传递给您的应用程序。 | Vitess有助于管理数据库方案的生命周期。它支持并自动处理各种方案，包括主故障迁移和数据备份。 |
| MySQL集群可以为不同的工作负载提供自定义数据库配置，例如用于写入的主数据库、用于Web客户端的快速只读副本、用于批处理作业的较慢只读副本等等。如果数据库具有水平分片，则需要为每个分片重复设置，并且应用需要在代码中加入逻辑以便找到正确的数据库。 | Vitess使用分布式一致性键值存储拓扑服务，如etcd或ZooKeeper。这意味着群集元数据始终是最新的，并且对于不同的客户端是一致的。 Vitess还提供了一个代理，可以有效地将查询路由到最合适的MySQL实例。 |

### 1.3.2 Vitess vs. NoSQL

| NoSQL                                                        | Vitess                                                       |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| NoSQL数据库没有传统数据库中库和表的概念，仅支持SQL语言的子集。 | Vitess不是简单的键值存储。它支持复杂的查询语义，例如where子句，JOINS，聚合函数等。 |
| NoSQL数据库不支持事务                                        |                                                              |
| NoSQL解决方案具有自定义API，可实现自定义结构，应用程序和工具。 | Vitess与MySQL之间仅有很小的差异，MySQL是一个大多数人已经习惯使用的数据库。 |
| NoSQL解决方案具有自定义API，可实现自定义结构，应用程序和工具。 | Vitess允许您使用MySQL的所有索引功能来优化查询性能。          |

## 1.4 架构

Vitess平台由许多服务器进程、命令行实用程序和基于Web的实用程序组成，由一致的元数据存储提供支持。

根据您当前的业务状态，您可以选择不同的方式最终实现vitess的完整部署。举例来说，如果是从头开始构建服务，那么使用Vitess的第一步就是定义数据库拓扑。如果是扩展现有的数据库， 那么可能需要先部署连接代理。

无论您是从一整套数据库开始，还是决定从小规模开始(今后再慢慢扩展)。Vitess工具和服务器都能贴心的帮助到您。对于较小规模的数据库，vttablet功能（如连接池和查询重写）可帮助您从现有硬件中榨取更多性能。对于大规模的数据库，Vitess提供的自动化工具在为更大规模的实施时给予更多的便利。

### 1.4.1 Vitess的组件架构图

![image-20191106112102734](/Users/xuel/Library/Application Support/typora-user-images/image-20191106112102734.png)

### 1.4.2 相关组件

* Topology

[拓扑服务](https://vitess.io/zh/docs/user-guides/topology-service) 一个元数据存储，包含有关正在运行的服务器、分片方案和复制图的信息。拓扑由一致的数据存储支持。您可以使用**vtctl** (命令行) 和 **vtctld** (web)查看拓扑.

在Kubernetes中，数据存储是etcd。 Vitess源代码还附带[Apache ZooKeeper](http://zookeeper.apache.org/)支持。

* vtgate

**vtgate** 是一个轻型代理服务器，它将流量路由到正确的vttablet，并将合并的结果返回给客户端。应用程序向vtgate发起查询。客户端使用起来非常简单，它只需要能够找到vtgate实例就能使vitess。

为了路由查询，vtgate综合考虑了分片方案、数据延迟以及vttablet及其对应底层MySQL实例的可用性。

* vttablet

**vttablet** 是一个位于MySQL数据库前面的代理服务器。vitess实现中每个MySQL实例都有一个vttablet。

执行的任务试图最大化吞吐量，同时保护mysql不受有害查询的影响。它的特性包括连接池、查询重写和重用重复数据。此外，vtTablet执行vtcl启动的管理任务，并提供用于过滤复制和数据导出的流式服务。

通过在MySQL数据库前运行vttablet并更改您的应用程序以使用Vitess客户端而不是MySQL驱动程序，您的应用程序将受益于vttablet的连接池，查询重写和重用数据集等功能。

* vtctl

vtctl是一个用于管理Vitess集群的命令行工具。它允许用户或应用程序轻松地与Vitess实现交互。使用vtctl，您可以识别主数据库和副本数据库，创建表，启动故障转移，执行分片（和重新分片）操作等。

当vtctl执行操作时，它会根据需要更lockserver。其他Vitess服务器会观察这些变化并做出相应的反应。例如，如果使用vtctl故障转移到新的主数据库，则vtgate会查看更改并将将写入流量切到新主服务器。

* vtctld

vtctld是一个HTTP服务器，允许您浏览存储在lockserver中的信息。它对于故障排除或获取服务器及其当前状态的高层概观非常有用。

* vtworker

**vtworker** 托管长时间运行的进程。它支持插件架构并提供代码库，以便您可以轻松选择要使用的vttablet。插件可用于以下类型的作业：

- 水平拆分或合并过程中检查数据的完整性
- 垂直拆分或合并过程中检查数据的完整性

vtworker还可以让您轻松添加其他验证程序。例如，如果一个keyspace中的索引表引用到另一keyspace中的数据，则可以执行片内完整性检查以验证类似外键的关系或跨分片完整性检查。

# 二 快速入门

## 2.1 Kubenetes上运行Vitess

### 2.1.1 场景

下面的示例将使用一个简单的commerce数据库来说明vitess如何将您从一个数据库扩展到一个完全分布式和分片的集群。这是一个相当常见的场景，它适用于电子商务之外的许多用例。*

随着公司的发展壮大，他们经常会面临这样一个问题：交易数据库中的数据量显著增加。数据大小急剧增加可能导致查询延迟、可管理性变差等影响性能的问题。本教程演示了Vitess如何与Kubernetes一起使用，通过利用分布式系统的水平分片来缓解系统在大规模数据下的数据查询性能问题。

### 2.1.2 先决条件

先决条件

- [下载 Vitess](https://github.com/vitessio/vitess)
- [安装 Minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/)
- 启动Minikube引擎：`minikube start --cpus = 4 --memory = 5000`。请注意其他资源要求。为了完成所有用例，将启动许多vttablet和MySQL实例。这些需要比Minikube使用的默认值更多的资源。
- [安装 etcd 管理端](https://github.com/coreos/etcd-operator/blob/master/doc/user/install_guide.md)
- [安装 helm](https://docs.helm.sh/using_helm/)
- 完成安装后, 运行 `helm init`

可选操作

- 在Ubuntu上安装Mysql客户端: `apt-get install mysql-client`
- 安装 vtctlclient
  - 安装 go 1.11+
  - `go get vitess.io/vitess/go/cmd/vtctlclient`
  - vtctlclient 将安装在目录 `$GOPATH/bin/`

### 2.1.3 启动单片keyspace集群

* 启动

你google了一通keyspace，却发现了很多关于NoSQL的东西。这TM是怎么回事？虽然这会花费你几个小时的时间，但是在仔细翻看vitess的古老卷轴之后，你会发现，原来在未分片(单片)的情形下，在NewSQL的世界中，keyspaces和databases本质上就是一类东西。最后，是时候开始我们的旅程了。

切换到helm示例目录:

```sh
cd examples/helm
```

在此目录中，您将看到一组yaml文件。每个文件名的第一个数字表示示例的阶段。接下来的两位数字表示执行它们的顺序。例如’101_initial_cluster.yaml’是第一阶段的第一个文件。我们现在将执行:

```sh
helm install ../../helm/vitess -f 101_initial_cluster.yaml
```

这将创建一个带有单个keyspace空间的初始Vitess集群。

* 验证集群

一旦成功，您应该看到以下状态：

```sh
~/...vitess/helm/vitess/templates> kubectl get pods,jobs
NAME                               READY     STATUS    RESTARTS   AGE
po/etcd-global-2cwwqfkf8d          1/1       Running   0          14m
po/etcd-operator-9db58db94-25crx   1/1       Running   0          15m
po/etcd-zone1-btv8p7pxsg           1/1       Running   0          14m
po/vtctld-55c47c8b6c-5v82t         1/1       Running   1          14m
po/vtgate-zone1-569f7b64b4-zkxgp   1/1       Running   2          14m
po/zone1-commerce-0-rdonly-0       6/6       Running   0          14m
po/zone1-commerce-0-replica-0      6/6       Running   0          14m
po/zone1-commerce-0-replica-1      6/6       Running   0          14m

NAME                                      DESIRED   SUCCESSFUL   AGE
jobs/commerce-apply-schema-initial        1         1            14m
jobs/commerce-apply-vschema-initial       1         1            14m
jobs/zone1-commerce-0-init-shard-master   1         1            14m
```

如果已安装MySQL客户端，则现在应该可以使用以下命令连接到群集：

```sh
~/...vitess/examples/helm> ./kmysql.sh
mysql> show tables;
+--------------------+
| Tables_in_commerce |
+--------------------+
| corder             |
| customer           |
| product            |
+--------------------+
3 rows in set (0.01 sec)
```

您还可以使用以下命令浏览vtctld控制台：（Ubuntu）

```sh
./kvtctld.sh
```



* 





# 参考链接

* [http://vitess.io](http://vitess.io/)
* https://github.com/vitessio/vitess
* https://vitess.io/zh/docs/
* https://vitess.io/zh/docs/get-started/kubernetes/