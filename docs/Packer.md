# 镜像定义工具Packer实战

# 一 概述

## 1.1 什么是packer

Packer是用一个配置文件，在多种云计算平台上创建完全一致镜像的开源工具。Packer是由[HashiCorp](https://www.hashicorp.com/)在2013年左右推出的。Packer可以在各种主流操作系统上运行，可以高速、并行在多种云平台上创建镜像。Packer并不能取代puppet或者Chef之类的主机配置工具，而是互为补充-Packer在创建镜像时，可以调用这些工具在基础镜像上安装、配置软件。

镜像是指一个预先安装了软件、配置好了的操作系统，这个镜像可以被用开快速创建云主机。每个云平台通常有不同的镜像格式。

## 1.2 为什么使用packer

Packer只是一个命令行工具，易于通过终端使用，也可以很简单的放到自动化工具里边，用来自动创建任何类型的主机镜像。

Packer是一个开放源代码工具，可从一个源配置为多个平台创建相同的机器映像。Packer轻巧，可在每个主要操作系统上运行，并且性能卓越，可为多个平台并行创建机器映像。Packer不会取代Chef或Puppet等配置管理。实际上，在构建映像时，Packer能够使用Chef或Puppet等工具在映像上安装软件。机器映像是一个静态单元，其中包含一个预先配置的操作系统和已安装的软件，用于快速创建新的运行中的机器。机器映像格式随每个平台而变化。一些示例包括用于EC2的AMI，用于VMware的VMDK / VMX文件，用于VirtualBox的OVF导出等。

## 1.3 基础设施即代码

基础设施即代码的意思是把基础设施的实现方式写成代码。告别手工配置各类资源而采用基础设施即代码的方式，可以获得多种好处：

- 可重复-可以随时重新创建基础架构，例如在灾备环境重新创建生产环境。
- 可一致性-在各个不同的环境之间实现最大程度的一致性，减少环境差异导致的生产、测试效率下降。
- 可追溯回滚-环境的所有变化都通过代码实现，而代码的变化是可以追溯回滚的。
- 快速-由于基础架构是通过代码实现的，那么改变或者重建系统时就会非常快，是敏捷开发和DevOps中必不可少的一步。

[Packer](https://www.packer.io/)可以说是基础设施即代码的第一步。

在云原生架构中，技术设施即代码与微服务/声明式的API/服务网格成为架构的重要组成部分

## 1.4 packer优点

* 快速的基础架构实施：Packer创建的镜像可以让运维人员几秒钟内创建一个预先配置好的云主机，而不是几分钟甚至几个小时。这一好处不但对生产环境，也对测试及开发环境有益。

* 多平台兼容：由于Packer可以为多个平台创建完全一致的镜像，你可以在腾讯云上运行生产环境，在自己的私有云里边运行测试环境，在自己的vmware虚拟机里运行开发环境。而每个环境都完全一致。

* 提高稳定性：Packer在创建镜像时安装并配置软件。安装或者配置步骤中的问题会在这时候发现，而不是在穿件云主机的时候。

* 提高可测试性：镜像创建完成后，可以很快的启动一个云主机并对其进行测试。如果测试通过，那就可以相信，通过这个镜像创建的云主机都会通过测试。

## 1.5 packer与传统创建镜像对比

| 名称       | 控制台创建镜像                                               | Packer 创建镜像                                              |
| :--------- | :----------------------------------------------------------- | ------------------------------------------------------------ |
| 使用方式   | 控制台点击                                                   | 使用配置文件构建                                             |
| 可复用性   | 低。每次都需执行相同操作，且无法保证镜像一致性，无法版本化管理 | 高。配置文件拷贝、修改即可，可版本化管理                     |
| 操作复杂度 | 高。先使用基础镜像创建云主机，并自行登陆到云主机中进行部署，之后手动制作镜像 | 低。执行配置文件，自动执行预配置自动化脚本，然后自动构建镜像 |
| 创建时间   | 长。流程化操作，需要人工值守，无法准确等待每个流程的执行     | 短。自动化工作流操作，完善的轮询等待机制，无缝衔接每个流程   |

# 二 应用场景

## 2.1 持续交付

Packer是用命令行驱动的，而且不需要很多资源。因此可以很容易把Packer放到持续交付的pipeline里，创建镜像。在持续交付随后的步骤中，可以对此镜像进行测试，看基础设施的变化是不是能够通过测试。如果测试通过，那就可以相信在这一镜像基础上搭建的云主机也会工作。这对基础设施变化的稳定性和可测试性提供了基础。

Packer是轻量级的、可移植的和命令行驱动的。这使得它成为一个完美的工具，放在您的连续交付管道中间。Packer可用于在对Chef/Puppet的每次更改上为多个平台生成新的机器映像。

作为这个管道的一部分，可以启动并测试新创建的映像，从而验证基础结构更改的工作。如果测试通过，您可以确信映像在部署时将工作。这为基础设施的变化带来了新的稳定性和可测试性。

## 2.2 dev/prod 产品比价

Packer帮助保持开发，暂存和生产尽可能相似。Packer可用于同时为多个平台生成映像。因此，如果将AWS用于生产而将VMware（也许与Vagrant结合使用）用于开发，则可以使用Packer从同一模板同时生成AMI和VMware计算机。

## 2.3 设备/演示创建

由于Packer可以并行地为多个平台创建一致的映像，因此非常适合创建电器和一次性产品演示。随着软件的更改，您可以自动创建带有预安装软件的设备。然后，潜在的用户可以通过将软件部署到他们选择的环境中来开始使用您的软件。打包具有复杂要求的软件从未如此简单。

# 三 安装

## 3.1 二进制安装

选择合适的二进制文件进行安装

https://www.packer.io/downloads.html

```
wget https://releases.hashicorp.com/packer/1.5.1/packer_1.5.1_linux_amd64.zip
unzip packer_1.5.1_linux_amd64.zip
cp packer /usr/bin/
```

## 3.2 源码编译安装	

### 3.2.1 go环境安装

此处示例为CentOS7

```shell
cd /opt
wget -c https://dl.google.com/go/go1.13.6.linux-amd64.tar.gz
tar -C /usr/local -zxvf  go1.13.6.linux-amd64.tar.gz
cat >>/etc/profile <<EOF	
export GOROOT=/usr/local/go
export PATH=\$PATH:\$GOROOT/bin
EOF
source /etc/profile

vim .bash_profile

export GOPATH=$HOME/go
```

![image-20200125093935368](/Users/xuel/Library/Application Support/typora-user-images/image-20200125093935368.png)

### 3.2.2 packer编译

```shell
$ mkdir -p $(go env GOPATH)/src/github.com/hashicorp && cd $_
$ git clone https://github.com/hashicorp/packer.git
$ cd packer

$ make dev
$ cp $(go env GOPATH)/bin/packer /usr/sbin
```

# 四 示例

## 4.1 腾讯云示例

### 4.1.1 编写template

```
{
  "variables": {
    "tc_secret_id": "{{env `TENCENTCLOUD_ACCESS_KEY`}}",
    "tc_secret_key": "{{env `TENCENTCLOUD_SECRET_KEY`}}"
  },		
  "builders": [
    {
    "type": "tencentcloud-cvm",
    "secret_id": "{{user `tc_secret_id`	}}",
    "secret_key": "{{user `tc_secret_key`}}",
    "region": "ap-guangzhou",
    "zone": "ap-guangzhou-3",
    "instance_type": "S2.SMALL1",	
    "disk_type": "CLOUD_PREMIUM",
    "associate_public_ip_address": true,
    "source_image_id": "img-9qabwvbn",
    "ssh_username" : "root",
    "image_name": "nginx-service-v1"
  }
  ],
  "provisioners": [{
    "type": "shell",
    "inline": [
      "rpm -ivh http://nginx.org/packages/centos/7/noarch/RPMS/nginx-release-centos-7-0.el7.ngx.noarch.rpm",
      "yum install -y nginx",
      "systemctl enable nginx",
      "systemctl start nginx"
    ]
  }]
}
```

这段代码一开始，引入了两个变量，分别是tc_secret_id和tc_secret_key，它们的值分别取自环境变量TENCENTCLOUD_ACCESS_KEY和TENCENTCLOUD_SECRET_KEY，这与[腾讯云Terraform应用指南](https://cloud.tencent.com/developer/article/1473713?from=10680)中提到设置是一样的。

随后紧跟的是一个builder，这个例子中指定在腾讯云广州大区创建一个自定义镜像nginx-service-v1，该镜像的基础镜像是腾讯云的共有镜像img-9qabwvbn，这个镜像id是从腾讯云控制台查到的，如下图：

builder后边是一个provisioner的定义，指定了几个命令行及参数。

### 4.1.2 执行

随后可以执行：



```js
TENCENTCLOUD_ACCESS_KEY=AKIDZyGQXbErpY4MPDl7D4g3HH2c5KL8Y8G8
TENCENTCLOUD_SECRET_KEY=kFUTDk38yZw4xc5JHzRdZFfspWxDE0Xq
# 验证文件正确性
[root@VM_0_17_centos data]# packer validate cvm-template.json
Template validated successfully.

# 执行
export TENCENTCLOUD_ACCESS_KEY=AKIDZyGQXbErpY4MPDl7D4g3HH2c5KL8Y8G8
export TENCENTCLOUD_SECRET_KEY=kFUTDk38yZw4xc5JHzRdZFfspWxDE0Xq
packer build tencent-nginx.json
```

### 4.1.3 查看执行结果

```shell
[root@VM_0_17_centos data]# packer build cvm-template.json
tencentcloud-cvm: output will be in this color.

==> tencentcloud-cvm: Trying to check image name: nginx-service-v1...
    tencentcloud-cvm: Image name: useable
==> tencentcloud-cvm: Trying to check source image: img-9qabwvbn...
    tencentcloud-cvm: Image found: CentOS 7.6 64bit
==> tencentcloud-cvm: Trying to create a new keypair: packer_5e2bb832...
    tencentcloud-cvm: Keypair created: skey-ae69a70z
==> tencentcloud-cvm: Trying to create a new vpc...
    tencentcloud-cvm: Vpc created: vpc-b4ofbh47
==> tencentcloud-cvm: Trying to create a new subnet...
    tencentcloud-cvm: Subnet created: subnet-5817hc7k
==> tencentcloud-cvm: Trying to create a new securitygroup...
    tencentcloud-cvm: Securitygroup created: sg-d9p13qth
==> tencentcloud-cvm: Trying to create securitygroup polices...
    tencentcloud-cvm: Securitygroup polices created
==> tencentcloud-cvm: Trying to create a new instance...
    tencentcloud-cvm: Waiting for instance ready
    tencentcloud-cvm: Instance created: ins-389nxgpa
==> tencentcloud-cvm: Using ssh communicator to connect: 134.175.96.85
==> tencentcloud-cvm: Waiting for SSH to become available...
==> tencentcloud-cvm: Connected to SSH!
==> tencentcloud-cvm: Provisioning with shell script: /tmp/packer-shell361573813
==> tencentcloud-cvm: warning: /var/tmp/rpm-tmp.s33FeG: Header V4 RSA/SHA1 Signature, key ID 7bd9bf62: NOKEY
    tencentcloud-cvm: Retrieving http://nginx.org/packages/centos/7/noarch/RPMS/nginx-release-centos-7-0.el7.ngx.noarch.rpm
    tencentcloud-cvm: Preparing...                          ########################################
    tencentcloud-cvm: Updating / installing...
    tencentcloud-cvm: nginx-release-centos-7-0.el7.ngx      ########################################
    tencentcloud-cvm: Loaded plugins: fastestmirror, langpacks
    tencentcloud-cvm: Determining fastest mirrors
    tencentcloud-cvm: Resolving Dependencies
    tencentcloud-cvm: --> Running transaction check
    tencentcloud-cvm: ---> Package nginx.x86_64 1:1.16.1-1.el7.ngx will be installed
    tencentcloud-cvm: --> Finished Dependency Resolution
    tencentcloud-cvm:
    tencentcloud-cvm: Dependencies Resolved
    tencentcloud-cvm:
    tencentcloud-cvm: ================================================================================
    tencentcloud-cvm:  Package       Arch           Version                       Repository     Size
    tencentcloud-cvm: ================================================================================
    tencentcloud-cvm: Installing:
    tencentcloud-cvm:  nginx         x86_64         1:1.16.1-1.el7.ngx            nginx         766 k
    tencentcloud-cvm:
    tencentcloud-cvm: Transaction Summary
    tencentcloud-cvm: ================================================================================
    tencentcloud-cvm: Install  1 Package
    tencentcloud-cvm:
    tencentcloud-cvm: Total download size: 766 k
    tencentcloud-cvm: Installed size: 2.7 M
    tencentcloud-cvm: Downloading packages:
    tencentcloud-cvm: Running transaction check
    tencentcloud-cvm: Running transaction test
    tencentcloud-cvm: Transaction test succeeded
    tencentcloud-cvm: Running transaction
    tencentcloud-cvm:   Installing : 1:nginx-1.16.1-1.el7.ngx.x86_64                              1/1
    tencentcloud-cvm: ----------------------------------------------------------------------
    tencentcloud-cvm:
    tencentcloud-cvm: Thanks for using nginx!
    tencentcloud-cvm:
    tencentcloud-cvm: Please find the official documentation for nginx here:
    tencentcloud-cvm: * http://nginx.org/en/docs/
    tencentcloud-cvm:
    tencentcloud-cvm: Please subscribe to nginx-announce mailing list to get
    tencentcloud-cvm: the most important news about nginx:
    tencentcloud-cvm: * http://nginx.org/en/support.html
    tencentcloud-cvm:
    tencentcloud-cvm: Commercial subscriptions for nginx are available on:
    tencentcloud-cvm: * http://nginx.com/products/
    tencentcloud-cvm:
    tencentcloud-cvm: ----------------------------------------------------------------------
    tencentcloud-cvm:   Verifying  : 1:nginx-1.16.1-1.el7.ngx.x86_64                              1/1
    tencentcloud-cvm:
    tencentcloud-cvm: Installed:
    tencentcloud-cvm:   nginx.x86_64 1:1.16.1-1.el7.ngx
    tencentcloud-cvm:
    tencentcloud-cvm: Complete!
==> tencentcloud-cvm: Created symlink from /etc/systemd/system/multi-user.target.wants/nginx.service to /usr/lib/systemd/system/nginx.service.
==> tencentcloud-cvm: Trying to detach keypair: skey-ae69a70z...
    tencentcloud-cvm: Keypair detached
==> tencentcloud-cvm: Trying to create a new image: nginx-service-v1...
    tencentcloud-cvm: Waiting for image ready
    tencentcloud-cvm: Image created: img-lw1glokq
==> tencentcloud-cvm: Cleaning up instance...
==> tencentcloud-cvm: Cleaning up securitygroup...
==> tencentcloud-cvm: Cleaning up subnet...
==> tencentcloud-cvm: Cleaning up vpc...
==> tencentcloud-cvm: Cleaning up keypair...
Build 'tencentcloud-cvm' finished.

==> Builds finished. The artifacts of successful builds are:
--> tencentcloud-cvm: Tencentcloud images(ap-guangzhou: img-lw1glokq) were created.
```

### 4.1.3 查看执行结果

![image-20200125114003541](/Users/xuel/Library/Application Support/typora-user-images/image-20200125114003541.png)



![image-20200125114042888](/Users/xuel/Library/Application Support/typora-user-images/image-20200125114042888.png)

![image-20200125114415044](/Users/xuel/Library/Application Support/typora-user-images/image-20200125114415044.png)

![image-20200125114435643](/Users/xuel/Library/Application Support/typora-user-images/image-20200125114435643.png)



# 五 反思

* 在基础架构向kubernetes微服务架构的转型中，如果架构处于cloud hosting，或混合云架构，Packer为您简化持续交付流程，快速的镜像测试。
* Packer 创建镜像对比人工手动点击，以其基础设施即代码，代码可复用性，版本管理，自动化流程管理等。

  



# 参考链接

* https://www.packer.io/intro/index.html
* https://cloud.tencent.com/developer/article/1474736
* https://www.mdslq.cn/archives/bc880644.html