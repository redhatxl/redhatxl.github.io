# velero入门

# 一 概述

Velero（以前称为 Heptio Ark）为您提供了用于备份和恢复 Kubernetes 集群资源和持久卷的工具。您可以通过云提供商或本地运行 Velero。 Velero 让您：

* 备份您的集群并在丢失时恢复。
* 将集群资源迁移到其他集群。
* 将您的生产集群复制到开发和测试集群。

# 二 velero组成

* 在集群上运行的服务器
* 本地运行的命令行客户端

# 三 velero工作原理

每个 Velero 操作——按需备份、计划备份、还原——都是一个定制资源，使用 Kubernetes 定制资源定义(CRD)定义，并存储在 etcd 中。Velero 还包括处理自定义资源以执行备份、恢复和所有相关操作的控制器。您可以备份或还原集群中的所有对象，或者可以按类型、命名空间和/或标签筛选对象。Velero 是灾难恢复用例的理想选择，也适用于在集群上执行系统操作(比如升级)之前对应用程序状态进行快照。

## 3.1 按需备份

* 将复制的 Kubernetes 对象的 tarball 上传到云对象存储中。
* 您可以选择指定要在备份期间执行的备份挂钩。例如，您可能需要告诉数据库在拍摄快照之前将其内存缓冲区刷新到磁盘。更多关于备用钩子。

您可以选择指定要在备份期间执行的备份挂钩。例如，您可能需要告诉数据库在拍摄快照之前将其内存缓冲区刷新到磁盘。更多关于备用钩子。

请注意，集群备份并非严格原子性的。如果在备份时正在创建或编辑 Kubernetes 对象，则它们可能不会包含在备份中。捕获不一致信息的几率很低，但有可能。

## 3.2 计划备份

计划操作允许您定期备份数据。第一次备份在第一次创建计划时执行，后续备份按计划的指定时间间隔进行。这些间隔由 Cron 表达式指定。

计划备份以名称 <SCHEDULE NAME>-<TIMESTAMP> 保存，其中 <TIMESTAMP> 的格式为 YYYYMMDDhhmmss。

## 3.3 恢复

恢复操作允许您从先前创建的备份中恢复所有对象和持久卷。您还可以仅恢复对象和持久卷的筛选子集。 Velero 支持多个命名空间重映射——例如，在单个还原中，命名空间“abc”中的对象可以在命名空间“def”下重新创建，命名空间“123”中的对象可以在“456”下重新创建。

还原的默认名称为 <BACKUP NAME>-<TIMESTAMP>，其中 <TIMESTAMP> 的格式为 YYYYMMDDhhmmss。您还可以指定自定义名称。恢复的对象还包括一个带有键 velero.io/restore-name 和值 <RESTORE NAME> 的标签。

默认情况下，备份存储位置以读写模式创建。但是，在还原期间，您可以将备份存储位置配置为只读模式，这将禁用该存储位置的备份创建和删除。这有助于确保在还原方案期间不会无意中创建或删除备份。

您可以选择指定要在还原期间或资源还原后执行的还原挂钩。例如，您可能需要在数据库应用程序容器启动之前执行自定义数据库还原操作。更多关于恢复钩子。

## 3.4 备份工作流

* Velero 客户端调用 Kubernetes API 服务器以创建备份对象。
* BackupController 注意到新的 Backup 对象并执行验证。
* BackupController 开始备份过程。它通过向 API 服务器查询资源来收集要备份的数据。
* BackupController 调用对象存储服务（例如 AWS S3）来上传备份文件。

默认情况下，velero backup create 为任何持久卷制作磁盘快照。您可以通过指定附加标志来调整快照。运行 velero backup create --help 以查看可用标志。可以使用选项 --snapshot-volumes=false 禁用快照。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210902160438.png)

## 3.5 备份的 API 版本

Velero 使用 Kubernetes API 服务器的首选版本为每个组/资源备份资源。恢复资源时，目标集群中必须存在相同的 API 组/版本才能成功恢复。

例如，如果备份的集群在 things API 组中有一个 gizmos 资源，具有 group/versions things/v1alpha1、 things/v1beta1和 things/v1，而服务器的首选组/版本是 things/v1，那么所有 gizmos 都将从 things/v1 API 端点备份。当还原这个集群的备份时，目标集群必须具有 things/v1端点，以便还原 gizmos。请注意，things/v1不需要成为目标集群中的首选版本; 它只需要存在即可。



## 3.6 设置备份到期

在创建备份时，可以通过添加标志 -- TTL < duration > 来指定 TTL (生存时间)。如果 Velero 发现现有的备份资源已经过期，它会删除:

* 备份资源
* 自云对象存储的备份文件
* 所有 PersistentVolume 快照
* 所有相关的恢复

TTL 标志允许用户在表单 -- TTL 24h0m0s 中指定以小时、分钟和秒为单位的值来指定备份保持期。如果未指定，则应用默认 TTL 值为30天。

## 3.7 对象存储同步

把对象的存储视为真理的源泉。它不断地检查是否始终存在正确的备份资源。如果存储桶中有一个格式正确的备份文件，但是在 Kubernetes API 中没有相应的备份资源，Velero 会同步从对象存储到 Kubernetes 的信息。

这允许还原功能在集群迁移场景中工作，在这种场景中，原始备份对象在新集群中不存在。

同样，如果备份对象存在于 Kubernetes 但不在对象存储中，那么它将从 Kubernetes 删除，因为备份 tarball 不再存在。

# 参考链接

* https://velero.io/docs/v1.6/
* https://github.com/vmware-tanzu/velero-plugin-for-aws