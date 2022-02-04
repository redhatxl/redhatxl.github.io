### Terraform-Book

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211209220240.png)

## 一 Terraform概念

Terraform 是一种安全有效地构建、更改和版本控制基础设施的工具(基础架构自动化的编排工具)。它的目标是 "Write, Plan, and create Infrastructure as Code", 基础架构即代码。Terraform 几乎可以支持所有市面上能见到的云服务。具体的说就是可以用代码来管理维护 IT 资源，把之前需要手动操作的一部分任务通过程序来自动化的完成，这样的做的结果非常明显：高效、不易出错。

Terraform 提供了对资源和提供者的灵活抽象。该模型允许表示从物理硬件、虚拟机和容器到电子邮件和 DNS 提供者的所有内容。由于这种灵活性，Terraform 可以用来解决许多不同的问题。这意味着有许多现有的工具与Terraform 的功能重叠。但是需要注意的是，Terraform 与其他系统并不相互排斥。它可以用于管理小到单个应用程序或达到整个数据中心的不同对象。

Terraform 使用配置文件描述管理的组件(小到单个应用程序，达到整个数据中心)。Terraform 生成一个执行计划，描述它将做什么来达到所需的状态，然后执行它来构建所描述的基础结构。随着配置的变化，Terraform 能够确定发生了什么变化，并创建可应用的增量执行计划。

Terraform 是用 Go 语言开发的开源项目，你可以在 [github](https://github.com/hashicorp/terraform) 上访问到它的源代码。

## 二 Terraform核心功能

- 基础架构即代码(Infrastructure as Code)
- 执行计划(Execution Plans)
- 资源图(Resource Graph)
- 自动化变更(Change Automation)

**基础架构即代码(Infrastructure as Code)** 使用高级配置语法来描述基础架构，这样就可以对数据中心的蓝图进行版本控制，就像对待其他代码一样对待它。

**执行计划(Execution Plans)** Terraform 有一个 plan 步骤，它生成一个执行计划。执行计划显示了当执行 apply 命令时 Terraform 将做什么。通过 plan 进行提前检查，可以使 Terraform 操作真正的基础结构时避免意外。

**资源图(Resource Graph)** Terraform 构建的所有资源的图表，它能够并行地创建和修改任何没有相互依赖的资源。因此，Terraform 可以高效地构建基础设施，操作人员也可以通过图表深入地解其基础设施中的依赖关系。

**自动化变更(Change Automation)** 把复杂的变更集应用到基础设施中，而无需人工交互。通过前面提到的执行计划和资源图，我们可以确切地知道 Terraform 将会改变什么，以什么顺序改变，从而避免许多可能的人为错误。

## 三 安装部署



```shell
https://www.terraform.io/downloads.html
unzip terraform_0.13.5_linux_amd64.zip
mv terraform /usr/local/bin/
terraform -version  # 查看Terraform版本和Provider的接口版本信息

```


Terraform是通过一个非常容易使用的命令行界面（CLI）来控制的，并且有且仅有一个命令行程序：`terraform`进行管理。输入`terraform`，可以看到当前版本可用的子命令列表，如`apply`，`plan`等。同时，`terraform`也响应`-h`和`help`，输入`terraform -h`或`terraform help`也可以查看所有可用命令。

初始化provider，使用Terraform配置语法生成 .tf 资源文件，使用CLI实现资源管理；Terraform会将整个资源部署情况更新在 `*.tf.state` 文件中，让用户在前端控制台和后端平台都清晰的把控自己的云资源

```shelll
vi k8smain.tf
provider "kubernetes" {}
terraform init  
ls -al
# 执行 terraform init 初始化Terraform。此步骤，Terraform会自动检测 .tf 文件中的 provider 字段，发送请求到Terraform官方GitHub下载最新版本相关资源的模块和插件，初始化成功时当前脚本的版本信息也会显示出来。
# 有新的版本发布时，可以通过 terraform init -upgrade 指令更新脚本，获取最新的应用。
# 同时，可以通过 terraform plan 预览将要完成的操作，准备好创建资源后，可以通过 terraform apply 进行资源部署。

```

## 四 Terraform关键概念

### 4.1 **Configuration：基础设施的定义和描述**

“基础设施即代码（Infrastructure as Code）”，这里的Code就是对基础设施资源的代码定义和描述，也就是通过代码表达我们想要管理的资源。

```shell
# VPC 资源
resource "alicloud_vpc" "vpc" {
        name          = "tf_vpc"
        cidr_block  = "172.16.0.0/16"
}
# VSwitch 资源
resource "alicloud_vswitch" "vswitch" {
        vpc_id            = alicloud_vpc.vpc.id
        cidr_block        = "172.16.1.0/24"
        availability_zone = "cn-beijing-a"
}
```

对所有资源的代码描述都需要定义在一个以 `tf` 结尾的文件用于Terraform加载和解析，这个文件我们称之为“Terraform模板”或者“Configuration”。

### 4.2 **Provider：基础设施管理组件**

Terraform 通常用于对云上基础设施，如虚拟机，网络资源，容器资源，存储资源等的创建，更新，查看，删除等管理动作，也可以实现对物理机的管理，如安装软件，部署应用等。

[【Provider】](https://yq.aliyun.com/go/articleRenderRedirect?url=https%3A%2F%2Fwww.terraform.io%2Fdocs%2Fproviders%2Findex.html) 是一个与Open API直接交互的后端驱动，Terraform 就是通过Provider来完成对基础设施资源的管理的。不同的基础设施提供商都需要提供一个Provider来实现对自家基础设施的统一管理。目前Terraform目前支持超过160多种的providers，大多数云平台的Provider插件均已经实现了，阿里云对应的Provider为 `alicloud` 。

在操作环境中，Terraform和Provider是两个独立存在的package，当运行Terraform时，Terraform会根据用户模板中指定的provider或者resource／datasource的标志自动的下载模板所用到的所有provider，并将其放在执行目录下的一个隐藏目录 `.terraform` 下。

```tcl
provider "alicloud" {
  version              = ">=1.56.0"
  region               = "cn-hangzhou"
  configuration_source = "terraform-alicloud-modules/classic-load-balance"
}
```

模板中显示指定了一个阿里云的Provider，并显示设置了provider的版本为 `1.56.0+` （默认下载最新的版本），指定了需要管理资源的region，指定了当前这个模板的标识。 通常Provider都包含两个主要元素 resource 和 data source。

### 4.3 **Resource：基础设施资源和服务的管理**

在Terraform中，一个具体的资源或者服务称之为一个resource，比如一台ECS 实例，一个VPC网络，一个SLB实例。每个特定的resource包含了若干可用于描述对应资源或者服务的属性字段，通过这些字段来定义一个完整的资源或者服务，比如实例的名称（name），实例的规格（instance_type），VPC或者VSwitch的网段（cidr_block）等。

定义一个Resource的语法非常简单，通过 `resource` 关键字声明，如下：

```shell
# 定义一个ECS实例
resource "alicloud_instance" "default" {
  image_id        = "ubuntu_16_04_64_20G_alibase_20190620.vhd"
  instance_type   = "ecs.sn1ne.large"
  instance_name   = "my-first-vm"
  system_disk_category = "cloud_ssd"
  ...
}
```

- 其中 `alicloud_instance` 为**资源类型（Resource Type)**，定义这个资源的类型，告诉Terraform这个Resource是阿里云的ECS实例还是阿里云的VPC。
- `default` 为**资源名称(Resource Name)**，资源名称在同一个模块中必须唯一，主要用于供其他资源引用该资源。
- 大括号里面的block块为**配置参数(Configuration Arguments)**，定义资源的属性，比如ECS 实例的规格、镜像、名称等。

显然这个Terraform模板的功能为在阿里云上创建一个ECS实例，镜像ID为 `ubuntu_16_04_64_20G_alibase_20190620.vhd` ，规格为 `ecs.sn1ne.large` ，自定义了实例名称和系统盘的类型。

除此之外，在Terraform中，一个资源与另一个资源的关系也定义为一个资源，如一块云盘与一台ECS实例的挂载，一个弹性IP（EIP）与一台ECS或者SLB实例的绑定关系。这样定义的好处是，一方面资源架构非常清晰，另一方面，当模板中有若干个EIP需要与若干台ECS实例绑定时，只需要通过Terraform的 `count` 功能就可以在无需编写大量重复代码的前提下实现绑定功能。

```shell
resource "alicloud_instance" "default" {
  count = 5
  ...
}
resource "alicloud_eip" "default" {
    count = 5
  ...
}
resource "alicloud_eip_association" "default" {
  count = 5
  instance_id = alicloud_instance.default[count.index].id
  allocation_id = alicloud_eip.default[count.index].id
}

```

显然这个Terraform模板的功能为在阿里云上创建5个ECS实例和5个弹性IP，并将它们一一绑定。

### 4.4 **Data Source：基础设施资源和服务的查询**

对资源的查询是运维人员或者系统最常使用的操作，比如，查看某个region下有哪些可用区，某个可用区下有哪些实例规格，每个region下有哪些镜像，当前账号下有多少机器等，通过对资源及其资源属性的查询可以帮助和引导开发者进行下一步的操作。

除此之外，在编写Terraform模板时，Resource使用的参数有些是固定的静态变量，但有些情况下可能参数变量不确定或者参数可能随时变化。比如我们创建ECS 实例时，通常需要指定我们自己的镜像ID和实例规格，但我们的模板可能随时更新，如果在代码中指定ImageID和Instance，则一旦我们更新镜像模板就需要重新修改代码。 在Terraform 中，Data Source 提供的就是一个查询资源的功能，每个data source实现对一个资源的动态查询，Data Souce的结果可以认为是动态变量，只有在运行时才能知道变量的值。

Data Sources通过 `data` 关键字声明，如下：

```shell
// Images data source for image_id
data "alicloud_images" "default" {
  most_recent = true
  owners      = "system"
  name_regex  = "^ubuntu_18.*_64"
}

data "alicloud_zones" "default" {
  available_resource_creation = "VSwitch"
  enable_details              = true
}

// Instance_types data source for instance_type
data "alicloud_instance_types" "default" {
  availability_zone = data.alicloud_zones.default.zones.0.id
  cpu_core_count    = 2
  memory_size       = 4
}

resource "alicloud_instance" "web" {
  image_id        = data.alicloud_images.default.images[0].id
  instance_type   = data.alicloud_instance_types.default.instance_types[0].id
  instance_name   = "my-first-vm"
  system_disk_category = "cloud_ssd"
  ...
}

```

如上例子中的ECS Instance 没有指定镜像ImageID和实例规格，而是通过 `data`引用，Terraform运行时将首先根据镜像名称前缀选择系统镜像，如果同时有多个镜像满足条件，则选择最新的镜像。实例规格也是类似，在某个可用区下选择2核4G的实例规格进行返回。

### 4.5 **State：保存资源关系及其属性文件的数据库**

Terraform创建和管理的所有资源都会保存到自己的数据库上，这个数据库不是通常意义上的数据库（MySQL，Redis等），而是一个文件名为 `terraform.tfstate` 的文件，在Terraform 中称之为 [`state`](https://yq.aliyun.com/go/articleRenderRedirect?url=https%3A%2F%2Fwww.terraform.io%2Fdocs%2Fstate%2Findex.html)[ ](https://yq.aliyun.com/go/articleRenderRedirect?url=https%3A%2F%2Fwww.terraform.io%2Fdocs%2Fstate%2Findex.html)，默认存放在执行Terraform命令的本地目录下。这个 `state` 文件非常重要，如果该文件损坏，Terraform 将认为已创建的资源被破坏或者需要重建（实际的云资源通常不会受到影响），因为在执行Terraform命令是，Terraform将会利用该文件与当前目录下的模板做Diff比较，如果出现不一致，Terraform将按照模板中的定义重新创建或者修改已有资源，直到没有Diff，因此可以认为Terraform是一个有状态服务。

当涉及多人协作时不仅需要拷贝模板，还需要拷贝 `state` 文件，这无形中增加了维护成本。幸运的是，目前Terraform支持把 `state` 文件放到远端的存储服务 `OSS` 上或者 `consul` 上，来实现 `state` 文件和模板代码的分离。具体细节可参考[官方文档Remote State](https://yq.aliyun.com/go/articleRenderRedirect?url=https%3A%2F%2Fwww.terraform.io%2Fdocs%2Fstate%2Fremote.html)或者关注后续文章的详细介绍。

### 4.6 **Backend：存放 State 文件的载体**

正如上节提到，Terraform 在创建完资源后，会将资源的属性存放在一个 `state` 文件中，这个文件可以存放在本地也可以存放在远端。存放 `state` 文件的载体就是 [`Backend`](https://yq.aliyun.com/go/articleRenderRedirect?url=https%3A%2F%2Fwww.terraform.io%2Fdocs%2Fbackends%2Findex.html) 。 `Backend` 分为本地（local）和远端（remote）两类，默认为本地。远端的类型也非常多，目前官方网站提供的有13种，并且阿里云的[OSS](https://yq.aliyun.com/go/articleRenderRedirect?url=https%3A%2F%2Fwww.terraform.io%2Fdocs%2Fbackends%2Ftypes%2Foss.html)就位列其中。

使用远端的Backend，既可以降低多人协作时对state的维护成本，而且可以将一些敏感的数据存放在远端，保证了数据的安全性。

### 4.7 **Provisioner：在机器上执行操作的组件**

[Provisioner](https://yq.aliyun.com/go/articleRenderRedirect?url=https%3A%2F%2Fwww.terraform.io%2Fdocs%2Fprovisioners%2Findex.html) 通常用来在本地机器或者登陆远程主机执行相关的操作，如 `local-exec` provisioner 用来执行本地的命令， `chef` provisioner 用来在远程机器安装，配置和执行chef client， `remote-exec` provisioner 用来登录远程主机并在其上执行命令。

Provisioner 通常跟 Provider一起配合使用，provider用来创建和管理资源，provisioner在创建好的机器上执行各种操作。

### 4.8 **变量**

Terraform 运行时会读取工作目录中所有的 *.tf, *.tfvars 文件，所以我们不必把所有的东西都写在单个文件中去，应按职责分列在不同的文件中，例如：

　　provider.tf -- provider 配置

　　terraform.tfvars -- 配置 provider 要用到的变量

　　varable.tf -- 通用变量

　　resource.tf -- 资源定义

　　data.tf -- 包文件定义

　　output.tf -- 输出



Terraform引用了一些环境变量来控制部分功能，这些环境变量都不是必需的，但是可以改变一些Terraform的默认行为，帮助用户适配更多应用场景

```
# 操作日志是重要的运维信息来源，用户可以通过设置日志类型TF_LOG和日志保存路径TF_LOG_PATH，将详细的日志打印到stderr，以获取调试信息。TF_LOG支持五种可用值，TRACE，DEBUG，INFO，WARN，ERROR，分别代表五种不同的日志级别，其中TRACE表示最详细的日志
export TF_LOG=TRACE   # DEBUG INFO WARN ERROR
export TF_LOG_PATH=/root/optf/tf.log
```

### 4.9 **Terraform中输入变量**

`variable`是Terraform重要的配置文件类型之一，通过对变量的集中管理，用户可以在资源文件中直接引用变量名进行赋值。首先需要先定义(声明)变量，放到一个`.tf`文件中，如：

创建`variable.tf`文件，配置参数的默认值

```shell
variable "access_key" {}
variable "secret_key" {}
variable "region" {
  default = "us-east-1"
}
variable "f5user" {
  type = string
  default = "admin"
}
variable "f5pass" {
  type = string
  default = "admin"
}

```

上面定义了变量。前两个变量是空的，第三个给了一个默认值(默认参数)。此时运行`terraform plan`，Terraform会提示输入这些尚未定义的变量。

引用变量，使用`${var.xxx}`的形式。这样在资源配置文件中，参数可以直接调用var.f5user

```shell
provider "aws" {
  access_key	=	"${var.access_key}"
  secret_key	=	"${var.secret_key}"
  region		=	"${var.region}"
}
```

### 4.10 **变量赋值**

前面我们声明了变量，但是还没有给变量赋值，无法真正使用。给变量赋值，有以下几种方法，下面几种方法按照变量赋值的优先顺序排序。

#### 4.10.1 命令行声明

使用terraform的各种命令时，使用`-var`选项，可以在后面直接跟变量的定义，如：

```
	# terraform apply \
	  -var 'access_key=foo' 
	  -var 'secret_key=bar'
	# ...
```

以这种方式赋值变量是一次性的，并不会保存它们的值，也就是说下一次重新执行命令时，需要重新赋值。

在terraform apply中，直接设置变量值会覆盖掉`variable.tf`中设置的默认值

#### 4.10.2 从文件导入



为永久性存储一个变量的值，可以将其放在文件中保存。Terraform会自动加载当前目录下扩展名为`.tfvars`和`.auto.tfvars`的文件来填充定义的变量。如果以其他格式存放，可以使用`-var-file`选项来手动指定需要加载的变量值文件。这些文件使用Terraform格式或JSON格式。

使用文件也方便版本控制，但是用户名、密码这种东西就不要用版本控制管理的。因此可以将用户名和密码这类信息单独放在一个文件中，使用`-var-file`来手动指定。其他的，可以自动填充，方便使用版本控制管理的，可以直接放在`.tfvars`文件中，Terraform会自动加载。

#### 4.10.3 环境变量

Terraform会读取`TF_VAR_name`这种格式的环境变量，用来填充定义好的变量。比如，环境变量中有一个`TF_VAR_access_key`的变量，Terraform就会读取到，并用于填充`access_key`变量。

#### 4.10.4 default值

如果某个变量没有采用以上任何一种方法来进行赋值，那么如果在变量的定义中有个`default`属性，那么Terraform就会使用`default`的值来对变量进行赋值。

#### 4.10.5 

没有使用任何方法来对变量赋值，在输入命令时使得Terraform不知道如何处理，此时就会出现交互界面，让用户手动输入变量值，来给变量赋值。

也可以利用`TF_VAR_name`把变量设置在环境变量中

```
export TF_VAR_f5user="admin"
```

配置`TF_INPUT`，可以关闭对未指定值的变量的提示。将刚才的`variable.tf`中设置的参数删除

```
export export TF_INPUT=1
```

执行Terraform指令，会要求写入参数值

设置`TF_INPUT`为`false`或`0`，再次执行指令，系统报错：未指定变量的值

```
export export TF_INPUT=0
```

CLI Config File

用户可以通过CLI的配置文件对CLI进行一些设置，适用于所有Terraform的工作目录，与资源配置文件是区分开的。

配置文件中支持的参数有：

① 是否开启更新与安全检查：`disable_checkpoint`

② 允许更新与安全检查，但禁止使用匿名id删除警告消息：`disable_checkpoint_signature`

③ 启用插件缓存，以字符串的形式指定插件缓存目录的位置：`plugin_cache_dir`

④ Terraform企业版凭证：`credentials`

可以在环境变量中配置CLI Config File的位置

```
export TF_CLI_CONFIG_FILE="$HOME/.terraformrc-custom"
```

## 五 资源管理常用命令

### 5.1 **terraform plan**：资源的预览

在初始化目录下执行 `terraform plan` 查看部署计划；参数前面的`+`代表新添加的资源，当销毁资源时，参数前面对应的符号会变为`-`；更改一些参数需要重新部署资源时，该资源前面的符号为`-/+`；在旧参数和新参数内容之间有`→`符号标识。

plan命令用于对模板中所定义资源的预览，主要用于以下几个场景：

- 预览当前模板中定义的资源是否符合管理预期，和Markdown的预览功能类似。

- 如果当前模板已经存在对应的state文件，那么plan命令将会展示模板定义与state文件内容的diff结果，如果有变更，会将结果在下方显示出来。
- 对DataSource而言，执行plan命令，即可直接获取并输出所要查询的资源及其属性。

### 5.2 **terraform apply**：资源的新建和变更

apply命令用于实际资源的新建和变更操作，为了安全起见，在命令运行过程中增加了人工交互的过程，即需要手动确认是否继续，当然也可以通过**--auto-approve**参数来跳过人工确认的过程。

apply命令适用于以下几种场景：

- 创建新的资源。
- 通过修改模板参数来修改资源的属性。
- 如果从当前模板中删除某个资源的定义，apply命令会将该资源彻底删除。可以理解为“资源的移除也是一种变更”。

### 5.3 **terraform show**：资源的展示

show命令用于展示当前state中所有被管理的资源及其所有属性值。

### 5.4 **terraform destroy**：资源的释放

destroy命令用于对资源的释放操作，为了安全起见，在命令执行过程中，也增加了人工交互的过程，如果想要跳过手动确认操作，可以通过**--force**参数来跳过。

terraform destroy默认会释放当前模板中定义的所有资源，如果只想释放其中某个特定的资源，可以通过参数`-target=<资源类型>.<资源名称>`来指定。

### 5.5 **terraform import**：资源的导入

import命令用于将存量的云资源导入到terraform state中，进而加入到Terraform的管理体系中

### 5.6 **terraform taint**：标记资源为被污染

taint命令用于把某个资源标记为被污染状态，当再次执行apply命令时，这个被污染的资源将会被先释放，然后再创建一个新的，相当于对这个特定资源做了先删除后新建的操作。

命令的详细格式为： `terraform taint <资源类型>.<资源名称>` ，如：

```
terraform taint bigip_ltm_pool_attachment.attach_nodeqat01
Resource instance bigip_ltm_pool_attachment.attach_nodeqat01 has been marked as tainted.
```

### 5.7 **terraform untaint**：取消被污染标记

untaint命令是taint的逆向操作，用于取消被污染标记，使其恢复到正常的状态。命令的详细格式和taint类似为：`terraform untaint <资源类型>.<资源名称>` ，如：

```
terraform untaint untaint bigip_ltm_pool_attachment.attach_nodeqat01
Resource instance bigip_ltm_pool_attachment.attach_nodeqat01 has been successfully untainted.
```

### 5.8 **terraform output**：打印出参及其值

如果在模板中显示定义了output参数，那么这个output的值将在apply命令之后展示，但plan命令并不会展示，如果想随时随地快速查看output的值，可以直接运行命令 terraform output

Terraform对资源状态的管理，实际上是对State文件中数据的管理。State文件保存了当前Terraform管理的所有资源及其属性，内容都是由Terraform自动存储的，为了保证数据的完整性，不建议手动修改State内容。对State数据的操作可以通过**terraform state**命令来完成。

### 5.9 **terraform state list**：列出当前state中的所有资源

**state list**命令会按照 `<资源类型>.<资源名称>` 的格式列出当前state中存在的所有资源（包括datasource），例如：

```
$ terraform state list
data.alicloud_slbs.default
alicloud_vpc.default
alicloud_vswitch.this
```

### 5.10 **terraform state show**：展示某一个资源的属性

**state show**命令按照Key-Value的格式展示出特定资源的所有属性及其值，命令的完整格式为 `terraformstate show <资源类型>.<资源名称>` ，例如：

```
$ terraform state show alicloud_vswitch.this
# alicloud_vswitch.this:
resource "alicloud_vswitch" "this" {  
    availability_zone = "eu-central-1a"
    cidr_block        = "172.16.0.0/24"
    id                = "vsw-gw8gl31wz******"
    vpc_id            = "vpc-gw8calnzt*******"
}
```

### 5.11 **terraform state pull**：获取当前state内容并展示

**state pull**命令用于原样展示当前state文件数据，类似与Shell下的cat命令，例如：

```
$ terraform state pull
{
    "version": 4,
    "terraform_version": "0.12.8",
    "serial": 615, 
    "lineage": "39aeeee2-b3bd-8130-c897-2cb8595cf8ec", 
    "outputs": {
        ***
    }
  }, 
"resources": [
    {     
        "mode": "data",    
        "type": "alicloud_slbs",     
        "name": "default",     
        "provider": "provider.alicloud",
          ***
    },
    {     
        "mode": "managed",    
        "type": "alicloud_vpc",     
        "name": "default",    
        "provider": "provider.alicloud",
          ***
    }
  ]
}
```

### 5.12 **terraform state rm**：移除特定的资源

**state rm**命令用于将state中的某个资源移除，但是实际上并不会真正删除这个资源，命令格式为：`terraformstate rm <资源类型>.<资源名称>` ，例如：

```
$ terraform state rm alicloud_vswitch.this
Removed alicloud_vswitch.this
Successfully removed 1 resource instance(s).
```

移除后，如果模板内容不变并且再次执行**apply**命令，将会新增一个同样的资源。移除后的资源可以再次通过**import**命令再次加入。

### 5.13 **terraform state mv**：变更特定资源的存放地址

如果想调整某个资源所在的state文件，可以通过**state mv**命令来完成，类似于Shell下的**mv**命令，这个命令的使用有多种选项，可以通过命令 **terraform state mv --help** 来详细了解。本文只介绍最常用的一种：`terraform state mv --state=./terraform.tfstate --state-out=<target path>/terraform-target.tfstate <资源类型>.<资源名称A> <资源类型>.<资源名称B>` ，如：

```
$ terraform state mv --state-out=../tf.tfstate alicloud_vswitch.this alicloud_vswitch.default
Move "alicloud_vswitch.this" to "alicloud_vswitch.default"
Successfully moved 1 object(s)
```

如上命令省略了默认的`--state=./terraform.tfstate`选项，命令最终的结果是将当前State中的VSwitch资源移动到了上层目录下名为 `tf.tfstate` 的State中，并且将VSwitch的资源名称由“this”改为了“default”。

### 5.14 **terraform refresh**：刷新当前state

**refresh**命令可以用来刷新当前State的内容，即再次调用API并拉取最新的数据写入到state文件中。

### 5.15 **terraform graph**：输出当前模板定义的资源关系图

每个模板定义的资源之间都存在不同程度的关系，如果想看资源关系图，可以使用命令**terraform graph**:

```
$ terraform graph
digraph {
        compound = "true"
        newrank = "true"
        subgraph "root" {
                "[root] alicloud_vpc.default" [label = "alicloud_vpc.default", shape = "box"]
                "[root] alicloud_vswitch.this" [label = "alicloud_vswitch.this", shape = "box"]             
                ******
                "[root] output.vswitchId" -> "[root] alicloud_vswitch.this"
                "[root] provider.alicloud (close)" -> "[root] alicloud_vswitch.this"
                ******
                "[root] root" -> "[root] provider.alicloud (close)"
        }
} 
```

该命令的结果还可以通过命令`terraform graph | dot -Tsvg > graph.svg`直接导出为一张图片（需要提前安装graphviz： `brew install graphviz `）：

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211210134911.png)

### 5.16 **terraform validate**：验证模板语法是否正确

Terraform模板的编写需要遵循其自身定义的一套简单的语法规范，编写完成后，如果想要检查模板是否存在语法错误或者在运行**plan**和**apply**命令的时候报语法错误，可以通过执行命令**terraform validate**来检查和定位错误出现的详细位置和原因。

*详细介绍每一个具体的指令，包括如何使用和可能遇到的问题*

- apply

`terraform apply` 用于应用所需的更改，以达到所需的配置状态，同时执行结果会保存在本地状态文件`terraform.tfstate`中。

**标准语法：**`terraform apply [options] [dir-or-plan]`

- `options`用来填写`apply`的flags
- `dir-or-plan`用来指定配置计划或计划的路径

在当前目录只配置`provider.tf`，不添加任何资源文件，执行`terraform apply`，显示没有任何资源被部署

在当前目录执行`terraform apply ./tftest`命令，创建在`/tftest`目录的资源文件将被部署

```
terraform apply ./tftest
```

options

- `-backup=path` - 备份文件的路径，设置为`-`时表示禁用

```
terraform apply -backup=-
```

删除`terraform.tfstate.backup`，执行`terraform apply -backup=-`，不再自动保存备份

- `-auto-approve` - 跳过部署计划前的审批过程，直接创建资源

```
terraform apply -auto-approve
```

- `-no-color` - 禁用输出时字符的颜色

```
terraform apply -no-color
```

- `-parallelism=n` - 限制并发操作的数量，默认是`10`

```
terraform apply -parallelism=5
```

- `-refresh=true` - 在计划和应用之前，更新每一个资源的状态

```
terraform apply -refresh=true
```

- `-state=path` - 状态文件的路径，默认为`terraform.tfstate`

```
terraform apply -state=./test_state
```

删除`terraform.tfstate`，执行`terraform apply -state=./test_state`，将状态文件保存在当前文件夹下的`test_state`中

- console

`terraform console`提供了一个用于评估和验证表达式的交互控制台。

**标准语法：**`terraform console [options] [dir]`

- `options`用来填写`console`的flags
- `dir`用来指定要使用的目录，默认为当前目录

```
    // Evaluating and experimenting with expressions
    $ echo "1 + 2" | terraform console
```

- destroy

`terraform destroy`用于销毁terraform管理的基础设施。

**标准语法：**`terraform destroy [options] [dir]`

`options`用来填写`destroy`的flags

- `dir`用来指定要使用的目录，默认为当前目录
- `-auto-approve` - 同`apply`命令中的`-auto-approve`，跳过销毁计划前的审批过程，直接销毁资源

- fmt

`terraform fmt`用于将terraform配置文件重写为规范格式和样式，确保文件的一致性。在升级Terraform之后，建议您在模块上预先运行`Terraform fmt`，使之前的文件适配新版本。

**标准语法：**`terraform fmt [options] [dir]`

- `options`用来填写`fmt`的flags
- `dir`用来指定要使用的目录，默认为当前目录
- `-list=false` - 不列出格式不一致的文件

```
    // Don't list the files containing formatting inconsistencies
    $ terraform fmt -list=false
```

- `-diff` - 显示格式更改的差异

```
    // Display diffs of formatting changes
    $ terraform fmt -diff
```

## 参考链接

* https://github.com/ilzj/terraform-book/blob/main/learn.md