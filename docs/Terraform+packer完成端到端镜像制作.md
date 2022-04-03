## Terraform+packer完成端到端基础设施编排

## 一 前言

Terraform总所周知，已成为云上基础设施编排的的行业标准，但其更多的关注点聚焦在支持的provider的各类资源编排，对于编排出来的基础设施，对其内部的配置变更，需要配合其他配置管理工具进行整合使用，以达到端到端的配置。

## 二 流程

Packer 是 HashiCorp 的开源工具，用于从源配置创建机器映像。您可以为您的特定用例配置带有操作系统和软件的 Packer 映像。

计算实例的 Terraform 配置可以使用 Packer 映像来配置您的实例，而无需手动配置。

## 三 场景

将服务器操作流程前置，通过packer将镜像制作出来，之后通过发布操作。



## 四 流程

* 本地生成ssh公钥私自钥，用于登录服务器
* 编排tencentcloud_key_pair 对象，用于后期登录服务器
* 编排cvm对象
* 编排null_resource 用于执行对服务器的操作
* 输出服务器详细信息

## 五 实战

### 5.1 编写TF代码

```shell

```

### 5.2 测试

```shell

```



![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220325134025.png)

## 六 其他



## 参考链接

* https://www.imooc.com/article/301275
* https://learn.hashicorp.com/tutorials/terraform/packer?in=terraform/provision
