# Velero plugin system

# 一 velero插件



# 二 插件种类

Velero 目前支持以下几种插件: 

* 对象存储-持久化并检索备份、备份日志文件、还原警告/错误文件、还原日志。
* 卷 Snapshotter ——从卷(备份期间)和卷(恢复期间)创建快照。
* 备份项目操作-在将单个项目存储到备份文件之前，对它们执行任意逻辑。
* 恢复项目操作——在库伯内特集群中恢复单个项目之前，对它们执行任意逻辑。

Velero 可以在一个单一的、可恢复的过程中托管多个插件。插件可以是任何支持的类型。去找main.go，去吧。

# 三 源码研读



# 四 构建插件

* 构建插件

```shell
$ make
```

* 构建一个镜像

```shell
$ make container
```

* 构建出来默认的名称为： `velero/velero-plugin-example:master`，如果想自定义运行下面命令

```shell
$ IMAGE=your-repo/your-name VERSION=your-version-tag make container 
```



# 五 部署插件

要将插件映像部署到 Velero 服务器

* 将镜像拉取到部署集群node节点上。
* 运行`velero plugin add <registry/image:version>`

# 六 使用插件

当插件被部署时，它只能被使用。为了使插件有效，你必须修改你的配置:

## 6.1 备份存储

1. 运行`kubectl edit backupstoragelocation <location-name> -n <velero-namespace>`,例如运行：`kubectl edit backupstoragelocation default -n velero`或者`velero backup-location create <location-name> --provider <provider-name>`

2. 更改 spec.provider 的值以启用对象存储插件。
3. 保存退出，插件将在下次备份恢复中使用。

## 6.2 卷快照存储

1. 运行`kubectl edit volumesnapshotlocation <location-name> -n <velero-namespace>`,例如运行：`kubectl edit volumesnapshotlocation default -n velero`或者`velero snapshot-location create <location-name> --provider <provider-name>`

2. 更改 spec.provider 的值以启用卷快照插件。
3. 保存退出，插件将在下次备份恢复中使用。







# 参考链接

* https://velero.io/docs/main/overview-plugins/
* https://github.com/vmware-tanzu/velero-plugin-example/tree/main
* https://github.com/vmware-tanzu/velero-plugin-for-csi
* https://github.com/AliyunContainerService/velero-plugin