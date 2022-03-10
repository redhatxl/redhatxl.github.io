# Terraform 最佳实践

## 一 最佳实践

* Plan before Apply/Stage before Prod

在正式应用资源清单钱，先plan，为保障线上安全，需要现在dev/stage环境测试，再上prod

* 环境隔离(global/stage/prod/mgmt)

单个环境进行隔离

* 每个目录单独.tfstate

每个目录单独.tfstate 状态管理，避免项目冲突与混乱

* 目录间使用terraofrm_remote_state共享状态

* 功能组件分离(web/db...)
* 使用模块modules
* Version Constraints
* variable中添加validation rules
* 使用带模版的模块(git ref)
* 将模块代码放到单独Repo
* Remote state存储使用加密，并启用版本（S3 Versioning）
* TF模版中避免硬编码（eg：使用date获取vpc_id/account_id/region）
* Tag（跟踪成本/区分手动创建）
* 使用terrascan扫描隐患。

* module编写完成后，
* 利用terraform fmt格式化内容
* 利用terratest进行测试

## 二 编写module规范

Module 仓库建立完毕后，首先要 fork 仓库并将其clone到本地。接下来，开始 Mudole 的编写工作。官方已经给出了一个标准的 Module 应该遵循的原则和规范，详见 standard-module-structure . 本文将在此基础上进行补充。

1. 基本原则 每个Module不宜包含过多的资源，要尽可能只包含同一产品的相关资源，这样带来的好处是Module的复杂度不高，便于维护和阅读； 对于统一产品的不同资源，应该分别放在不同的子module中，然后在最外层的main.tf中组织所有的子资源，比如 module slb 中定义了两个资源 slb instance 和 slb attachment，这两个子资源分别定义在了 modules/slb 和 modules/slb_attachment 中，然后在最外层的 main.tf中将这两个module组合为一个新的module 每个module要尽可能单元化，以便可以在实际使用过程中自由添加和删除而不影响其他resource。即一个module尽可能叙述1-2件事情，如创建一个slb实例，并将一组ecs的挂载到这个slb下（slb的作用就是实现对ecs的负载均衡），至于slb的listener配置，应该放置在一个独立的mudule中，一方面listeners比较复杂，涉及四种协议，另一方面，对于同一协议的listener，除了监听端口外，大部分的配置都是相同的，而这些相同的配置可以通过一个单独的module被复用在不同其他资源模板中； 模板编写过程中，需要使用到大量的TF语法，详见configuration syntax 和 interpolation syntax。
2. main.tf 每个 Module 都必须包含一个 main.tf 用于存放 resource 和 datasource。resource和datasource的参数禁止使用硬编码，必须通过变量进行引用； 为了标准起见，每个 resource 和 data source 的命名均以一些关键字或者关键字前缀为主，如this，default，尽量避免使用foo，test等 为了更好的了解Module被他人引用的次数，阿里云Provider支持对每个Module进行打标，即在main.tf的provider中声明字段configuration_source，格式如下： provider "alicloud" { ... configuration_source = "terraform-alicloud-modules/demo" // This should be replaced by the specified owner and module name }
3. variables.tf 每个变量都要添加该参数对应的描述信息，这个信息最终是要呈现在terraform registry官网上的； 对于一些非关系型的参数，可设置一个默认值，如name，description等 对于复杂类型的变量，要显示声明其类型，如list，map 子资源的变量要在Readme中以列表的形式予以呈现
4. outputs.tf module中output的作用是被其他模板和module引用，因此，每个module要讲一些重要的信息输出出来，如资源ID，资源name 等； 重复资源的变量要以列表的形式予以输出，如module ecs-instance中，创建多个instance资源，这些资源的ID应该输出到一个list变量instance_ids中 和variables一样，子资源的output变量也要在Readme中以列表的形式予以呈现
5. README 描述下当前Module是用来干什么的，涉及哪些resource和data source 增加 Usage，指明该如何使用这个 Module。Module 的source 格式为 source = "<Repo Organization>/<NAME>/alicloud"，如：source = "terraform-alicloud-modules/slb-listener/alicloud" 添加module暴露的入参和出参，帮助开发者更好的使用Module 具体细节可参考其他module
6. 执行可开启日志

```shell
export TF_LOG=TRACE
export TF_LOG_PATH=./terraform.log
```

执行完毕后，可查看 Terraform 本地文件夹会生成一个 `terraform.log` 的文件。文件记录了  provider 定义的日志输出。

## 三 实战

### 3.1 使用stats文件作为查询源

在实际应用场景中，可能module A 需要依赖已经运行完成后的module B 的outputs，例如实例绑定安全组，需要先运行安全组module将结果存储在state文件中，module A以module B的运行结果state文件作为数据输入源进行查询操作。

例如使用cos作为backend后端存储。

```yaml
provider "tencentcloud" {
  region = "ap-beijing"
}

terraform {
  required_providers {
    tencentcloud = {
      source  = "registry.terraform.io/tencentcloudstack/tencentcloud"
      version = ">=1.61.5"
    }
  }
  backend "cos" {
    region = "ap-beijing"
    bucket = "cfabackend-xxxxxx"
    prefix = "terraform/state"
  }
}
```

使用cos中的state文件作为terraform_remote_state。

```yaml
data "terraform_remote_state" "cam" {
  backend = "cos"
  config = {
    region = "ap-beijing"
    bucket = "cfabackend-xxxxxx"
    prefix = "terraform/state"
  }
}

locals {
    standards = data.terraform_remote_state.cam.outputs
}

output "result" {
  value = {
    items = { for k, v in local.standards :k=>v}
  }
}
```











​	