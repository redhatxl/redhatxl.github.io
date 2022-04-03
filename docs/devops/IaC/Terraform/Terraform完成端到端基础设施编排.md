## Terraform完成端到端基础设施编排

## 一 前言

Terraform总所周知，已成为云上基础设施编排的的行业标准，但其更多的关注点聚焦在支持的provider的各类资源编排，对于编排出来的基础设施，对其内部的配置变更，需要配合其他配置管理工具进行整合使用，以达到端到端的配置。

## 二 流程

Terraform利用资源null_resource 和内部属性provisioner 的 remote-exec 类型对编排出来的基础设施进行配置管理，完成自定义脚本执行，从而达到端到端的资源编排和配置管理。

## 三 场景

利用terraform编排出腾讯云服务器，并在服务器内部进行基础设施安装

## 四 流程

* 本地生成ssh公钥私自钥，用于登录服务器
* 编排tencentcloud_key_pair 对象，用于后期登录服务器
* 编排cvm对象
* 编排null_resource 用于执行对服务器的操作
* 输出服务器详细信息

## 五 实战

### 5.1 生成ssh密钥

```shell
ssh-keygen -t rsa -P "" -f xuel_rsa -q 
```

### 5.2 编写TF代码

```shell
locals {
  cidr_block_vpc_name    = "10.0.0.0/16"
  cidr_block_subnet_name = "10.0.1.0/24"
  count                  = 2
}

// 用于查找惊喜
data "tencentcloud_images" "image" {
  image_type = ["PUBLIC_IMAGE"]
  os_name    = "centos 7.5"
}

// 查找实例类型
data "tencentcloud_instance_types" "instanceType" {
  cpu_core_count = 1
  memory_size    = 1
  filter {
    name   = "instance-family"
    values = ["S3"]
  }
}

// 查找可用zone
data "tencentcloud_availability_zones" "zone" {}

// 定义vpc资源
resource "tencentcloud_vpc" "vpc" {
  cidr_block = local.cidr_block_vpc_name
  name       = "xuel_tf_vpc"
}

// 定义子网资源
resource "tencentcloud_subnet" "subnet" {
  availability_zone = data.tencentcloud_availability_zones.zone.zones.0.name
  cidr_block        = local.cidr_block_subnet_name
  name              = "xuel_tf_subnet"
  vpc_id            = tencentcloud_vpc.vpc.id
}

// 定义公钥对
resource "tencentcloud_key_pair" "xuel-tf-key" {
  key_name   = "xueltfkey"
  public_key = "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDCcJxzfjqQDikRRShExu22kvNOfMKeWrn++s5nQMTm+9SNsQISEk+P15JhFVachgjumupMcOIrfJQAAQcnzNmxoRCTCJQfmegGpDZVpE1cyHUhGMA5kwu67PmK1Snm8hkg2Xzlduhr1xysL2mRn3+6o5tsFXhGrYOcSSXnf5SpTPgMjqo339ksH0iv8kvu3NaZRueygLYaVEMjixJvsnUisL3uY8LQ+4cm2Zu5mdQamhWhN0kkSdlfbjPgzxexL4AglD9YDy4I9Q80vKzy33Ubwo17a2aNCF3uPpYvCKiV0H9z2XtMxisKDfsQQA01Q1vpccUIK6L48xSbers8hV2xxpSEWzEuoZg18eG2ikAencA6mhGjFWcp9A1dllY2rUhcEdrjcjXji1SGabjYLsv23ki6EMGjM/AK+fq+vj3pIPUMpscX3xVDGmz/zusq6v1KfOtQw7B/Dg8c2cxKUlEWZqqC3A7rt3JO/RVEbeqSe5mlRm2yngINVemmhkcfZNs= xuel@kaliarchmacbookpro"
}

// 定义cvm实例
resource "tencentcloud_instance" "xuel_tf_ansible" {
  availability_zone = data.tencentcloud_availability_zones.zone.zones[0].name
  image_id          = data.tencentcloud_images.image.images[0].image_id
  instance_type     = data.tencentcloud_instance_types.instanceType.instance_types[0].instance_type
  vpc_id            = tencentcloud_vpc.vpc.id
  subnet_id         = tencentcloud_subnet.subnet.id
  //  allocate_public_ip = true
  system_disk_size = 50
  system_disk_type = "CLOUD_PREMIUM"
  key_name         = tencentcloud_key_pair.xuel-tf-key.id
  allocate_public_ip = true
  internet_max_bandwidth_out = 2

  data_disks {
    data_disk_size = 50
    data_disk_type = "CLOUD_PREMIUM"
  }
  hostname             = format("xuel-tf-server-%d", count.index)
  instance_charge_type = "POSTPAID_BY_HOUR"
  count                = local.count
  instance_name        = format("xuel-tf-server-%d", count.index)

  tags = {
    tagkey = "xueltag"
  }

}

// 定义后期shell 执行
resource "null_resource" "shell" {
  count = local.count
	// 定义triggers
  triggers = {
    instance_ids = element(tencentcloud_instance.xuel_tf_ansible.*.id, count.index)
  }
  
  // 定义执行者
  provisioner "remote-exec" {
    connection {
      host        = element(tencentcloud_instance.xuel_tf_ansible.*.public_ip, count.index)
      type        = "ssh"
      user        = "root"
      private_key = file("/Users/xuel/.ssh/id_rsa")

    }
		// 定义内部执行脚本
    inline = [
      "echo index.html > /xueltf.txt",
      "yum -y install nginx"
    ]
  }
}

// 定义输出
output "summary" {
  value = {
    //    image = {for k,v in data.tencentcloud_images.image:k=>v}
    //    instanceType = {for k,v in data.tencentcloud_instance_types
    //    .instanceType:k=>v}
    //    zone = {for k,v in data.tencentcloud_availability_zones.zone: k=>v}
    instance = { for k, v in tencentcloud_instance.xuel_tf_ansible : k => v }
  }
}
```

### 5.3 测试

```shell
$ terraform apply                                                   

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # null_resource.shell[0] will be created
  + resource "null_resource" "shell" {
      + id       = (known after apply)
      + triggers = (known after apply)
    }

  # null_resource.shell[1] will be created
  + resource "null_resource" "shell" {
      + id       = (known after apply)
      + triggers = (known after apply)
    }

  # tencentcloud_instance.xuel_tf_ansible[0] will be created
  + resource "tencentcloud_instance" "xuel_tf_ansible" {
      + allocate_public_ip                      = true
      + availability_zone                       = "ap-guangzhou-3"
      + create_time                             = (known after apply)
      + disable_monitor_service                 = false
      + disable_security_service                = false
      + expired_time                            = (known after apply)
      + force_delete                            = false
      + hostname                                = "xuel-tf-server-0"
      + id                                      = (known after apply)
      + image_id                                = "img-oikl1tzv"
      + instance_charge_type                    = "POSTPAID_BY_HOUR"
      + instance_charge_type_prepaid_renew_flag = (known after apply)
      + instance_name                           = "xuel-tf-server-0"
      + instance_status                         = (known after apply)
      + instance_type                           = "S3.SMALL1"
      + internet_charge_type                    = (known after apply)
      + internet_max_bandwidth_out              = 2
      + key_name                                = (known after apply)
      + private_ip                              = (known after apply)
      + project_id                              = 0
      + public_ip                               = (known after apply)
      + running_flag                            = true
      + security_groups                         = (known after apply)
      + subnet_id                               = (known after apply)
      + system_disk_id                          = (known after apply)
      + system_disk_size                        = 50
      + system_disk_type                        = "CLOUD_PREMIUM"
      + tags                                    = {
          + "tagkey" = "xueltag"
        }
      + vpc_id                                  = (known after apply)

      + data_disks {
          + data_disk_id           = (known after apply)
          + data_disk_size         = 50
          + data_disk_type         = "CLOUD_PREMIUM"
          + delete_with_instance   = true
          + encrypt                = false
          + throughput_performance = 0
        }
    }

  # tencentcloud_instance.xuel_tf_ansible[1] will be created
  + resource "tencentcloud_instance" "xuel_tf_ansible" {
      + allocate_public_ip                      = true
      + availability_zone                       = "ap-guangzhou-3"
      + create_time                             = (known after apply)
      + disable_monitor_service                 = false
      + disable_security_service                = false
      + expired_time                            = (known after apply)
      + force_delete                            = false
      + hostname                                = "xuel-tf-server-1"
      + id                                      = (known after apply)
      + image_id                                = "img-oikl1tzv"
      + instance_charge_type                    = "POSTPAID_BY_HOUR"
      + instance_charge_type_prepaid_renew_flag = (known after apply)
      + instance_name                           = "xuel-tf-server-1"
      + instance_status                         = (known after apply)
      + instance_type                           = "S3.SMALL1"
      + internet_charge_type                    = (known after apply)
      + internet_max_bandwidth_out              = 2
      + key_name                                = (known after apply)
      + private_ip                              = (known after apply)
      + project_id                              = 0
      + public_ip                               = (known after apply)
      + running_flag                            = true
      + security_groups                         = (known after apply)
      + subnet_id                               = (known after apply)
      + system_disk_id                          = (known after apply)
      + system_disk_size                        = 50
      + system_disk_type                        = "CLOUD_PREMIUM"
      + tags                                    = {
          + "tagkey" = "xueltag"
        }
      + vpc_id                                  = (known after apply)

      + data_disks {
          + data_disk_id           = (known after apply)
          + data_disk_size         = 50
          + data_disk_type         = "CLOUD_PREMIUM"
          + delete_with_instance   = true
          + encrypt                = false
          + throughput_performance = 0
        }
    }

  # tencentcloud_key_pair.xuel-tf-key will be created
  + resource "tencentcloud_key_pair" "xuel-tf-key" {
      + id         = (known after apply)
      + key_name   = "xueltfkey"
      + project_id = 0
      + public_key = "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDCcJxzfjqQDikRRShExu22kvNOfMKeWrn++s5nQMTm+9SNsQISEk+P15JhFVachgjumupMcOIrfJQAAQcnzNmxoRCTCJQfmegGpDZVpE1cyHUhGMA5kwu67PmK1Snm8hkg2Xzlduhr1xysL2mRn3+6o5tsFXhGrYOcSSXnf5SpTPgMjqo339ksH0iv8kvu3NaZRueygLYaVEMjixJvsnUisL3uY8LQ+4cm2Zu5mdQamhWhN0kkSdlfbjPgzxexL4AglD9YDy4I9Q80vKzy33Ubwo17a2aNCF3uPpYvCKiV0H9z2XtMxisKDfsQQA01Q1vpccUIK6L48xSbers8hV2xxpSEWzEuoZg18eG2ikAencA6mhGjFWcp9A1dllY2rUhcEdrjcjXji1SGabjYLsv23ki6EMGjM/AK+fq+vj3pIPUMpscX3xVDGmz/zusq6v1KfOtQw7B/Dg8c2cxKUlEWZqqC3A7rt3JO/RVEbeqSe5mlRm2yngINVemmhkcfZNs="
    }

  # tencentcloud_subnet.subnet will be created
  + resource "tencentcloud_subnet" "subnet" {
      + availability_zone  = "ap-guangzhou-3"
      + available_ip_count = (known after apply)
      + cidr_block         = "10.0.1.0/24"
      + create_time        = (known after apply)
      + id                 = (known after apply)
      + is_default         = (known after apply)
      + is_multicast       = true
      + name               = "xuel_tf_subnet"
      + route_table_id     = (known after apply)
      + vpc_id             = (known after apply)
    }

  # tencentcloud_vpc.vpc will be created
  + resource "tencentcloud_vpc" "vpc" {
      + assistant_cidrs        = (known after apply)
      + cidr_block             = "10.0.0.0/16"
      + create_time            = (known after apply)
      + default_route_table_id = (known after apply)
      + dns_servers            = (known after apply)
      + id                     = (known after apply)
      + is_default             = (known after apply)
      + is_multicast           = true
      + name                   = "xuel_tf_vpc"
    }

Plan: 7 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  + summary = {
      + instance = {
          + 0 = {
              + allocate_public_ip                      = true
              + availability_zone                       = "ap-guangzhou-3"
              + bandwidth_package_id                    = null
              + cam_role_name                           = null
              + cdh_host_id                             = null
              + cdh_instance_type                       = null
              + create_time                             = (known after apply)
              + data_disks                              = [
                  + {
                      + data_disk_id           = (known after apply)
                      + data_disk_size         = 50
                      + data_disk_snapshot_id  = null
                      + data_disk_type         = "CLOUD_PREMIUM"
                      + delete_with_instance   = true
                      + encrypt                = false
                      + throughput_performance = 0
                    },
                ]
              + disable_monitor_service                 = false
              + disable_security_service                = false
              + expired_time                            = (known after apply)
              + force_delete                            = false
              + hostname                                = "xuel-tf-server-0"
              + id                                      = (known after apply)
              + image_id                                = "img-oikl1tzv"
              + instance_charge_type                    = "POSTPAID_BY_HOUR"
              + instance_charge_type_prepaid_period     = null
              + instance_charge_type_prepaid_renew_flag = (known after apply)
              + instance_count                          = null
              + instance_name                           = "xuel-tf-server-0"
              + instance_status                         = (known after apply)
              + instance_type                           = "S3.SMALL1"
              + internet_charge_type                    = (known after apply)
              + internet_max_bandwidth_out              = 2
              + keep_image_login                        = null
              + key_name                                = (known after apply)
              + password                                = null
              + placement_group_id                      = null
              + private_ip                              = (known after apply)
              + project_id                              = 0
              + public_ip                               = (known after apply)
              + running_flag                            = true
              + security_groups                         = (known after apply)
              + spot_instance_type                      = null
              + spot_max_price                          = null
              + stopped_mode                            = null
              + subnet_id                               = (known after apply)
              + system_disk_id                          = (known after apply)
              + system_disk_size                        = 50
              + system_disk_type                        = "CLOUD_PREMIUM"
              + tags                                    = {
                  + "tagkey" = "xueltag"
                }
              + user_data                               = null
              + user_data_raw                           = null
              + vpc_id                                  = (known after apply)
            }
          + 1 = {
              + allocate_public_ip                      = true
              + availability_zone                       = "ap-guangzhou-3"
              + bandwidth_package_id                    = null
              + cam_role_name                           = null
              + cdh_host_id                             = null
              + cdh_instance_type                       = null
              + create_time                             = (known after apply)
              + data_disks                              = [
                  + {
                      + data_disk_id           = (known after apply)
                      + data_disk_size         = 50
                      + data_disk_snapshot_id  = null
                      + data_disk_type         = "CLOUD_PREMIUM"
                      + delete_with_instance   = true
                      + encrypt                = false
                      + throughput_performance = 0
                    },
                ]
              + disable_monitor_service                 = false
              + disable_security_service                = false
              + expired_time                            = (known after apply)
              + force_delete                            = false
              + hostname                                = "xuel-tf-server-1"
              + id                                      = (known after apply)
              + image_id                                = "img-oikl1tzv"
              + instance_charge_type                    = "POSTPAID_BY_HOUR"
              + instance_charge_type_prepaid_period     = null
              + instance_charge_type_prepaid_renew_flag = (known after apply)
              + instance_count                          = null
              + instance_name                           = "xuel-tf-server-1"
              + instance_status                         = (known after apply)
              + instance_type                           = "S3.SMALL1"
              + internet_charge_type                    = (known after apply)
              + internet_max_bandwidth_out              = 2
              + keep_image_login                        = null
              + key_name                                = (known after apply)
              + password                                = null
              + placement_group_id                      = null
              + private_ip                              = (known after apply)
              + project_id                              = 0
              + public_ip                               = (known after apply)
              + running_flag                            = true
              + security_groups                         = (known after apply)
              + spot_instance_type                      = null
              + spot_max_price                          = null
              + stopped_mode                            = null
              + subnet_id                               = (known after apply)
              + system_disk_id                          = (known after apply)
              + system_disk_size                        = 50
              + system_disk_type                        = "CLOUD_PREMIUM"
              + tags                                    = {
                  + "tagkey" = "xueltag"
                }
              + user_data                               = null
              + user_data_raw                           = null
              + vpc_id                                  = (known after apply)
            }
        }
    }


Warning: This data source will been deprecated in Terraform TencentCloud provider later version. Please use `tencentcloud_availability_zones_by_product` instead.

  on main.tf line 21, in data "tencentcloud_availability_zones" "zone":
  21: data "tencentcloud_availability_zones" "zone" {}


Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes


tencentcloud_vpc.vpc: Creating...
tencentcloud_key_pair.xuel-tf-key: Creating...
tencentcloud_key_pair.xuel-tf-key: Creation complete after 2s [id=skey-ermat2er]
tencentcloud_vpc.vpc: Creation complete after 5s [id=vpc-gu4gmx1h]
tencentcloud_subnet.subnet: Creating...
tencentcloud_subnet.subnet: Creation complete after 2s [id=subnet-079md4re]
tencentcloud_instance.xuel_tf_ansible[1]: Creating...
tencentcloud_instance.xuel_tf_ansible[0]: Creating...
tencentcloud_instance.xuel_tf_ansible[0]: Still creating... [10s elapsed]
tencentcloud_instance.xuel_tf_ansible[1]: Still creating... [10s elapsed]
tencentcloud_instance.xuel_tf_ansible[1]: Still creating... [20s elapsed]
tencentcloud_instance.xuel_tf_ansible[0]: Still creating... [20s elapsed]
tencentcloud_instance.xuel_tf_ansible[1]: Creation complete after 24s [id=ins-562f4vl0]
tencentcloud_instance.xuel_tf_ansible[0]: Creation complete after 25s [id=ins-ifzhv2g6]
null_resource.shell[0]: Creating...
null_resource.shell[1]: Creating...
null_resource.shell[0]: Provisioning with 'remote-exec'...
null_resource.shell[1]: Provisioning with 'remote-exec'...
null_resource.shell[0] (remote-exec): Connecting to remote host via SSH...
null_resource.shell[0] (remote-exec):   Host: 175.178.92.230
null_resource.shell[0] (remote-exec):   User: root
null_resource.shell[0] (remote-exec):   Password: false
null_resource.shell[0] (remote-exec):   Private key: true
null_resource.shell[0] (remote-exec):   Certificate: false
null_resource.shell[0] (remote-exec):   SSH Agent: true
null_resource.shell[0] (remote-exec):   Checking Host Key: false
null_resource.shell[1] (remote-exec): Connecting to remote host via SSH...
null_resource.shell[1] (remote-exec):   Host: 175.178.227.93
null_resource.shell[1] (remote-exec):   User: root
null_resource.shell[1] (remote-exec):   Password: false
null_resource.shell[1] (remote-exec):   Private key: true
null_resource.shell[1] (remote-exec):   Certificate: false
null_resource.shell[1] (remote-exec):   SSH Agent: true
null_resource.shell[1] (remote-exec):   Checking Host Key: false
null_resource.shell[1] (remote-exec): Connected!
null_resource.shell[0] (remote-exec): Connecting to remote host via SSH...
null_resource.shell[0] (remote-exec):   Host: 175.178.92.230
null_resource.shell[0] (remote-exec):   User: root
null_resource.shell[0] (remote-exec):   Password: false
null_resource.shell[0] (remote-exec):   Private key: true
null_resource.shell[0] (remote-exec):   Certificate: false
null_resource.shell[0] (remote-exec):   SSH Agent: true
null_resource.shell[0] (remote-exec):   Checking Host Key: false
null_resource.shell[1] (remote-exec): Loaded plugins: fastestmirror, langpacks
null_resource.shell[1] (remote-exec): Determining fastest mirrors
null_resource.shell[1] (remote-exec): epel             | 4.7 kB     00:00
null_resource.shell[1] (remote-exec): extras           | 2.9 kB     00:00
null_resource.shell[1] (remote-exec): os               | 3.6 kB     00:00
null_resource.shell[1] (remote-exec): updates          | 2.9 kB     00:00
null_resource.shell[0] (remote-exec): Connected!
null_resource.shell[1] (remote-exec): (1/7): epel/7/x86_ |  96 kB   00:00
null_resource.shell[1] (remote-exec): (2/7): os/7/x86_64 | 153 kB   00:00
null_resource.shell[1] (remote-exec): (3/7): extras/7/x8 | 246 kB   00:00
null_resource.shell[1] (remote-exec): (5/7): epel/7/x 3% | 1.1 MB   --:-- ETA
null_resource.shell[1] (remote-exec): (4/7): epel/7/x86_ | 1.1 MB   00:00
null_resource.shell[1] (remote-exec): (5/7): epel/7/x86_ | 7.0 MB   00:00
null_resource.shell[1] (remote-exec): (6/7): os/7/x86_64 | 6.1 MB   00:00
null_resource.shell[0] (remote-exec): Loaded plugins: fastestmirror, langpacks
null_resource.shell[0] (remote-exec): Determining fastest mirrors
null_resource.shell[0] (remote-exec): epel             | 4.7 kB     00:00
null_resource.shell[1] (remote-exec): (7/7): updates/7/x |  14 MB   00:00
null_resource.shell[0] (remote-exec): extras           | 2.9 kB     00:00
null_resource.shell[0] (remote-exec): os               | 3.6 kB     00:00
null_resource.shell[0] (remote-exec): updates          | 2.9 kB     00:00
null_resource.shell[0] (remote-exec): (1/7): epel/7/x86_ |  96 kB   00:00
null_resource.shell[0]: Still creating... [10s elapsed]
null_resource.shell[1]: Still creating... [10s elapsed]
null_resource.shell[0] (remote-exec): (2/7): extras/7/x8 | 246 kB   00:00
null_resource.shell[0] (remote-exec): (4/7): epel/7/x 3% | 1.1 MB   --:-- ETA
null_resource.shell[0] (remote-exec): (3/7): os/7/x86_64 | 153 kB   00:00
null_resource.shell[0] (remote-exec): (4/7): epel/7/x86_ | 1.1 MB   00:00
null_resource.shell[0] (remote-exec): (5/7): epel/7/x86_ | 7.0 MB   00:00
null_resource.shell[0] (remote-exec): (7/7): updates 51% |  15 MB   00:00 ETA
null_resource.shell[0] (remote-exec): (6/7): os/7/x86_64 | 6.1 MB   00:00
null_resource.shell[0] (remote-exec): (7/7): updates 89% |  26 MB   00:00 ETA
null_resource.shell[0] (remote-exec): (7/7): updates/7/x |  14 MB   00:00
null_resource.shell[1] (remote-exec): Resolving Dependencies
null_resource.shell[1] (remote-exec): --> Running transaction check
null_resource.shell[1] (remote-exec): ---> Package nginx.x86_64 1:1.20.1-9.el7 will be installed
null_resource.shell[0] (remote-exec): Resolving Dependencies
null_resource.shell[0] (remote-exec): --> Running transaction check
null_resource.shell[0] (remote-exec): ---> Package nginx.x86_64 1:1.20.1-9.el7 will be installed
null_resource.shell[1] (remote-exec): --> Processing Dependency: nginx-filesystem = 1:1.20.1-9.el7 for package: 1:nginx-1.20.1-9.el7.x86_64
null_resource.shell[0] (remote-exec): --> Processing Dependency: nginx-filesystem = 1:1.20.1-9.el7 for package: 1:nginx-1.20.1-9.el7.x86_64
null_resource.shell[1] (remote-exec): --> Processing Dependency: libcrypto.so.1.1(OPENSSL_1_1_0)(64bit) for package: 1:nginx-1.20.1-9.el7.x86_64
null_resource.shell[1] (remote-exec): --> Processing Dependency: libssl.so.1.1(OPENSSL_1_1_0)(64bit) for package: 1:nginx-1.20.1-9.el7.x86_64
null_resource.shell[1] (remote-exec): --> Processing Dependency: libssl.so.1.1(OPENSSL_1_1_1)(64bit) for package: 1:nginx-1.20.1-9.el7.x86_64
null_resource.shell[1] (remote-exec): --> Processing Dependency: nginx-filesystem for package: 1:nginx-1.20.1-9.el7.x86_64
null_resource.shell[1] (remote-exec): --> Processing Dependency: libcrypto.so.1.1()(64bit) for package: 1:nginx-1.20.1-9.el7.x86_64
null_resource.shell[1] (remote-exec): --> Processing Dependency: libprofiler.so.0()(64bit) for package: 1:nginx-1.20.1-9.el7.x86_64
null_resource.shell[1] (remote-exec): --> Processing Dependency: libssl.so.1.1()(64bit) for package: 1:nginx-1.20.1-9.el7.x86_64
null_resource.shell[1] (remote-exec): --> Running transaction check
null_resource.shell[1] (remote-exec): ---> Package gperftools-libs.x86_64 0:2.6.1-1.el7 will be installed
null_resource.shell[1] (remote-exec): ---> Package nginx-filesystem.noarch 1:1.20.1-9.el7 will be installed
null_resource.shell[1] (remote-exec): ---> Package openssl11-libs.x86_64 1:1.1.1k-2.el7 will be installed
null_resource.shell[0] (remote-exec): --> Processing Dependency: libcrypto.so.1.1(OPENSSL_1_1_0)(64bit) for package: 1:nginx-1.20.1-9.el7.x86_64
null_resource.shell[0] (remote-exec): --> Processing Dependency: libssl.so.1.1(OPENSSL_1_1_0)(64bit) for package: 1:nginx-1.20.1-9.el7.x86_64
null_resource.shell[0] (remote-exec): --> Processing Dependency: libssl.so.1.1(OPENSSL_1_1_1)(64bit) for package: 1:nginx-1.20.1-9.el7.x86_64
null_resource.shell[0] (remote-exec): --> Processing Dependency: nginx-filesystem for package: 1:nginx-1.20.1-9.el7.x86_64
null_resource.shell[0] (remote-exec): --> Processing Dependency: libcrypto.so.1.1()(64bit) for package: 1:nginx-1.20.1-9.el7.x86_64
null_resource.shell[0] (remote-exec): --> Processing Dependency: libprofiler.so.0()(64bit) for package: 1:nginx-1.20.1-9.el7.x86_64
null_resource.shell[0] (remote-exec): --> Processing Dependency: libssl.so.1.1()(64bit) for package: 1:nginx-1.20.1-9.el7.x86_64
null_resource.shell[0] (remote-exec): --> Running transaction check
null_resource.shell[0] (remote-exec): ---> Package gperftools-libs.x86_64 0:2.6.1-1.el7 will be installed
null_resource.shell[0] (remote-exec): ---> Package nginx-filesystem.noarch 1:1.20.1-9.el7 will be installed
null_resource.shell[0] (remote-exec): ---> Package openssl11-libs.x86_64 1:1.1.1k-2.el7 will be installed
null_resource.shell[1] (remote-exec): --> Finished Dependency Resolution

null_resource.shell[1] (remote-exec): Dependencies Resolved

null_resource.shell[1] (remote-exec): ========================================
null_resource.shell[1] (remote-exec):  Package
null_resource.shell[1] (remote-exec):        Arch   Version        Repository
null_resource.shell[1] (remote-exec):                                    Size
null_resource.shell[1] (remote-exec): ========================================
null_resource.shell[1] (remote-exec): Installing:
null_resource.shell[1] (remote-exec):  nginx x86_64 1:1.20.1-9.el7 epel 587 k
null_resource.shell[1] (remote-exec): Installing for dependencies:
null_resource.shell[1] (remote-exec):  gperftools-libs
null_resource.shell[1] (remote-exec):        x86_64 2.6.1-1.el7    os   272 k
null_resource.shell[1] (remote-exec):  nginx-filesystem
null_resource.shell[1] (remote-exec):        noarch 1:1.20.1-9.el7 epel  24 k
null_resource.shell[1] (remote-exec):  openssl11-libs
null_resource.shell[1] (remote-exec):        x86_64 1:1.1.1k-2.el7 epel 1.5 M

null_resource.shell[1] (remote-exec): Transaction Summary
null_resource.shell[1] (remote-exec): ========================================
null_resource.shell[1] (remote-exec): Install  1 Package (+3 Dependent packages)

null_resource.shell[1] (remote-exec): Total download size: 2.3 M
null_resource.shell[1] (remote-exec): Installed size: 6.6 M
null_resource.shell[1] (remote-exec): Downloading packages:
null_resource.shell[1] (remote-exec): (1/4): nginx-files |  24 kB   00:00
null_resource.shell[1] (remote-exec): (2/4): gperftools- | 272 kB   00:00
null_resource.shell[1] (remote-exec): (3/4): nginx-1.20. | 587 kB   00:00
null_resource.shell[1] (remote-exec): (4/4): openssl11-l | 1.5 MB   00:00
null_resource.shell[1] (remote-exec): ----------------------------------------
null_resource.shell[1] (remote-exec): Total      6.3 MB/s | 2.3 MB  00:00
null_resource.shell[1] (remote-exec): Running transaction check
null_resource.shell[1] (remote-exec): Running transaction test
null_resource.shell[1] (remote-exec): Transaction test succeeded
null_resource.shell[1] (remote-exec): Running transaction
null_resource.shell[1] (remote-exec): Warning: RPMDB altered outside of yum.
null_resource.shell[0] (remote-exec): --> Finished Dependency Resolution

null_resource.shell[0] (remote-exec): Dependencies Resolved

null_resource.shell[0] (remote-exec): ========================================
null_resource.shell[0] (remote-exec):  Package
null_resource.shell[0] (remote-exec):        Arch   Version        Repository
null_resource.shell[0] (remote-exec):                                    Size
null_resource.shell[0] (remote-exec): ========================================
null_resource.shell[0] (remote-exec): Installing:
null_resource.shell[0] (remote-exec):  nginx x86_64 1:1.20.1-9.el7 epel 587 k
null_resource.shell[0] (remote-exec): Installing for dependencies:
null_resource.shell[0] (remote-exec):  gperftools-libs
null_resource.shell[0] (remote-exec):        x86_64 2.6.1-1.el7    os   272 k
null_resource.shell[0] (remote-exec):  nginx-filesystem
null_resource.shell[0] (remote-exec):        noarch 1:1.20.1-9.el7 epel  24 k
null_resource.shell[0] (remote-exec):  openssl11-libs
null_resource.shell[0] (remote-exec):        x86_64 1:1.1.1k-2.el7 epel 1.5 M

null_resource.shell[0] (remote-exec): Transaction Summary
null_resource.shell[0] (remote-exec): ========================================
null_resource.shell[0] (remote-exec): Install  1 Package (+3 Dependent packages)

null_resource.shell[0] (remote-exec): Total download size: 2.3 M
null_resource.shell[0] (remote-exec): Installed size: 6.6 M
null_resource.shell[0] (remote-exec): Downloading packages:
null_resource.shell[0] (remote-exec): (1/4): nginx-files |  24 kB   00:00
null_resource.shell[0] (remote-exec): (2/4): gperftools- | 272 kB   00:00
null_resource.shell[0] (remote-exec): (3/4): nginx-1.20. | 587 kB   00:00
null_resource.shell[0] (remote-exec): (4/4): openssl11-l | 1.5 MB   00:00
null_resource.shell[0] (remote-exec): ----------------------------------------
null_resource.shell[0] (remote-exec): Total      5.3 MB/s | 2.3 MB  00:00
null_resource.shell[0] (remote-exec): Running transaction check
null_resource.shell[0] (remote-exec): Running transaction test
null_resource.shell[0] (remote-exec): Transaction test succeeded
null_resource.shell[0] (remote-exec): Running transaction
null_resource.shell[1]: Still creating... [20s elapsed]
null_resource.shell[0]: Still creating... [20s elapsed]
null_resource.shell[0] (remote-exec):   Installing : 1:openss [         ] 1/4
null_resource.shell[0] (remote-exec):   Installing : 1:openss [#        ] 1/4
null_resource.shell[0] (remote-exec):   Installing : 1:openss [##       ] 1/4
null_resource.shell[0] (remote-exec):   Installing : 1:openss [###      ] 1/4
null_resource.shell[0] (remote-exec):   Installing : 1:openss [####     ] 1/4
null_resource.shell[0] (remote-exec):   Installing : 1:openss [#####    ] 1/4
null_resource.shell[0] (remote-exec):   Installing : 1:openss [######   ] 1/4
null_resource.shell[0] (remote-exec):   Installing : 1:openss [#######  ] 1/4
null_resource.shell[0] (remote-exec):   Installing : 1:openss [######## ] 1/4
null_resource.shell[0] (remote-exec):   Installing : 1:openssl11-libs-1   1/4
null_resource.shell[0] (remote-exec):   Installing : gperftoo [         ] 2/4
null_resource.shell[0] (remote-exec):   Installing : gperftoo [#        ] 2/4
null_resource.shell[0] (remote-exec):   Installing : gperftoo [##       ] 2/4
null_resource.shell[0] (remote-exec):   Installing : gperftoo [###      ] 2/4
null_resource.shell[0] (remote-exec):   Installing : gperftoo [####     ] 2/4
null_resource.shell[0] (remote-exec):   Installing : gperftoo [#####    ] 2/4
null_resource.shell[0] (remote-exec):   Installing : gperftoo [######   ] 2/4
null_resource.shell[0] (remote-exec):   Installing : gperftoo [#######  ] 2/4
null_resource.shell[0] (remote-exec):   Installing : gperftoo [######## ] 2/4
null_resource.shell[0] (remote-exec):   Installing : gperftools-libs-2.   2/4
null_resource.shell[0] (remote-exec):   Installing : 1:nginx- [         ] 3/4
null_resource.shell[0] (remote-exec):   Installing : 1:nginx- [##       ] 3/4
null_resource.shell[0] (remote-exec):   Installing : 1:nginx- [###      ] 3/4
null_resource.shell[0] (remote-exec):   Installing : 1:nginx- [####     ] 3/4
null_resource.shell[0] (remote-exec):   Installing : 1:nginx- [#####    ] 3/4
null_resource.shell[0] (remote-exec):   Installing : 1:nginx- [######   ] 3/4
null_resource.shell[0] (remote-exec):   Installing : 1:nginx- [#######  ] 3/4
null_resource.shell[0] (remote-exec):   Installing : 1:nginx-filesystem   3/4
null_resource.shell[0] (remote-exec):   Installing : 1:nginx- [         ] 4/4
null_resource.shell[0] (remote-exec):   Installing : 1:nginx- [#        ] 4/4
null_resource.shell[0] (remote-exec):   Installing : 1:nginx- [##       ] 4/4
null_resource.shell[0] (remote-exec):   Installing : 1:nginx- [###      ] 4/4
null_resource.shell[0] (remote-exec):   Installing : 1:nginx- [####     ] 4/4
null_resource.shell[0] (remote-exec):   Installing : 1:nginx- [#####    ] 4/4
null_resource.shell[0] (remote-exec):   Installing : 1:nginx- [######   ] 4/4
null_resource.shell[0] (remote-exec):   Installing : 1:nginx- [#######  ] 4/4
null_resource.shell[0] (remote-exec):   Installing : 1:nginx- [######## ] 4/4
null_resource.shell[0] (remote-exec):   Installing : 1:nginx-1.20.1-9.e   4/4
null_resource.shell[0] (remote-exec):   Verifying  : 1:nginx-filesystem   1/4
null_resource.shell[0] (remote-exec):   Verifying  : gperftools-libs-2.   2/4
null_resource.shell[0] (remote-exec):   Verifying  : 1:openssl11-libs-1   3/4
null_resource.shell[0] (remote-exec):   Verifying  : 1:nginx-1.20.1-9.e   4/4

null_resource.shell[0] (remote-exec): Installed:
null_resource.shell[0] (remote-exec):   nginx.x86_64 1:1.20.1-9.el7

null_resource.shell[0] (remote-exec): Dependency Installed:
null_resource.shell[0] (remote-exec):   gperftools-libs.x86_64 0:2.6.1-1.el7
null_resource.shell[0] (remote-exec):   nginx-filesystem.noarch 1:1.20.1-9.el7
null_resource.shell[0] (remote-exec):   openssl11-libs.x86_64 1:1.1.1k-2.el7

null_resource.shell[0] (remote-exec): Complete!
null_resource.shell[0] (remote-exec): ● nginx.service - The nginx HTTP and reverse proxy server
null_resource.shell[0] (remote-exec):    Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
null_resource.shell[0] (remote-exec):    Active: active (running) since Fri 2022-03-25 13:39:46 CST; 7ms ago
null_resource.shell[0] (remote-exec):   Process: 1879 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
null_resource.shell[0] (remote-exec):   Process: 1877 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
null_resource.shell[0] (remote-exec):   Process: 1875 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
null_resource.shell[0] (remote-exec):  Main PID: 1881 (nginx)
null_resource.shell[0] (remote-exec):    CGroup: /system.slice/nginx.service
null_resource.shell[0] (remote-exec):            ├─1881 nginx: master proce...
null_resource.shell[0] (remote-exec):            └─1883 nginx: master proce...

null_resource.shell[0] (remote-exec): Mar 25 13:39:46 xuel-tf-server-0 systemd[1]: ...
null_resource.shell[0] (remote-exec): Mar 25 13:39:46 xuel-tf-server-0 nginx[1877]: ...
null_resource.shell[0] (remote-exec): Mar 25 13:39:46 xuel-tf-server-0 nginx[1877]: ...
null_resource.shell[0] (remote-exec): Mar 25 13:39:46 xuel-tf-server-0 systemd[1]: ...
null_resource.shell[0] (remote-exec): Hint: Some lines were ellipsized, use -l to show in full.
null_resource.shell[0]: Creation complete after 21s [id=1417174727151706057]
null_resource.shell[1] (remote-exec): ** Found 2 pre-existing rpmdb problem(s), 'yum check' output follows:
null_resource.shell[1] (remote-exec): sudo-1.8.23-9.el7.x86_64 has missing requires of config(sudo) = ('0', '1.8.23', '9.el7')
null_resource.shell[1] (remote-exec): sudo-1.8.23-10.el7_9.1.x86_64 is a duplicate with sudo-1.8.23-9.el7.x86_64
null_resource.shell[1] (remote-exec):   Installing : 1:openss [         ] 1/4
null_resource.shell[1] (remote-exec):   Installing : 1:openss [#        ] 1/4
null_resource.shell[1] (remote-exec):   Installing : 1:openss [##       ] 1/4
null_resource.shell[1] (remote-exec):   Installing : 1:openss [###      ] 1/4
null_resource.shell[1] (remote-exec):   Installing : 1:openss [####     ] 1/4
null_resource.shell[1] (remote-exec):   Installing : 1:openss [#####    ] 1/4
null_resource.shell[1] (remote-exec):   Installing : 1:openss [######   ] 1/4
null_resource.shell[1] (remote-exec):   Installing : 1:openss [#######  ] 1/4
null_resource.shell[1] (remote-exec):   Installing : 1:openss [######## ] 1/4
null_resource.shell[1] (remote-exec):   Installing : 1:openssl11-libs-1   1/4
null_resource.shell[1] (remote-exec):   Installing : gperftoo [         ] 2/4
null_resource.shell[1] (remote-exec):   Installing : gperftoo [#        ] 2/4
null_resource.shell[1] (remote-exec):   Installing : gperftoo [##       ] 2/4
null_resource.shell[1] (remote-exec):   Installing : gperftoo [###      ] 2/4
null_resource.shell[1] (remote-exec):   Installing : gperftoo [####     ] 2/4
null_resource.shell[1] (remote-exec):   Installing : gperftoo [#####    ] 2/4
null_resource.shell[1] (remote-exec):   Installing : gperftoo [######   ] 2/4
null_resource.shell[1] (remote-exec):   Installing : gperftoo [#######  ] 2/4
null_resource.shell[1] (remote-exec):   Installing : gperftoo [######## ] 2/4
null_resource.shell[1] (remote-exec):   Installing : gperftools-libs-2.   2/4
null_resource.shell[1] (remote-exec):   Installing : 1:nginx- [         ] 3/4
null_resource.shell[1] (remote-exec):   Installing : 1:nginx- [##       ] 3/4
null_resource.shell[1] (remote-exec):   Installing : 1:nginx- [###      ] 3/4
null_resource.shell[1] (remote-exec):   Installing : 1:nginx- [####     ] 3/4
null_resource.shell[1] (remote-exec):   Installing : 1:nginx- [#####    ] 3/4
null_resource.shell[1] (remote-exec):   Installing : 1:nginx- [######   ] 3/4
null_resource.shell[1] (remote-exec):   Installing : 1:nginx- [#######  ] 3/4
null_resource.shell[1] (remote-exec):   Installing : 1:nginx-filesystem   3/4
null_resource.shell[1] (remote-exec):   Installing : 1:nginx- [         ] 4/4
null_resource.shell[1] (remote-exec):   Installing : 1:nginx- [#        ] 4/4
null_resource.shell[1] (remote-exec):   Installing : 1:nginx- [##       ] 4/4
null_resource.shell[1] (remote-exec):   Installing : 1:nginx- [###      ] 4/4
null_resource.shell[1] (remote-exec):   Installing : 1:nginx- [####     ] 4/4
null_resource.shell[1] (remote-exec):   Installing : 1:nginx- [#####    ] 4/4
null_resource.shell[1] (remote-exec):   Installing : 1:nginx- [######   ] 4/4
null_resource.shell[1] (remote-exec):   Installing : 1:nginx- [#######  ] 4/4
null_resource.shell[1] (remote-exec):   Installing : 1:nginx- [######## ] 4/4
null_resource.shell[1] (remote-exec):   Installing : 1:nginx-1.20.1-9.e   4/4
null_resource.shell[1] (remote-exec):   Verifying  : 1:nginx-filesystem   1/4
null_resource.shell[1] (remote-exec):   Verifying  : gperftools-libs-2.   2/4
null_resource.shell[1] (remote-exec):   Verifying  : 1:openssl11-libs-1   3/4
null_resource.shell[1] (remote-exec):   Verifying  : 1:nginx-1.20.1-9.e   4/4

null_resource.shell[1] (remote-exec): Installed:
null_resource.shell[1] (remote-exec):   nginx.x86_64 1:1.20.1-9.el7

null_resource.shell[1] (remote-exec): Dependency Installed:
null_resource.shell[1] (remote-exec):   gperftools-libs.x86_64 0:2.6.1-1.el7
null_resource.shell[1] (remote-exec):   nginx-filesystem.noarch 1:1.20.1-9.el7
null_resource.shell[1] (remote-exec):   openssl11-libs.x86_64 1:1.1.1k-2.el7

null_resource.shell[1] (remote-exec): Complete!
null_resource.shell[1] (remote-exec): ● nginx.service - The nginx HTTP and reverse proxy server
null_resource.shell[1] (remote-exec):    Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
null_resource.shell[1] (remote-exec):    Active: active (running) since Fri 2022-03-25 13:39:48 CST; 29ms ago
null_resource.shell[1] (remote-exec):   Process: 1877 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
null_resource.shell[1] (remote-exec):   Process: 1875 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
null_resource.shell[1] (remote-exec):   Process: 1873 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
null_resource.shell[1] (remote-exec):  Main PID: 1879 (nginx)
null_resource.shell[1] (remote-exec):    CGroup: /system.slice/nginx.service
null_resource.shell[1] (remote-exec):            ├─1879 nginx: master proce...
null_resource.shell[1] (remote-exec):            └─1881 nginx: master proce...

null_resource.shell[1] (remote-exec): Mar 25 13:39:47 xuel-tf-server-1 systemd[1]: ...
null_resource.shell[1] (remote-exec): Mar 25 13:39:47 xuel-tf-server-1 nginx[1875]: ...
null_resource.shell[1] (remote-exec): Mar 25 13:39:47 xuel-tf-server-1 nginx[1875]: ...
null_resource.shell[1] (remote-exec): Mar 25 13:39:48 xuel-tf-server-1 systemd[1]: ...
null_resource.shell[1] (remote-exec): Hint: Some lines were ellipsized, use -l to show in full.
null_resource.shell[1]: Creation complete after 24s [id=295940961618086294]

Apply complete! Resources: 7 added, 0 changed, 0 destroyed.

Outputs:

summary = {
  "instance" = {
    "0" = {
      "allocate_public_ip" = true
      "availability_zone" = "ap-guangzhou-3"
      "bandwidth_package_id" = tostring(null)
      "cam_role_name" = ""
      "cdh_host_id" = tostring(null)
      "cdh_instance_type" = tostring(null)
      "create_time" = "2022-03-25T05:39:18Z"
      "data_disks" = tolist([
        {
          "data_disk_id" = "disk-41j4k4fc"
          "data_disk_size" = 50
          "data_disk_snapshot_id" = ""
          "data_disk_type" = "CLOUD_PREMIUM"
          "delete_with_instance" = true
          "encrypt" = false
          "throughput_performance" = 0
        },
      ])
      "disable_monitor_service" = false
      "disable_security_service" = false
      "expired_time" = ""
      "force_delete" = false
      "hostname" = "xuel-tf-server-0"
      "id" = "ins-ifzhv2g6"
      "image_id" = "img-oikl1tzv"
      "instance_charge_type" = "POSTPAID_BY_HOUR"
      "instance_charge_type_prepaid_period" = tonumber(null)
      "instance_charge_type_prepaid_renew_flag" = ""
      "instance_count" = tonumber(null)
      "instance_name" = "xuel-tf-server-0"
      "instance_status" = "RUNNING"
      "instance_type" = "S3.SMALL1"
      "internet_charge_type" = "TRAFFIC_POSTPAID_BY_HOUR"
      "internet_max_bandwidth_out" = 2
      "keep_image_login" = tobool(null)
      "key_name" = "skey-ermat2er"
      "password" = tostring(null)
      "placement_group_id" = tostring(null)
      "private_ip" = "10.0.1.17"
      "project_id" = 0
      "public_ip" = "175.178.92.230"
      "running_flag" = true
      "security_groups" = toset([
        "sg-el0589mr",
      ])
      "spot_instance_type" = tostring(null)
      "spot_max_price" = tostring(null)
      "stopped_mode" = tostring(null)
      "subnet_id" = "subnet-079md4re"
      "system_disk_id" = "disk-iwrdhd2c"
      "system_disk_size" = 50
      "system_disk_type" = "CLOUD_PREMIUM"
      "tags" = tomap({
        "tagkey" = "xueltag"
      })
      "user_data" = tostring(null)
      "user_data_raw" = tostring(null)
      "vpc_id" = "vpc-gu4gmx1h"
    }
    "1" = {
      "allocate_public_ip" = true
      "availability_zone" = "ap-guangzhou-3"
      "bandwidth_package_id" = tostring(null)
      "cam_role_name" = ""
      "cdh_host_id" = tostring(null)
      "cdh_instance_type" = tostring(null)
      "create_time" = "2022-03-25T05:39:16Z"
      "data_disks" = tolist([
        {
          "data_disk_id" = "disk-hbccuysq"
          "data_disk_size" = 50
          "data_disk_snapshot_id" = ""
          "data_disk_type" = "CLOUD_PREMIUM"
          "delete_with_instance" = true
          "encrypt" = false
          "throughput_performance" = 0
        },
      ])
      "disable_monitor_service" = false
      "disable_security_service" = false
      "expired_time" = ""
      "force_delete" = false
      "hostname" = "xuel-tf-server-1"
      "id" = "ins-562f4vl0"
      "image_id" = "img-oikl1tzv"
      "instance_charge_type" = "POSTPAID_BY_HOUR"
      "instance_charge_type_prepaid_period" = tonumber(null)
      "instance_charge_type_prepaid_renew_flag" = ""
      "instance_count" = tonumber(null)
      "instance_name" = "xuel-tf-server-1"
      "instance_status" = "RUNNING"
      "instance_type" = "S3.SMALL1"
      "internet_charge_type" = "TRAFFIC_POSTPAID_BY_HOUR"
      "internet_max_bandwidth_out" = 2
      "keep_image_login" = tobool(null)
      "key_name" = "skey-ermat2er"
      "password" = tostring(null)
      "placement_group_id" = tostring(null)
      "private_ip" = "10.0.1.9"
      "project_id" = 0
      "public_ip" = "175.178.227.93"
      "running_flag" = true
      "security_groups" = toset([
        "sg-el0589mr",
      ])
      "spot_instance_type" = tostring(null)
      "spot_max_price" = tostring(null)
      "stopped_mode" = tostring(null)
      "subnet_id" = "subnet-079md4re"
      "system_disk_id" = "disk-apa47ubq"
      "system_disk_size" = 50
      "system_disk_type" = "CLOUD_PREMIUM"
      "tags" = tomap({})
      "user_data" = tostring(null)
      "user_data_raw" = tostring(null)
      "vpc_id" = "vpc-gu4gmx1h"
    }
  }
}

```

生成图片

```shell
terraform graph | dot -Tsvg > graph.svg
```

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/graph.svg)

通过公务IP进行测试：

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220325134025.png)

## 六 其他

目前配置管理通过terraform提供的provisioner来实现，但是其存在一定的弊端，例如脚本内容直接耦合在tf实例中，不能很好的解耦基础设施和配置，provisioner 中的脚本如果执行失败不可控，不具备

## 参考链接

* https://developer.aliyun.com/article/118719
* https://spacelift.io/blog/ansible-vs-terraform
* https://www.gingerdoc.com/tutorials/how-to-use-ansible-with-terraform-for-configuration-management
* https://www.hashicorp.com/resources/ansible-terraform-better-together
* https://alex.dzyoba.com/blog/terraform-ansible/

