## Terraform Cloud

## 一 背景

Hashicorp在2019年5月决定将Terraform Cloud的远程状态管理功能免费开放给开源版用户。这个版本也同时开放了更多的免费功能给不超过5人的团队使用。Terraform Cloud的功能分成免费版、团队版以及集中控制功能，本文主要对免费版功能进行介绍。

Terraform Cloud 是一个帮助团队一起使用 Terraform 的应用程序。它管理 Terraform 在一个一致和可靠的环境中运行，包括容易访问共享的状态和秘密数据，用于批准基础设施更改的访问控制，用于共享 Terraform 模块的私有注册，用于管理 Terraform 配置内容的详细策略控制，等等。

## 二 账号注册

https://app.terraform.io/signup/account?utm_source=cloud_landing&utm_content=offers_tfc

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220309164618.png)



![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220309164955.png)

## 三 简单示例

尝试一个示例配置

1. 在终端中运行以下命令并按照提示获取 API 令牌以供 Terraform 使用。如果您没有 Terraform 0.13 或更高版本，则需要安装它 第一的

```shell
terraform login
```

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220309165742.png)

获取登录token，填写至cli终端进行登录。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220309165957.png)

1. 克隆示例仓库 .

```shell
git clone https://github.com/hashicorp/tfc-getting-started.git
```

3. 运行设置脚本并按照提示完成设置并执行您的第一次 Terraform Cloud 运行！

```shell
cd tfc-getting-started && ./scripts/setup.sh
```

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220309170707.png)

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220309170735.png)

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220309170933.png)

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220309173148.png)

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220309173342.png)

* state文件

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220309173547.png)

## 四 自建组织和工作空间

创建组织，创建workspace

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220309173738.png)

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220309173830.png)

选择github，除了GitHub还支持GitLab，Bitbucket，Azure DevOps。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220309173903.png)

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220309174017.png)

Fork 这个示例仓库到自己github：https://github.com/redhatxl/tencent-cloud-simple-example

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220309174254.png)

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220309174317.png)

下一步对workspace的变量进行配置。这里的变量包括以前在单机版上的环境变量，以及源代码tfvars文件中的terraform变量：

注意这里可以有选择的将一些变量标记成敏感，这样该变量的具体数值就不会在界面上显示，而其它用户甚至管理员也不能看到这个值。

变量配置完成以后，就可以通过图形界面驱动计划和实施了：

* 手动触发plan

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220309182910.png)

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220309190141.png)

* 测试GitHub提交PR出发

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220310165855.png)

此刻terraform自动就回去执行。

## 五 注意事项

* 可以在workspace设置terraform的版本：

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220310170033.png)

* tencent的provider需要指明source地址

```shell
provider "tencentcloud" {
  secret_id  = ""
  secret_key = ""
  region     = ""
}


terraform {
  required_providers {
    tencentcloud = {
      source  = "registry.terraform.io/tencentcloudstack/tencentcloud"

    }
  }

}
```





## 参考链接

* https://cloud.tencent.com/developer/article/1505878
* https://app.terraform.io/