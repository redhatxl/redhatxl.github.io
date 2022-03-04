# Prometheus 完全笔记

# 一 概述

Prometheus 是一个开源监控系统，它前身是 [SoundCloud](http://soundcloud.com/)的告警工具包。从 2012 年开始，许多公司和组织开始使用 Prometheus。该项目的开发人员和用户社区非常活跃，越来越多的开发人员和用户参与到该项目中。目前它是一个独立的开源项目，且不依赖于任何公司。为了强调这点和明确该项目治理结构，Prometheus 在 2016 年继[Kurberntes](http://kubernetes.io/) 之后，加入了 [Cloud Native Computing Foundation](https://cncf.io/)。

## 1.1 prometheus核心概念

### 1.1.1 数据模型

Prometheus 从根本上存储的所有数据都是时间序列数据（Time Serie Data，简称时序数据）。时序数据是具有时间戳的数据流，该数据流属于某个度量指标（Metric）和该度量指标下的多个标签（Label）。除了提供存储功能，Prometheus 还可以利用查询表达式来执行非常灵活和复杂的查询。

* 度量指标和标签

每个时间序列（Time Serie，简称时序）由度量指标和一组标签键值对唯一确定。

度量指标名称描述了被监控系统的某个测量特征（比如 http_requests_total 表示 http 请求总数）。度量指标名称由 ASCII 字母、数字、下划线和冒号组成，须匹配正则表达式 `[a-zA-Z_:][a-zA-Z0-9_:]*`。

标签开启了 Prometheus 的多维数据模型。对于同一个度量指标，不同标签值组合会形成特定维度的时序。Prometheus 的查询语言可以通过度量指标和标签对时序数据进行过滤和聚合。改变任何度量指标上的任何标签值，都会形成新的时序。标签名称可以包含 ASCII 字母、数字和下划线，须匹配正则表达式 `[a-zA-Z_][a-zA-Z0-9_]*`，带有 _ 下划线的标签名称保留为内部使用。标签值可以包含任意 Unicode 字符，包括中文。

* 采样值（Sample）

  时序数据其实就是一系列采样值。每个采样值包括：

  * 一个 64 位的浮点数值
  * 一个精确到毫秒的时间戳

* 注解（Notation）

一个注解由一个度量指标和一组标签键值对构成。形式如下：

```
[metric name]{[label name]=[label value], ...}
```

例如，度量指标为 api_http_requests_total，标签为 method="POST"、handler="/messages" 的注解表示如下：

```
api_http_requests_total{method="POST", handler="/messages"}
```

### 1.1.2 度量指标类型

* 计数器（Counter）

计数器是一种累计型的度量指标，它是一个只能递增的数值。计数器主要用于统计类似于服务请求数、任务完成数和错误出现次数这样的数据。

* 计量器（Gauge）

计量器表示一个既可以增加, 又可以减少的度量指标值。计量器主要用于测量类似于温度、内存使用量这样的瞬时数据。

* 直方图（Histogram）

  直方图对观察结果（通常是请求持续时间或者响应大小这样的数据）进行采样，并在可配置的桶中对其进行统计。有以下几种方式来产生直方图（假设度量指标为 <basename>）：

  * 按桶计数，相当于 `<basename>_bucket{le="<upper inclusive bound>"}`
  * 采样值总和，相当于 `<basename>_sum`
  * 采样值总数，相当于 `<basename>_count` ，也等同于把所有采样值放到一个桶里来计数 `<basename>_bucket{le="+Inf"}`

* 汇总（Summary）

  类似于直方图，汇总也对观察结果进行采样。除了可以统计采样值总和和总数，它还能够按分位数统计。有以下几种方式来产生汇总（假设度量指标为 <basename>）：

  * 按分位数，也就是采样值小于该分位数的个数占总数的比例小于 φ，相当于 `<basename>{quantile="<φ>"}`
  * 采样值总和，相当于 `<basename>_sum`
  * 采样值总数，相当于 `<basename>_count`

### 1.1.3 任务（Job）和实例（Instance）

在 Prometheus 里，可以从中抓取采样值的端点称为实例，为了性能扩展而复制出来的多个这样的实例形成了一个任务。

例如下面的 api-server 任务有四个相同的实例：

```shell
job: api-server
instance 1: 1.2.3.4:5670
instance 2: 1.2.3.4:5671
instance 3: 5.6.7.8:5670
instance 4: 5.6.7.8:5671
```

Prometheus 抓取完采样值后，会自动给采样值添加下面的标签和值：

- job: 抓取所属任务。
- instance: 抓取来源实例

另外每次抓取时，Prometheus 还会自动在以下时序里插入采样值：

- `up{job="[job-name]", instance="instance-id"}`：采样值为 1 表示实例健康，否则为不健康
- `scrape_duration_seconds{job="[job-name]", instance="[instance-id]"}`：采样值为本次抓取消耗时间
- `scrape_samples_post_metric_relabeling{job="<job-name>", instance="<instance-id>"}`：采样值为重新打标签后的采样值个数
- `scrape_samples_scraped{job="<job-name>", instance="<instance-id>"}`：采样值为本次抓取到的采样值个数

## 1.2 prometheus特点

* 多维度数据模型，一个时间序列由一个度量指标和多个标签键值对确定
* 灵活的查询语言，对收集的时许数据进行重组
* 强大的数据可视化功能，除了内置的浏览器，也支持grafana集成
* 高效存储，内存加本地磁盘，可通过功能分片和联盟来拓展性能
* 运维简单，只依赖于本地磁盘，go二进制安装包没有任何其他依赖
* 精简告警
* 非常多的客户端库
* 提供了许多导出器来收集常用系统指标

## 1.3 altermanager 核心概念

### 1.3.1 分组

分组将类似性质的警报分类为单个通知。在许多系统一次性失败并且数百到数千个警报可能同时发生的较大中断期间，这尤其有用。

示例：发生网络分区时，群集中正在运行数十或数百个服务实例。一半的服务实例无法再访问数据库。Prometheus中的警报规则配置为在每个服务实例无法与数据库通信时发送警报。结果，数百个警报被发送到Alertmanager。

作为用户，人们只想获得单个页面，同时仍能够确切地看到哪些服务实例受到影响。因此，可以将Alertmanager配置为按群集和alertname对警报进行分组，以便发送单个紧凑通知。

通过配置文件中的路由树配置警报的分组，分组通知的定时以及这些通知的接收器。

### 1.3.2 抑制

如果某些其他警报已经触发，则抑制是抑制某些警报的通知的概念。示例：正在触发警报，通知无法访问整个集群。Alertmanager可以配置为在该特定警报触发时将与该集群有关的所有其他警报静音。这可以防止数百或数千个与实际问题无关的触发警报的通知。通过Alertmanager的配置文件配置禁止。

### 1.3.3 沉默

沉默是在给定时间内简单地静音警报的简单方法。基于匹配器配置静默，就像路由树一样。检查传入警报它们是否匹配活动静默的所有相等或正则表达式匹配器。如果他们这样做，则不会发送该警报的通知。在Alertmanager的Web界面中配置了静音。

### 1.3.4 客户端行为

Alertmanager对其客户的行为有特殊要求。这些仅适用于不使用Prometheus发送警报的高级用例。

### 1.3.5 高可用

Alertmanager支持配置以创建用于高可用性的集群。这可以使用--cluster- *标志进行配置。重要的是不要在Prometheus和它的Alertmanagers之间加载平衡流量，而是将Prometheus指向所有Alertmanagers的列表。

# 二 架构

## 2.1 prometheus架构图

![image-20190824212703954](/Users/xuel/Library/Application Support/typora-user-images/image-20190824212703954.png)

![image-20190825205430423](/Users/xuel/Library/Application Support/typora-user-images/image-20190825205430423.png)

## 2.2 altermanager架构图

![image-20190825104025896](/Users/xuel/Library/Application Support/typora-user-images/image-20190825104025896.png)

# 三 与其他监控系统对不

## 3.1 Prometheus vs. Zabbix

- [Zabbix](https://www.zabbix.com/) 使用的是 C 和 PHP, Prometheus 使用 Golang, 整体而言 Prometheus 运行速度更快一点。
- Zabbix 属于传统主机监控，主要用于物理主机、交换机、网络等监控，Prometheus 不仅适用主机监控，还适用于 Cloud、SaaS、Openstack、Container 监控。
- Zabbix 在传统主机监控方面，有更丰富的插件。
- Zabbix 可以在 WebGui 中配置很多事情，Prometheus 需要手动修改文件配置。、

## 3.2 Prometheus vs. Nagios

- [Nagios](https://www.nagios.org/) 数据不支持自定义 Labels, 不支持查询，告警也不支持去噪、分组, 没有数据存储，如果想查询历史状态，需要安装插件。
- Nagios 是上世纪 90 年代的监控系统，比较适合小集群或静态系统的监控Nagios 太古老，很多特性都没有，Prometheus 要优秀很多。

## 3.3 Prometheus vs Sensu

- [Sensu](https://sensuapp.org/) 广义上讲是 Nagios 的升级版本，它解决了很多 Nagios 的问题，如果你对 Nagios 很熟悉，使用 Sensu 是个不错的选择。
- Sensu 依赖 RabbitMQ 和 Redis，数据存储上扩展性更好。

## 3.4 Prometheus vs InfluxDB

- [InfluxDB](https://www.influxdata.com/) 是一个开源的时序数据库，主要用于存储数据，如果想搭建监控告警系统，需要依赖其他系统。
- InfluxDB 在存储水平扩展以及高可用方面做的更好, 毕竟核心是数据库。



# 四 安装部署

## 4.1 prometheus安装

* 二进制安装

```shell
cd /opt && wget https://github.com/prometheus/prometheus/releases/download/v2.12.0/prometheus-2.12.0.linux-amd64.tar.gz 
tar -zxf prometheus-2.12.0.linux-amd64.tar.gz
mv prometheus-2.12.0.linux-amd64 prometheus
chown root.root prometheus -R

# 配置为服务
cat >/usr/lib/systemd/system/prometheus.service <<EOF
[Unit]
Description=Prometheus
Documentation=https://prometheus.io/
After=network.target

[Service]
Type=simple
ExecStart=/opt/prometheus/prometheus --config.file=/opt/prometheus/prometheus.yml
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

# 设置服务开机自启动
systemctl enable prometheus
systemctl start prometheus

# 直接启动
nohup ./prometheus --config.file=prometheus.yml 2>&1 1>prometheus.log &

# 查看服务
[root@VM_0_13_centos pushgateway]# netstat -lntup |grep prometheus
tcp6       0      0 :::9090                 :::*                    LISTEN      16655/prometheus
```

* 源码编译安装

```shell
$ go get github.com/prometheus/prometheus/cmd/...
$ prometheus --config.file=your_config.yml


# 或者make build
$ mkdir -p $GOPATH/src/github.com/prometheus
$ cd $GOPATH/src/github.com/prometheus
$ git clone https://github.com/prometheus/prometheus.git
$ cd prometheus
$ make build
$ ./prometheus --config.file=your_config.yml
```



* docker安装

```shell
docker run --name prometheus -d -p 127.0.0.1:9090:9090 prom/prometheus
```





## 4.2 alertmanager安装

* 二进制安装

```shell
cd /opt && wget -c https://github.com/prometheus/alertmanager/releases/download/v0.18.0/alertmanager-0.18.0.linux-amd64.tar.gz
tar zxf alertmanager-0.18.0.linux-amd64.tar.gz
mv alertmanager-0.18.0.linux-amd64 alertmanager
chown root.root alertmanager -R

# 配置服务
cat >/usr/lib/systemd/system/alertmanager.service <<EOF
[Unit]
Description=Alertmanager
Documentation=https://prometheus.io/
After=network.target

[Service]
Type=simple
ExecStart=/opt/alertmanager/alertmanager --config.file=/opt/alertmanager/alertmanager.yml
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

# 设置服务开机自启动
systemctl enable alertmanager
systemctl start alertmanager

# 直接启动
nohup ./alertmanager --config.file=alertmanager.yml 2>&1 1>alertmanager.log &

# 查看服务
[root@VM_0_13_centos pushgateway]# netstat -lntup |grep alertmanager
tcp6       0      0 :::9094                 :::*                    LISTEN      17237/alertmanager
tcp6       0      0 :::9093                 :::*                    LISTEN      17237/alertmanager
udp6       0      0 :::9094                 :::*                                17237/alertmanager
```

* 编译安装

```shell
$ GO15VENDOREXPERIMENT=1 go get github.com/prometheus/alertmanager/cmd/...
# cd $GOPATH/src/github.com/prometheus/alertmanager
$ alertmanager --config.file=<your_file>

# 手动源码构建
$ mkdir -p $GOPATH/src/github.com/prometheus
$ cd $GOPATH/src/github.com/prometheus
$ git clone https://github.com/prometheus/alertmanager.git
$ cd alertmanager
$ make build
$ ./alertmanager --config.file=<your_file>

# amtool构建
$ make build BINARIES=amtool
```



* docker安装

```shell
docker pull quay.io/prometheus/alertmanager
```



## 4.3 node_export安装

利用node_export来监控主机，官方也提供了很多其他的export可以用来直接使用

* 二进制安装

```shell
cd /opt && wget -c https://github.com/prometheus/node_exporter/releases/download/v0.18.1/node_exporter-0.18.1.linux-amd64.tar.gz
tar zxf node_exporter-0.18.1.linux-amd64.tar.gz
mv node_exporter-0.18.1.linux-amd64 node_exporter
chown root.root node_exporter -R

# 配置服务
cat >/usr/lib/systemd/system/node_exporter.service <<EOF
[Unit]
Description=node_exporter
Documentation=https://prometheus.io/
After=network.target

[Service]
Type=simple
ExecStart=/opt/node_exporter/node_exporter
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

# 设置服务开机自启动
systemctl enable node_exporter
systemctl start node_exporter

# 直接启动
nohup ./node_exporter --config.file=node_exporter.yml 2>&1 1>node_exporter.log &

# 查看服务
[root@VM_0_13_centos pushgateway]# netstat -lntup |grep node_export
tcp6       0      0 :::9100                 :::*                    LISTEN      4551/node_exporter
```

## 4.4 pushgateway

* 安装

```shell
cd /opt && wget -c https://github.com/prometheus/pushgateway/releases/download/v0.9.1/pushgateway-0.9.1.linux-amd64.tar.gz
tar zxf pushgateway-0.9.1.linux-amd64.tar.gz
mv pushgateway-0.9.1.linux-amd64 pushgateway
chown root.root pushgateway -R

# 配置服务
cat >/usr/lib/systemd/system/pushgateway.service <<EOF
[Unit]
Description=pushgateway
Documentation=https://prometheus.io/
After=network.target

[Service]
Type=simple
ExecStart=/opt/pushgateway/pushgateway
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

# 设置服务开机自启动
systemctl enable pushgateway
systemctl start pushgateway

# 直接启动
nohup ./pushgateway --config.file=node_exporter.yml 2>&1 1>node_exporter.log &

# 查看服务
[root@VM_0_13_centos pushgateway]# netstat -lntup |grep push
tcp6       0      0 :::9091                 :::*                    LISTEN      5982/pushgateway
```

* 查看web页面

![image-20190826182701121](/Users/xuel/Library/Application Support/typora-user-images/image-20190826182701121.png)

* shell命令创建

```shell
echo "some_metric 3.14" | curl --data-binary @- http://localhost:9091/metrics/job/some_job
```

* 发送复杂数据

```shell
cat <<EOF | curl --data-binary @- http://localhost:9091/metrics/job/some_job/instance/some_instance
# TYPE some_metric counter
some_metric{label="val1"} 42
# TYPE another_metric gauge
# HELP another_metric Just an example.
another_metric 2398.283
EOF
```

## 4.5 Grafana配置

### 4.5.1 grafana安装

* 安装grafana

```shell
wget https://dl.grafana.com/oss/release/grafana-6.3.3-1.x86_64.rpm 
sudo yum localinstall grafana-6.3.3-1.x86_64.rpm -y

systemctl enable grafana-server.service
systemctl start grafana-server.service
# web页面3000 登录信息：admin/admin

# 安装插件
grafana-cli plugins install grafana-piechart-panel
systemctl restart grafana-server
```

### 4.5.2 添加数据源

添加prometheus，填写prometheus的管理地址

![image-20190826155302403](/Users/xuel/Library/Application Support/typora-user-images/image-20190826155302403.png)

* 导入dashboard

通过https://grafana.com/grafana/dashboards中获取



*  配置dashboard

![image-20190826144058377](/Users/xuel/Library/Application Support/typora-user-images/image-20190826144058377.png)

![image-20190826150244677](/Users/xuel/Library/Application Support/typora-user-images/image-20190826150244677.png)

### 4.5.3 grafana告警邮件配置

* 修改grafana配置文件，添加email配置

```shell
# 修改/etc/grafana/grafana.ini

[smtp]
enabled = true
host = smtxxxxxxom:465
user = 1xxxxxxxxx
# If the password contains # or ; you have to wrap it with trippel quotes. Ex """#password;"""
password = xxxxxxxxxxxx
;cert_file =
;key_file =
;skip_verify = false
from_address = 1xxxxxxxxxxx3316@163.com
;from_name = Grafana
;ehlo_identity = dashboard.example.com
```

* grafana web界面配置Notification channels

![image-20190826155540354](/Users/xuel/Library/Application Support/typora-user-images/image-20190826155540354.png)

![image-20190826155618702](/Users/xuel/Library/Application Support/typora-user-images/image-20190826155618702.png)



### 4..4 alter配置

⚠️：Template variables are not supported in alert queries，在查询中不能使用模版语法，不然无法创建告警

![image-20190826163837032](/Users/xuel/Library/Application Support/typora-user-images/image-20190826163837032.png)

![image-20190826164353945](/Users/xuel/Library/Application Support/typora-user-images/image-20190826164353945.png)

告警测试

![image-20190826164422146](/Users/xuel/Library/Application Support/typora-user-images/image-20190826164422146.png)

查看告警历史

![image-20190826164436089](/Users/xuel/Library/Application Support/typora-user-images/image-20190826164436089.png)

告警触发

![image-20190826165103853](/Users/xuel/Library/Application Support/typora-user-images/image-20190826165103853.png)

![image-20190826164858353](/Users/xuel/Library/Application Support/typora-user-images/image-20190826164858353.png)



# 五 PromQL

PromQL（Prometheus Query Language）是 Prometheus 自己开发的表达式语言，语言表现力很丰富，内置函数也很多。使用它可以对时序数据进行筛选和聚合。

## 5.1 PromQL语法

### 5.1.1 数据类型

PromQL 表达式计算出来的值有以下几种类型：

- 瞬时向量 (Instant vector): 一组时序，每个时序只有一个采样值
- 区间向量 (Range vector): 一组时序，每个时序包含一段时间内的多个采样值
- 标量数据 (Scalar): 一个浮点数
- 字符串 (String): 一个字符串，暂时未用

### 5.1.2 时序选择器

* 瞬时向量选择器

  瞬时向量选择器用来选择一组时序在某个采样点的采样值。

  最简单的情况就是指定一个度量指标，选择出所有属于该度量指标的时序的当前采样值。比如下面的表达式：

  ```
  http_requests_total
  ```

  可以通过在后面添加用大括号包围起来的一组标签键值对来对时序进行过滤。比如下面的表达式筛选出了 job 为 prometheus，并且 group 为 canary 的时序：

  ```
  http_requests_total{job="prometheus", group="canary"}
  ```

  匹配标签值时可以是等于，也可以使用正则表达式。总共有下面几种匹配操作符：

  * =：完全相等
  * !=: 不相等
  * =~: 正则表达式匹配
  * !~: 正则表达式不匹配

  下面的表达式筛选出了 environment 为 staging 或 testing 或 development，并且 method 不是 GET 的时序：

  ```
  http_requests_total{environment=~"staging|testing|development",method!="GET"}
  ```

  度量指标名可以使用内部标签 `__name__` 来匹配，表达式 `http_requests_total` 也可以写成 `{__name__="http_requests_total"}`。表达式 `{__name__=~"job:.*"}` 匹配所有度量指标名称以 `job:` 打头的时序。

* 区间向量选择器

  区间向量选择器类似于瞬时向量选择器，不同的是它选择的是过去一段时间的采样值。可以通过在瞬时向量选择器后面添加包含在 `[]` 里的时长来得到区间向量选择器。比如下面的表达式选出了所有度量指标为 http_requests_total 且 job 为 prometheus 的时序在过去 5 分钟的采样值。

  ```
  http_requests_total{job="prometheus"}[5m]
  ```

  时长的单位可以是下面几种之一：

  - s：seconds
  - m：minutes
  - h：hours
  - d：days
  - w：weeks
  - y：years

* 偏移修饰器

  前面介绍的选择器默认都是以当前时间为基准时间，偏移修饰器用来调整基准时间，使其往前偏移一段时间。偏移修饰器紧跟在选择器后面，使用 `offset` 来指定要偏移的量。比如下面的表达式选择度量名称为 http_requests_total 的所有时序在 5 分钟前的采样值。

  ```
  http_requests_total offset 5m
  ```

  下面的表达式选择 http_requests_total 度量指标在 1 周前的这个时间点过去 5 分钟的采样值。

  ```
  http_requests_total[5m] offset 1w
  ```

## 5.2 PromQL操作符

### 5.2.1 二元操作符

PromQL 的二元操作符支持基本的逻辑和算术运算，包含算术类、比较类和逻辑类三大类。

* 算术类二元操作符

  算术类二元操作符有以下几种：

  * +：加
  * -：减
  * *：乘
  * /：除
  * %：求余
  * ^：乘方

  算术类二元操作符可以使用在标量与标量、向量与标量，以及向量与向量之间

  > 二元操作符上下文里的向量特指瞬时向量，不包括区间向量。

  * 标量与标量之间，结果很明显，跟通常的算术运算一致。
  * 向量与标量之间，相当于把标量跟向量里的每一个标量进行运算，这些计算结果组成了一个新的向量。
  * 向量与向量之间，会稍微麻烦一些。运算的时候首先会为左边向量里的每一个元素在右边向量里去寻找一个匹配元素（匹配规则后面会讲），然后对这两个匹配元素执行计算，这样每对匹配元素的计算结果组成了一个新的向量。如果没有找到匹配元素，则该元素丢弃。

* 比较类二元操作符

  比较类二元操作符有以下几种：

  * == (equal)
  * != (not-equal)
  * \> (greater-than)
  * < (less-than)
  * \>= (greater-or-equal)
  * <= (less-or-equal)

  比较类二元操作符同样可以使用在标量与标量、向量与标量，以及向量与向量之间。默认执行的是过滤，也就是保留值。可以通过在运算符后面跟 `bool` 修饰符来使得返回值 0 和 1，而不是过滤。

  * 标量与标量之间，必须跟 bool 修饰符，因此结果只可能是 0（false） 或 1（true）。
  * 向量与标量之间，相当于把向量里的每一个标量跟标量进行比较，结果为真则保留，否则丢弃。如果后面跟了 bool 修饰符，则结果分别为 1 和 0。
  * 向量与向量之间，运算过程类似于算术类操作符，只不过如果比较结果为真则保留左边的值（包括度量指标和标签这些属性），否则丢弃，没找到匹配也是丢弃。如果后面跟了 bool 修饰符，则保留和丢弃时结果相应为 1 和 0。

* 逻辑类二元操作符

  逻辑操作符仅用于向量与向量之间。

  * and：交集
  * or：合集
  * unless：补集

  具体运算规则如下：

  * `vector1 and vector2` 的结果由在 vector2 里有匹配（标签键值对组合相同）元素的 vector1 里的元素组成。
  * `vector1 or vector2` 的结果由所有 vector1 里的元素加上在 vector1 里没有匹配（标签键值对组合相同）元素的 vector2 里的元素组成。
  * `vector1 unless vector2` 的结果由在 vector2 里没有匹配（标签键值对组合相同）元素的 vector1 里的元素组成。

* 二元操作符优先级

  PromQL 的各类二元操作符运算优先级如下：

  1. ^
  2. *, /, %
  3. +, -
  4. ==, !=, <=, <, >=, >
  5. and, unless
  6. or

### 5.2.2 向量匹配

前面算术类和比较类操作符都需要在向量之间进行匹配。共有两种匹配类型，`one-to-one` 和 `many-to-one` / `one-to-many`。

* One-to-one 向量匹配

  相同则为匹配，并且只会有一个匹配元素。可以使用 `ignoring` 关键词来忽略不参与匹配的标签，或者使用 `on`关键词来指定要参与匹配的标签。语法如下：

  ```
  <vector expr> <bin-op> ignoring(<label list>) <vector expr>
  <vector expr> <bin-op> on(<label list>) <vector expr>
  ```

  比如对于下面的输入：

  ```
  method_code:http_errors:rate5m{method="get", code="500"}  24
  method_code:http_errors:rate5m{method="get", code="404"}  30
  method_code:http_errors:rate5m{method="put", code="501"}  3
  method_code:http_errors:rate5m{method="post", code="500"} 6
  method_code:http_errors:rate5m{method="post", code="404"} 21
  
  method:http_requests:rate5m{method="get"}  600
  method:http_requests:rate5m{method="del"}  34
  method:http_requests:rate5m{method="post"} 120
  ```

  执行下面的查询：

  ```
  method_code:http_errors:rate5m{code="500"} / ignoring(code) method:http_requests:rate5m
  ```

  得到的结果为：

  ```
  {method="get"}  0.04            //  24 / 600
  {method="post"} 0.05            //   6 / 120
  ```

  也就是每一种 method 里 code 为 500 的请求数占总数的百分比。由于 method 为 put 和 del 的没有匹配元素所以没有出现在结果里。

* Many-to-one / one-to-many 向量匹配

  这种匹配模式下，某一边会有多个元素跟另一边的元素匹配。这时就需要使用 `group_left` 或 `group_right` 组修饰符来指明哪边匹配元素较多，左边多则用 group_left，右边多则用 group_right。其语法如下：

  ```
  <vector expr> <bin-op> ignoring(<label list>) group_left(<label list>) <vector expr>
  <vector expr> <bin-op> ignoring(<label list>) group_right(<label list>) <vector expr>
  <vector expr> <bin-op> on(<label list>) group_left(<label list>) <vector expr>
  <vector expr> <bin-op> on(<label list>) group_right(<label list>) <vector expr>
  ```

  > 组修饰符只适用于算术类和比较类操作符。

  对于前面的输入，执行下面的查询：

  ```
  method_code:http_errors:rate5m / ignoring(code) group_left method:http_requests:rate5m
  ```

  将得到下面的结果：

  ```
  {method="get", code="500"}  0.04            //  24 / 600
  {method="get", code="404"}  0.05            //  30 / 600
  {method="post", code="500"} 0.05            //   6 / 120
  {method="post", code="404"} 0.175           //  21 / 120
  ```

  也就是每种 method 的每种 code 错误次数占每种 method 请求数的比例。这里匹配的时候 ignoring 了 code，才使得两边可以形成 Many-to-one 形式的匹配。由于左边多，所以需要使用 group_left 来指明。

  >  Many-to-one / one-to-many 过于高级和复杂，要尽量避免使用。很多时候通过 ignoring 就可以解决问题。

### 5.2.3 聚合操作符

PromQL 的聚合操作符用来将向量里的元素聚合得更少。总共有下面这些聚合操作符：

- sum：求和
- min：最小值
- max：最大值
- avg：平均值
- stddev：标准差
- stdvar：方差
- count：元素个数
- count_values：等于某值的元素个数
- bottomk：最小的 k 个元素
- topk：最大的 k 个元素
- quantile：分位数

聚合操作符语法如下：

```
<aggr-op>([parameter,] <vector expression>) [without|by (<label list>)]
```

其中 `without` 用来指定不需要保留的标签（也就是这些标签的多个值会被聚合），而 `by` 正好相反，用来指定需要保留的标签（也就是按这些标签来聚合）。

下面来看几个示例：

```
sum(http_requests_total) without (instance)
```

http_requests_total 度量指标带有 application、instance 和 group 三个标签。上面的表达式会得到每个 application 的每个 group 在所有 instance 上的请求总数。效果等同于下面的表达式：

```
sum(http_requests_total) by (application, group)
```

下面的表达式可以得到所有 application 的所有 group 的所有 instance 的请求总数。

```
sum(http_requests_total)
```

## 5.3 函数

Prometheus 内置了一些函数来辅助计算，下面介绍一些典型的，完整的列表请参考 [官方文档](https://prometheus.io/docs/prometheus/latest/querying/functions/)。

- abs()：绝对值
- sqrt()：平方根
- exp()：指数计算
- ln()：自然对数
- ceil()：向上取整
- floor()：向下取整
- round()：四舍五入取整
- delta()：计算区间向量里每一个时序第一个和最后一个的差值
- sort()：排序



# 六 配置告警规则

将报警集成到睿象云

## 6.1 prometheus集成altermanager

```shell
# 编辑prometheus.yml

alerting:
  alertmanagers:
  - static_configs:
    - targets:
       - 'localhost:9093'
       
rule_files:
  - "onealter.yml"
```

## 6.2 编写rule_files规则文件

```shell
# 编写onealter.yml
groups:
  - name: test-rule
    rules:
      - alert: 节点内存使用量
        expr: (node_memory_MemTotal_bytes - (node_memory_MemFree_bytes+node_memory_Buffers_bytes+node_memory_Cached_bytes )) / node_memory_MemTotal_bytes * 100 > 40
        for: 1m
        labels:
          user: prometheus
        annotations:
          summary: "{{$labels.instance}}:内存超过40%"
          description: "{{$labels.instance}}:内存超过40%"
```

## 6.3 编辑altermanager.yml配置webhook回调

```shell
# 编辑

global:
  resolve_timeout: 5m

route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'team-X-pager'
receivers:
- name: 'team-X-pager'
  webhook_configs:
  - url: 'http://api.aiops.com/alert/api/event/prometheus/f307ded7-9a96-4e34-101d-dfc421a8743a'
    send_resolved: true
inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'dev', 'instance']
```

## 6.4 查看alter告警

![image-20190826150506131](/Users/xuel/Library/Application Support/typora-user-images/image-20190826150506131.png)

![image-20190826150526506](/Users/xuel/Library/Application Support/typora-user-images/image-20190826150526506.png)





# 参考链接

* [官网](https://prometheus.io/)
* [Alermanager](https://github.com/prometheus/alertmanager)
* [第三方插件](https://prometheus.io/docs/instrumenting/exporters/#third-party-exporters)
* [routing tree edit](https://prometheus.io/webtools/alerting/routing-tree-editor/)
* [grafana dashboard](https://grafana.com/grafana/dashboards)