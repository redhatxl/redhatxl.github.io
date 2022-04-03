## Terraform使用cloud-init完成端到端基础设施编排

## 一 前言

Terraform总所周知，已成为云上基础设施编排的的行业标准，但其更多的关注点聚焦在支持的provider的各类资源编排，对于编排出来的基础设施，对其内部的配置变更，需要配合其他配置管理工具进行整合使用，以达到端到端的配置。

## 二 相关概念

### 2.1 cloud-init

cloud-init 是大多数 Linux 发行版和所有主要云提供商都提供的标准配置支持工具。 cloud-init 允许您将 shell 脚本传递给您的实例，该实例根据您的规范安装或配置机器。开通服务器，将cloud-init内容放到实例的meta_data 中，通过服务器进行服务内部配置管理。

## 三 场景

利用terraform编排出腾讯云服务器，并通过cloud-init 在服务器内部进行配置。

## 四 流程

* 在服务器的metadata中填写脚本执行内容，通过编排实现。

## 五 实战

### 5.1 编写TF代码

```shell
locals {
  cidr_block_vpc_name    = "10.0.0.0/16"
  cidr_block_subnet_name = "10.0.1.0/24"
  count                  = 2
  metadata_file = "add-ssh-web-app.sh"
}

data "template_file" "user_data" {

  template = file(format("./scripts/%s", local.metadata_file))
}

data "tencentcloud_images" "image" {
  image_type = ["PUBLIC_IMAGE"]
  os_name    = "centos 7.5"
}

data "tencentcloud_instance_types" "instanceType" {
  cpu_core_count = 1
  memory_size    = 1
  filter {
    name   = "instance-family"
    values = ["S3"]
  }
}

data "tencentcloud_availability_zones" "zone" {}

resource "tencentcloud_vpc" "vpc" {
  cidr_block = local.cidr_block_vpc_name
  name       = "xuel_tf_vpc"
}

resource "tencentcloud_subnet" "subnet" {
  availability_zone = data.tencentcloud_availability_zones.zone.zones.0.name
  cidr_block        = local.cidr_block_subnet_name
  name              = "xuel_tf_subnet"
  vpc_id            = tencentcloud_vpc.vpc.id
}

resource "tencentcloud_key_pair" "xuel-tf-key" {
  key_name   = "xueltfkey"
  public_key = "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDCcJxzfjqQDikRRShExu22kvNOfMKeWrn++s5nQMTm+9SNsQISEk+P15JhFVachgjumupMcOIrfJQAAQcnzNmxoRCTCJQfmegGpDZVpE1cyHUhGMA5kwu67PmK1Snm8hkg2Xzlduhr1xysL2mRn3+6o5tsFXhGrYOcSSXnf5SpTPgMjqo339ksH0iv8kvu3NaZRueygLYaVEMjixJvsnUisL3uY8LQ+4cm2Zu5mdQamhWhN0kkSdlfbjPgzxexL4AglD9YDy4I9Q80vKzy33Ubwo17a2aNCF3uPpYvCKiV0H9z2XtMxisKDfsQQA01Q1vpccUIK6L48xSbers8hV2xxpSEWzEuoZg18eG2ikAencA6mhGjFWcp9A1dllY2rUhcEdrjcjXji1SGabjYLsv23ki6EMGjM/AK+fq+vj3pIPUMpscX3xVDGmz/zusq6v1KfOtQw7B/Dg8c2cxKUlEWZqqC3A7rt3JO/RVEbeqSe5mlRm2yngINVemmhkcfZNs= xuel@kaliarchmacbookpro"
}

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
  user_data = base64encode(data.template_file.user_data.rendered)

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
tencentcloud_key_pair.xuel-tf-key: Refreshing state... [id=skey-4141cd1b]
tencentcloud_vpc.vpc: Refreshing state... [id=vpc-i2ir344l]
tencentcloud_subnet.subnet: Refreshing state... [id=subnet-4znec1qo]

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

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
      + key_name                                = "skey-4141cd1b"
      + private_ip                              = (known after apply)
      + project_id                              = 0
      + public_ip                               = (known after apply)
      + running_flag                            = true
      + security_groups                         = (known after apply)
      + subnet_id                               = "subnet-4znec1qo"
      + system_disk_id                          = (known after apply)
      + system_disk_size                        = 50
      + system_disk_type                        = "CLOUD_PREMIUM"
      + tags                                    = {
          + "tagkey" = "xueltag"
        }
      + user_data                               = "IyEvYmluL2Jhc2gKeXVtIC15IGluc3RhbGwgbmdpbngKZWNobyB4dWVsdGZ0ZXN0ID4gL3Vzci9zaGFyZS9uZ2lueC9odG1sL2luZGV4Lmh0bWwKc3lzdGVtY3RsIHN0YXJ0IG5naW54CnN5c3RlbWN0bCBzdGF0dXMgbmdpbng="
      + vpc_id                                  = "vpc-i2ir344l"

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
      + key_name                                = "skey-4141cd1b"
      + private_ip                              = (known after apply)
      + project_id                              = 0
      + public_ip                               = (known after apply)
      + running_flag                            = true
      + security_groups                         = (known after apply)
      + subnet_id                               = "subnet-4znec1qo"
      + system_disk_id                          = (known after apply)
      + system_disk_size                        = 50
      + system_disk_type                        = "CLOUD_PREMIUM"
      + tags                                    = {
          + "tagkey" = "xueltag"
        }
      + user_data                               = "IyEvYmluL2Jhc2gKeXVtIC15IGluc3RhbGwgbmdpbngKZWNobyB4dWVsdGZ0ZXN0ID4gL3Vzci9zaGFyZS9uZ2lueC9odG1sL2luZGV4Lmh0bWwKc3lzdGVtY3RsIHN0YXJ0IG5naW54CnN5c3RlbWN0bCBzdGF0dXMgbmdpbng="
      + vpc_id                                  = "vpc-i2ir344l"

      + data_disks {
          + data_disk_id           = (known after apply)
          + data_disk_size         = 50
          + data_disk_type         = "CLOUD_PREMIUM"
          + delete_with_instance   = true
          + encrypt                = false
          + throughput_performance = 0
        }
    }

Plan: 2 to add, 0 to change, 0 to destroy.

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
              + key_name                                = "skey-4141cd1b"
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
              + subnet_id                               = "subnet-4znec1qo"
              + system_disk_id                          = (known after apply)
              + system_disk_size                        = 50
              + system_disk_type                        = "CLOUD_PREMIUM"
              + tags                                    = {
                  + "tagkey" = "xueltag"
                }
              + user_data                               = "IyEvYmluL2Jhc2gKeXVtIC15IGluc3RhbGwgbmdpbngKZWNobyB4dWVsdGZ0ZXN0ID4gL3Vzci9zaGFyZS9uZ2lueC9odG1sL2luZGV4Lmh0bWwKc3lzdGVtY3RsIHN0YXJ0IG5naW54CnN5c3RlbWN0bCBzdGF0dXMgbmdpbng="
              + user_data_raw                           = null
              + vpc_id                                  = "vpc-i2ir344l"
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
              + key_name                                = "skey-4141cd1b"
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
              + subnet_id                               = "subnet-4znec1qo"
              + system_disk_id                          = (known after apply)
              + system_disk_size                        = 50
              + system_disk_type                        = "CLOUD_PREMIUM"
              + tags                                    = {
                  + "tagkey" = "xueltag"
                }
              + user_data                               = "IyEvYmluL2Jhc2gKeXVtIC15IGluc3RhbGwgbmdpbngKZWNobyB4dWVsdGZ0ZXN0ID4gL3Vzci9zaGFyZS9uZ2lueC9odG1sL2luZGV4Lmh0bWwKc3lzdGVtY3RsIHN0YXJ0IG5naW54CnN5c3RlbWN0bCBzdGF0dXMgbmdpbng="
              + user_data_raw                           = null
              + vpc_id                                  = "vpc-i2ir344l"
            }
        }
    }


Warning: This data source will been deprecated in Terraform TencentCloud provider later version. Please use `tencentcloud_availability_zones_by_product` instead.

  on main.tf line 30, in data "tencentcloud_availability_zones" "zone":
  30: data "tencentcloud_availability_zones" "zone" {}


Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

tencentcloud_instance.xuel_tf_ansible[0]: Creating...
tencentcloud_instance.xuel_tf_ansible[1]: Creating...
tencentcloud_instance.xuel_tf_ansible[0]: Still creating... [10s elapsed]
tencentcloud_instance.xuel_tf_ansible[1]: Still creating... [10s elapsed]
tencentcloud_instance.xuel_tf_ansible[1]: Still creating... [20s elapsed]
tencentcloud_instance.xuel_tf_ansible[0]: Still creating... [20s elapsed]
tencentcloud_instance.xuel_tf_ansible[0]: Creation complete after 30s [id=ins-9jm7nugy]
tencentcloud_instance.xuel_tf_ansible[1]: Creation complete after 30s [id=ins-17yf6ate]

Apply complete! Resources: 2 added, 0 changed, 0 destroyed.

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
      "create_time" = "2022-03-25T07:55:36Z"
      "data_disks" = tolist([
        {
          "data_disk_id" = "disk-jh6puobk"
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
      "id" = "ins-9jm7nugy"
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
      "key_name" = "skey-4141cd1b"
      "password" = tostring(null)
      "placement_group_id" = tostring(null)
      "private_ip" = "10.0.1.16"
      "project_id" = 0
      "public_ip" = "175.178.248.164"
      "running_flag" = true
      "security_groups" = toset([
        "sg-el0589mr",
      ])
      "spot_instance_type" = tostring(null)
      "spot_max_price" = tostring(null)
      "stopped_mode" = tostring(null)
      "subnet_id" = "subnet-4znec1qo"
      "system_disk_id" = "disk-ffynw5ag"
      "system_disk_size" = 50
      "system_disk_type" = "CLOUD_PREMIUM"
      "tags" = tomap({})
      "user_data" = "IyEvYmluL2Jhc2gKeXVtIC15IGluc3RhbGwgbmdpbngKZWNobyB4dWVsdGZ0ZXN0ID4gL3Vzci9zaGFyZS9uZ2lueC9odG1sL2luZGV4Lmh0bWwKc3lzdGVtY3RsIHN0YXJ0IG5naW54CnN5c3RlbWN0bCBzdGF0dXMgbmdpbng="
      "user_data_raw" = tostring(null)
      "vpc_id" = "vpc-i2ir344l"
    }
    "1" = {
      "allocate_public_ip" = true
      "availability_zone" = "ap-guangzhou-3"
      "bandwidth_package_id" = tostring(null)
      "cam_role_name" = ""
      "cdh_host_id" = tostring(null)
      "cdh_instance_type" = tostring(null)
      "create_time" = "2022-03-25T07:55:34Z"
      "data_disks" = tolist([
        {
          "data_disk_id" = "disk-lssq5o74"
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
      "id" = "ins-17yf6ate"
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
      "key_name" = "skey-4141cd1b"
      "password" = tostring(null)
      "placement_group_id" = tostring(null)
      "private_ip" = "10.0.1.5"
      "project_id" = 0
      "public_ip" = "175.178.247.246"
      "running_flag" = true
      "security_groups" = toset([
        "sg-el0589mr",
      ])
      "spot_instance_type" = tostring(null)
      "spot_max_price" = tostring(null)
      "stopped_mode" = tostring(null)
      "subnet_id" = "subnet-4znec1qo"
      "system_disk_id" = "disk-fazgk0hs"
      "system_disk_size" = 50
      "system_disk_type" = "CLOUD_PREMIUM"
      "tags" = tomap({})
      "user_data" = "IyEvYmluL2Jhc2gKeXVtIC15IGluc3RhbGwgbmdpbngKZWNobyB4dWVsdGZ0ZXN0ID4gL3Vzci9zaGFyZS9uZ2lueC9odG1sL2luZGV4Lmh0bWwKc3lzdGVtY3RsIHN0YXJ0IG5naW54CnN5c3RlbWN0bCBzdGF0dXMgbmdpbng="
      "user_data_raw" = tostring(null)
      "vpc_id" = "vpc-i2ir344l"
    }
  }
}
```

通过公务IP进行测试：

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220325134025.png)

## 六 其他

目前配置管理通过cloud-init 通过文件形式读进来操作配置内容，但是具体任务的执行状态还是不可控，需要通过其他方式将运行结果进行反馈。

## 参考链接

* https://learn.hashicorp.com/tutorials/terraform/cloud-init
