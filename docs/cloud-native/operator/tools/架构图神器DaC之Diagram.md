# 架构图神器DaC之Diagram

# 一 背景

通过图可以用 Python 代码绘制云系统架构。它的诞生是为了在没有任何设计工具的情况下对一个新的系统架构设计进行原型化。您还可以描述或可视化现有的系统体系结构。图目前支持的主要供应商包括: AWS，Azure，GCP，Kubernetes，阿里巴巴云，甲骨文云等。它还支持 On-Premise 节点、 SaaS 以及主要的编程框架和语言。

代码关系图还允许您跟踪任何版本控制系统中的体系结构关系图更改。注意: 它不控制任何实际的云资源，也不生成云形成或地形代码。它只是用于绘制云系统体系结构图。

相比于在 UI 上对各种图标进行拖拽和调整，这种DaC(Diagram as Code)方式更符合我们程序员的使用习惯。

# 二 支持范围

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20201206211914.png)

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20201206211933.png)

# 三 示例图片

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20201206211851.png)

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20201206211832.png)



# 四 安装部署

## 4.1 Python环境

python环境版本为3.6以上，由于我本地python环境管理使用anconda，也可以使用其他工具。

```shell
 conda create -n py-diagrams python=3.6
conda activate py-diagrams

```

## 4.2 graphviz安装

它使用Graphviz来呈现图表，因此您需要安装Graphviz来使用图表。安装graphviz（或已经安装）之后，安装这些关系图。

```shell
apt-get install graphviz

# macos

brew install graphviz
```



# 五 使用

## 5.1 编写代码

```python
# diagram.py
from diagrams import Diagram
from diagrams.aws.compute import EC2
from diagrams.aws.database import RDS
from diagrams.aws.network import ELB

with Diagram("Grouped Workers", show=False, direction="TB"):
    ELB("lb") >> [EC2("worker1"),
                  EC2("worker2"),
                  EC2("worker3"),
                  EC2("worker4"),
                  EC2("worker5")] >> RDS("events")
```

## 5.2 生成图片

```shell
python diagram.py
```

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20201206211808.png)

# 六 逻辑简介

## 6.1 绘图对象

### 6.1.1 Diagram

`Diagram`：这是表示图的最主要的对象，代表一个架构图

使用 `Diagram` 类来创建图环境上下文，使用 `with` 语法来使用这个上下文。`Diagram` 的第一个参数是会被用作架构图的名称以及输出的图片文件名（转换为小写+下划线）。

`Diagram` 支持参数：

- `outformat`：指定输出图片的类型，默认是 `png`，可以是 `png`、`jpg`、`svg` 和 `pdf`
- `show`：指定是否显示图片，默认是 `False`
- `graph_attr`、`node_attr` 和 `edge_attr`：指定 `Graphviz` 属性选项，用来控制图、点、线的样式，详情查看 [参考链接](https://www.graphviz.org/doc/info/attrs.html)

### 6.1.2 Node	

`Node`：表示一个节点或系统组件，比如`快速开始`中的`SLB`、`ECS`和`RDS`都是架构图中的节点。

节点之间的关系使用操作符来表示，分别是：

- `>>`：左节点指向右节点
- `<<`：右节点指向左节点
- `-`：节点互相连接，没有方向

### 6.1.3 Cluster

`Cluster`：表示集群或分组，可将多个节点放到一个集群中，它也是一个上下文管理器，使用 `with` 语法。

## 6.2 贡献云厂商

需要在 `resources/aws` 文件夹中更新资源图片，然后执行 `./autogen.sh` 即可。`./autogen.sh` 会对 `resources/` 做这么几件事：

- 将特定云供应商的 `svg` 图片转换为 `png`
- 将特定云供应商的图片调整为圆角图片
- 自动生成节点类代码
- 自动生成文档
- 使用 `black` 格式化自动生成的代码

## 6.3 自定义图片

* 定义图片变量，将链接地址urlretrieve为icon，就可以使用。

```python
from urllib.request import urlretrieve

from diagrams import Cluster, Diagram
from diagrams.aws.database import Aurora
from diagrams.custom import Custom
from diagrams.k8s.compute import Pod

# Download an image to be used into a Custom Node class
#rabbitmq_url = "https://jpadilla.github.io/rabbitmqapp/assets/img/icon.png"
rabbitmq_url = "https://img2018.cnblogs.com/blog/1538609/201907/1538609-20190720105324996-496731759.jpg"
rabbitmq_icon = "rabbitmq.png"
urlretrieve(rabbitmq_url, rabbitmq_icon)

with Diagram("01-Broker Consumers", show=False):
    with Cluster("Consumers"):
        consumers = [
            Pod("worker"),
            Pod("worker"),
            Pod("worker")]

    queue = Custom("Message queue", rabbitmq_icon)

    queue >> consumers >> Aurora("Database")
```

* 生成图片

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20201206211740.png)


# 参考链接

* https://github.com/mingrammer/diagrams

* https://github.com/blushft/go-diagrams
* https://diagrams.mingrammer.com/docs/getting-started/installation

* https://diagrams.mingrammer.com/docs/getting-started/examples

* https://blog.csdn.net/wangbinxin001/article/details/104344692
* https://www.tonyballantyne.com/graphs.html