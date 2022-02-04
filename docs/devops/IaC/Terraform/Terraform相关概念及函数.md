# Terraform 相关概念及函数

## 一 Metadata

Metadata是Terraform支持的内置元参数，可以在 provider，resource，data块中使用。本章节主要介绍 resource块支持的元参数，主要包括：

- depends_on：用于指定资源的依赖项
- count：用于创建多个相同配置的资源
- for_each：用于根据映射、字符串集合创建多个资源
- provider：用于选择非默认的 provider
- lifecycle：用于定制资源的生命周期

### 1.1 depends_on

在同一个 Terraform 配置文件中可以包含多个资源。通过在资源中引用其他资源的属性值，Terraform可以自动推断出资源的依赖关系。然而，某些资源的依赖关系对于Terraform是不可见的，这就需要使用 depends_on 来创建显式依赖。我们可以使用 depends_on 来更改资源的创建顺序或执行顺序，使其在所依赖资源之后处理。depends_on 的表达式是依赖资源的地址列表。例如我们在远程操作一台ECS服务器之前，需要为其绑定EIP或配置NAT规则。

```
resource "huaweicloud_compute_instance" "myinstance" {
  ...
}

resource "huaweicloud_vpc_eip" "myeip" {
  ...
}

resource "huaweicloud_compute_eip_associate" "associated" {
  public_ip   = huaweicloud_vpc_eip.myeip.address
  instance_id = huaweicloud_compute_instance.myinstance.id
}

resource "null_resource" "provision" {
  depends_on = [huaweicloud_compute_eip_associate.associated]

  provisioner "remote-exec" {
    connection {
      # 通过公网地址访问 ECS
      host = huaweicloud_vpc_eip.myeip.address
      ...
    }
    inline = [
      ...
    ]
  }
}
```

### 1.2 count

默认情况下，Terraform的 resource块只配置一个资源。当我们需要创建多个相同的资源时，如果配置多个独立的 resource块就显得很冗余，且不利于维护。我们可以使用 count 或 for_each 参数在同一个 resource块中管理多个相同的资源。在同一个 resource块中不能同时使用count 和 for_each 参数。示例如下：

```
resource "huaweicloud_evs_volume" "volumes" {
  count = 3

  size              = 20
  volume_type       = "SSD"
  availability_zone = "cn-north-4a"
}
```

我们通过如上配置创建了3个相同的云硬盘（EVS）。在很多情况下，Provider 要求创建资源的某些参数具有唯一性，这时我们可以使用 "count.index" 属性来进行区分，这是一个从0开始计数的索引值。

```
resource "huaweicloud_vpc" "vpcs" {
  count = 2
  name  = "myvpc_${count.index}"
  cidr  = "192.168.0.0/16"
}
```

我们通过如上配置创建了两个VPC，名字分别为 myvpc_0 和 myvpc_1，它们具有相同的CIDR值。如果进一步修改CIDR值，我们可以声明一个string列表用于存储不同VPC的CIDR值，然后通过 count.index 去访问列表元素。

```
variable "name_list" {
  type    = list(string)
  default = ["vpc_demo1", "vpc_demo2"]
}
variable "cidr_list" {
  type    = list(string)
  default = ["192.168.0.0/16", "172.16.0.0/16"]
}

resource "huaweicloud_vpc" "vpcs" {
  count = 2
  name  = var.name_list[count.index]
  cidr  = var.cidr_list[count.index]
}
```

使用 count 创建的资源需要通过索引值进行访问，格式为：<资源类型>.<名称>[索引值]

```
# 访问第一个VPC
> huaweicloud_vpc.vpcs[0]

# 访问第一个VPC的ID
> huaweicloud_vpc.vpcs[0].id

# 访问所有VPC的ID
> huaweicloud_vpc.vpcs[*].id
```

### 1.3 for_each

for_each 在功能上与 count 相似，for_each 使用键值对或字符串集合的形式快速地将值填入到对应的属性中，不仅可以优化脚本结构也有利于理解多实例间的关系。

在使用映射类型表达时，我们可以使用 "each.key" 和 "each.value" 来访问映射的键和值。以创建VPC为例，通过 for_each 中的键值对，我们可以灵活配置VPC的名称和CIDR。

```
resource "hauweicloud_vpc" "vpc" {
  for_each = {
    vpc_demo1 = "192.168.0.0/16"
    vpc_demo2 = "172.16.0.0/16"
}

  name = each.key
  cidr = each.value
}
```

在使用字符串集合类型表达时，"each.key" 等同于 "each.value"，我们一般使用 each.key表示，另外，可以通过 toset() 函数将定义的 list 类型进行转化

```
resource "huaweicloud_networking_secgroup" "mysecgroup" {
  for_each = toset(["secgroup_demo1", "secgroup_demo2"])
  name     = each.key
}

# 通过变量表示 for _each
variable "secgroup_name" {
  type = set(string)
}
resource "huaweicloud_networking_secgroup" "mysecgroup" {
  for_each = var.secgroup_name
  name     = each.key
}
```

使用 for_each 创建的资源需要通过键名进行访问，格式为：<资源类型>.<名称>[键名]

```
# 访问 vpc_demo1
> huaweicloud_vpc.vpcs["vpc_demo1"]

# 访问 vpc_demo1 的ID
> huaweicloud_vpc.vpcs["vpc_demo1"].id
```

由于 count 和 for_each 都可用于创建多个资源，建议参考以下规则进行选择：

1、如果资源实例的参数完全或者大部分一致，建议使用count；

2、如果资源的某些参数需要使用不同的值并且这些值不能由整数派生，建议使用 for_each；

### 1.4 provider

在Terraform中，我们可以使用 provider块创建多个配置，其中一个 provider块为默认配置，其它块使用 "alias" 标识为非默认配置。在资源中使用元参数 provider 可以选择非默认的 provider块。例如我们需要在不同的地区管理资源，首先需要声明多个 provider块：

```
provider "huaweicloud" {
  region = "cn-north-1"
  ...
}

provider "huaweicloud" {
  alias  = "guangzhou"
  region = "cn-south-1"
  ...
}
```

示例中我们声明了北京和广州的华为云provider，并对广州地区的provider增加了别名。我们在资源中使用元参数 provider 来选择非默认的 provider块，其格式为：<provider名称>.<别名>。

```
resource "huaweicloud_networking_secgroup" "mysecgroup" {
  # 使用非默认 provider块名，对应非默认provider块的别名(alias)
  provider = huaweicloud.guangzhou
  ...
}
```

华为云Provider 支持在Resource中指定region参数，可以在不同的地区创建资源。相比 alias + provider 的方式，这种方式更加灵活简单。

```
provider "huaweicloud" {
  region = "cn-north-1"
  ...
}

resource "huaweicloud_vpc" "example" {
  region = "cn-south-1"
  name   = "terraform_vpc"
  cidr   = "192.168.0.0/16"
}
```

### 1.5 lifecycle

每个资源实例都具有创建 、更新和销毁三个阶段，在一个资源实例的生命周期过程中都会经历其中的2至3个阶段。通过元参数 lifecycle 可以对资源实例的生命周期过程进行改变，lifecycle 支持以下参数：

#### 1.5.1 **create_before_destroy**

默认情况下，当我们需要改变资源中不支持更新的参数时，Terraform会先销毁已有实例，再使用新配置的参数创建新的对象进行替换。当我们将 create_before_destroy 参数设置为 true 时，Terraform将先创建新的实例，再销毁之前的实例。这个参数可以适用于保持业务连续的场景，由于新旧实例会同时存在，需要提前确认资源实例是否有唯一的名称要求或其他约束。

```
lifecycle {
  create_before_destroy = true
}
```

#### 1.5.2 **prevent_destroy**

当我们将 prevent_destroy 参数设置为true时，Terraform将会阻止对此资源的删除操作并返回错误。这个元参数可以作为一种防止因意外操作而重新创建成本较高实例的安全措施，例如数据库实例。如果要删除此资源，需要将这个配置删除后再执行 destroy 操作。

```
lifecycle {
  prevent_destroy = true
}
```

#### 1.5.3 **ignore_changes**



默认情况下，Terraform plan/apply 操作将检测云上资源的属性和本地资源块中的差异，如果不一致将会调用更新或者重建操作来匹配配置。我们可以用 ignore_changes 来忽略某些参数不进行更新或重新。ignore_changes 的值可以是属性的相对地址列表，对于 Map 和 List 类型，可以使用索引表示法引用，如 tags["Name"]，list[0] 等。

```
resource "huaweicloud_rds_instance" "myinstance" {
  ...
  lifecycle {
    ignore_changes = [
      name,
    ]
  }
}
```

此时，Terraform 将会忽略对 name 参数的修改。除了列表之外，我们也可以使用关键字 all 忽略所有属性的更新。

```
resource "huaweicloud_rds_instance" "myinstance" {
  ...
  lifecycle {
    ignore_changes = all
  }
}
```



## 二 常用函数

### 2.1 cocat与flatten 区别

* Concat 接受两个或多个列表并将它们组合成单个列表。

```shell
> concat(["a", ""], ["b", "c"])
[
  "a",
  "",
  "b",
  "c",
]
```

* flatten

  flatten 接受一个列表并将任何列表元素替换为列表内容的扁平序列。

```shell
> flatten([["a", "b"], [], ["c"]])
["a", "b", "c"]

```

### 2.2 lookup 和 element

* lookup

```go
> lookup({a="ay", b="bee"}, "a", "what?")
ay
> lookup({a="ay", b="bee"}, "c", "what?")
what?

```

* element

```go
# element 从列表中检索单个元素。
> element(["a", "b", "c"], 1)
b

# 如果给定的索引大于列表的长度，则通过将索引取模列表的长度来“环绕”索引：
> element(["a", "b", "c"], 3)
a

> element(["a", "b", "c"], length(["a", "b", "c"])-1)
c

```

### 2.3 Merge

Merge 接受任意数量的映射或对象，并返回包含来自所有参数的合并元素集的单个映射或对象。如果多个给定的映射或对象定义了相同的键或属性，那么参数序列中后面的那个将优先。如果参数类型不匹配，则在应用合并规则之后，结果类型将是与属性类型结构匹配的对象。

```go
> merge({a="b", c="d"}, {e="f", c="z"})
{
  "a" = "b"
  "c" = "z"
  "e" = "f"
}

> merge({a="b"}, {a=[1,2], c="z"}, {d=3})
{
  "a" = [
    1,
    2,
  ]
  "c" = "z"
  "d" = 3
}

```





## 三 表达式

如果 var.list 是所有具有属性 id 的对象的列表，则可以使用以下 for 表达式生成 id 列表：

```go
[for o in var.list : o.id]
var.list[*].id


var.list[*].interfaces[0].name
[for o in var.list : o.interfaces[0].name]

```







## 参考链接

* https://support.huaweicloud.com/basics-terraform/terraform_0015.html
* https://cloud.tencent.com/document/product/1213/67049
* https://www.terraform.io/language/functions/merge