# Velero 使用

# 一 灾难恢复

使用计划和只读备份存储位置如果您定期备份集群的资源，那么在发生意外事故(如服务中断)时，您就能够返回到以前的状态。与 Velero 一起这样做看起来如下

1. 在集群上首次运行 Velero 服务器后，设置每日备份（根据需要替换命令中的 <SCHEDULE NAME>）：

```fallback
velero schedule create <SCHEDULE NAME> --schedule "0 7 * * *"
```

这将创建一个名为 < schedule name >-< timestamp > 的 Backup 对象。默认的备份保持期(表示为 TTL (生存时间))为30天(720小时) ; 您可以使用 -- TTL < duration > 标志根据需要更改这个值。有关备份过期的更多信息，请参见 velero 如何工作。

2. 灾难发生了，您需要重新创建您的资源。
3. 将您的备份存储位置更新为只读模式（这可以防止在还原过程中在备份存储位置创建或删除备份对象）

```bash
kubectl patch backupstoragelocation <STORAGE LOCATION NAME> \
    --namespace velero \
    --type merge \
    --patch '{"spec":{"accessMode":"ReadOnly"}}'
```

4. 使用最新的 Velero 备份创建还原：

```bash
velero restore create --from-backup <SCHEDULE NAME>-<TIMESTAMP>
```

5. 准备好后，将备份存储位置恢复为读写模式：

```bash
kubectl patch backupstoragelocation <STORAGE LOCATION NAME> \
   --namespace velero \
   --type merge \
   --patch '{"spec":{"accessMode":"ReadWrite"}}'
```

# 二 集群迁移

## 2.1 执行迁移

使用 Backups 和 Restores Velero 可以帮助您将资源从一个集群移植到另一个集群，只要您将每个 Velero 实例指向相同的云对象存储位置。此场景假设您的集群由相同的云提供商托管。注意，Velero 并不支持跨云提供者迁移持久卷快照。如果您希望在云平台之间迁移卷数据，请启用 restic，它将在文件系统级备份卷内容。

1. （集群 1）假设您还没有使用 Velero 调度操作检查点数据，您需要先备份整个集群（根据需要替换 <BACKUP-NAME>）：

```fallback
velero backup create <BACKUP-NAME>
```

默认备份保留期，表示为 TTL（生存时间），为 30 天（720 小时）；您可以根据需要使用 --ttl <DURATION> 标志进行更改。有关备份到期的更多信息，请参阅 velero 的工作原理。

2. (Cluster 2)配置 BackupStorageLocations 和 VolumeSnapshotLocations，指向 Cluster 1使用的位置，使用 velero 备份位置 create 和 velero 快照位置 create。使用 -- access-mode = ReadOnly 标志创建 velero 备份位置 create，确保将 BackupStorageLocations 配置为只读。
3. （集群 2）确保已创建 Velero 备份对象。 Velero 资源与云存储中的备份文件同步。

```fallback
velero backup describe <BACKUP-NAME>
```

注意：默认同步间隔为 1 分钟，因此请务必等待再检查。您可以使用 --backup-sync-period 标志将此间隔配置到 Velero 服务器。

4. （集群 2）一旦您确认正确的备份 (<BACKUP-NAME>) 现在存在，您可以使用以下命令恢复所有内容：

```shell
velero restore create --from-backup <BACKUP-NAME>
```



## 2.2 验证两个集群

检查第二个集群是否按预期运行：

1. *(Cluster 2)* Run:

   ```fallback
   velero restore get
   ```

2. Then run:

   ```fallback
   velero restore describe <RESTORE-NAME-FROM-GET-COMMAND>
   ```

如果遇到问题，请确保 Velero 在两个集群的相同命名空间中运行。

# 三 资源过滤

按命名空间、类型或标签筛选对象。

当不使用过滤选项时，Velero 包括备份或恢复中的所有对象。

## 3.1 包含

* –include-namespaces
  * 备份命名空间及其对象。

- ```bash
  velero backup create <backup-name> --include-namespaces <namespace>
  ```

  * 还原两个名称空间及其对象。

  ```shell
  velero restore create <backup-name> --include-namespaces <namespace1>,<namespace2>
  ```

- –include-resources

  - 备份集群中的所有部署。

  ```shell
  velero backup create <backup-name> --include-resources deployments
  ```

  * 恢复集群中的所有部署和配置映射。

  ```shell
  velero restore create <backup-name> --include-resources deployments,configmaps
  ```

  * 备份命名空间中的部署。

  ```shell
  velero backup create <backup-name> --include-resources deployments --include-namespaces <namespace>
  ```

- –include-cluster-resources

此选项可以具有三个可能的值：

* * true: 包含所有集群范围的资源。
  * false：不包括集群范围的资源。
  * nil ：（“自动”或未提供）
    * 备份或恢复所有命名空间时包括集群范围的资源。默认值：真。
    * 使用命名空间过滤时，不包括集群范围的资源。默认值：假。
      * 如果由自定义操作（例如 PVC->PV）触发，某些相关的集群范围资源可能仍会被备份/恢复，除非 --include-cluster-resources=false。

1. 备份整个集群，包括集群范围内的资源。

```go
	velero backup create <backup-name>
```

2. 仅恢复集群中的命名空间资源。		

```shell
velero restore create <backup-name> --include-cluster-resources=false
```

3. 备份命名空间并包含集群范围的资源。

```shell
velero backup create <backup-name> --include-namespaces <namespace> --include-cluster-resources=true 
```

* –selector
  * 包含与标签选择器匹配的资源。

- ```bash
  velero backup create <backup-name> --selector <key>=<value>
  ```

## 3.2 排除

从备份中排除特定资源。

通配符排除被忽略。

* –exclude-namespaces
  *  排除命名空间 从集群备份中排除 kube-system。

    ```shell
    velero backup create <backup-name> --exclude-namespaces kube-system
    ```

  * 在还原期间排除两个命名空间。

    ```shell
    velero restore create <backup-name> --exclude-namespaces <namespace1>,<namespace2>
    ```

* – exclude-resources

  * 从备份中排除秘密。

    ```shell
    velero backup create <backup-name> --exclude-resources secrets
    ```

  * 排除机密和角色绑定。

    ```shell
    velero backup create <backup-name> --exclude-resources secrets,rolebindings
    ```

* velero.io/exclude-from-backup=true

  带有 velero.io/exclude-from-backup=true 标签的资源不包含在备份中，即使它包含匹配的选择器标签。

# 四 Backup Reference

## 4.1 从备份中排除特定项目

即使它们与备份规范中定义的资源/名称空间/标签选择器匹配，也可以排除备份中的单个项。要做到这一点，标签的项目如下:

```bash
kubectl label -n <ITEM_NAMESPACE> <RESOURCE>/<NAME> velero.io/exclude-from-backup=true
```

## 4.2 指定特定种类资源的备份顺序

要按照特定顺序备份特定类型的资源，可以使用 option-ordered-resources 指定将 sort 映射到该类型特定资源的有序列表。资源名称用逗号分隔，名称格式为“ namespace/resourcename”。对于群集范围资源，只需使用资源名称。映射中的键值对用分号分隔。善良的名字是复数形式。

```bash
velero backup create backupName --include-cluster-resources=true --ordered-resources 'pods=ns1/pod1,ns1/pod2;persistentvolumes=pv4,pv8' --include-namespaces=ns1

velero backup create backupName --ordered-resources 'statefulsets=ns1/sts1,ns1/sts0' --include-namespaces=ns1
```

# 五 Backup Hooks

## 5.1 Backup Hooks

在执行备份时，可以指定一个或多个命令，以便在备份 pod 时在 pod 中的容器中执行。这些命令可以配置为在任何自定义操作处理(“ pre”hooks)之前运行，或者在所有自定义操作完成并备份了自定义操作指定的任何附加项(“ post”hooks)之后运行。注意，钩子不会在容器的 shell 中执行。

有两种方法可以指定钩子：pod 本身的注释和备份规范中的注释。

## 5.2 将 Hooks 指定为 Pod 注释

您可以在 pod 上使用以下注解让 Velero 在备份 pod 时执行一个钩子：

### 5.2.1 Pre hooks

* pre.hook.backup.velero.io/container

应执行命令的容器。默认为 pod 中的第一个容器。可选的。

* pre.hook.backup.velero.io/command

要执行的命令。如果需要多个参数，将命令指定为 JSON 数组，例如 ["/usr/bin/uname", "-a"]

* pre.hook.backup.velero.io/on-error

如果命令返回非零退出代码，该怎么办。默认为失败。有效值为失败并继续。可选的。

* pre.hook.backup.velero.io/timeout

等待命令执行的时间。如果命令超过超时，则认为挂钩错误。默认为 30 秒。可选的。



### 5.2.2 Post hooks

* post.hook.backup.velero.io/container

应执行命令的容器。默认为 pod 中的第一个容器。可选的。

* post.hook.backup.velero.io/command

要执行的命令。如果需要多个参数，将命令指定为 JSON 数组，例如 ["/usr/bin/uname", "-a"]

* post.hook.backup.velero.io/on-error

如果命令返回非零退出代码，该怎么办。默认为失败。有效值为失败并继续。可选的。

* post.hook.backup.velero.io/超时

等待命令执行的时间。如果命令超过超时，则认为挂钩错误。默认为 30 秒。可选的。

## 5.3 在备份规范中指定钩子

请参阅有关备份 API 类型的文档，了解如何在备份规范中指定挂钩。[Backup API Type](https://velero.io/docs/v1.7/api-types/backup/)

## 5.4 使用 fsfreeze 的钩子示例

此示例将引导您使用 pre 和 post 挂钩来冻结文件系统。冻结文件系统有助于确保在拍摄快照之前所有挂起的磁盘 I/O 操作都已完成。

本示例使用 [examples/nginx-app/with-pv.yaml](https://github.com/vmware-tanzu/velero/blob/v1.7.0/examples/nginx-app/with-pv.yaml)。按照您的提供商的[steps for your provider](https://velero.io/docs/v1.7/backup-hooks/cloud-common.md)此示例。

## 5.5 Annotations

Velero [example/nginx-app/with-pv.yaml](https://github.com/vmware-tanzu/velero/blob/v1.7.0/examples/nginx-app/with-pv.yaml)作为将 pre 和 post 钩子注释直接添加到声明式部署的示例。下面是就地更新对象的示例。

```shell
kubectl annotate pod -n nginx-example -l app=nginx \
    pre.hook.backup.velero.io/command='["/sbin/fsfreeze", "--freeze", "/var/log/nginx"]' \
    pre.hook.backup.velero.io/container=fsfreeze \
    post.hook.backup.velero.io/command='["/sbin/fsfreeze", "--unfreeze", "/var/log/nginx"]' \
    post.hook.backup.velero.io/container=fsfreeze
```

现在通过创建备份来测试 pre 和 post 挂钩。您可以使用 Velero 日志来验证 pre 和 post 挂钩是否正在运行并无错误退出。

```shell
velero backup create nginx-hook-test

velero backup get nginx-hook-test
velero backup logs nginx-hook-test | grep hookCommand
```

## 5.6 使用多个命令

要使用多个命令，请将目标命令包装在一个 shell 中，并用 ;、&& 或其他 shell 条件结构将它们分开。

```shell
    pre.hook.backup.velero.io/command='["/bin/bash", "-c", "echo hello > hello.txt && echo goodbye > goodbye.txt"]'
```

# 六 Restore Reference

## 6.1 恢复到不同的命名空间

Velero 可以将资源还原到与备份它们的命名空间不同的命名空间中。为此，请使用 --namespace-mappings 标志：

```bash
velero restore create RESTORE_NAME \
  --from-backup BACKUP_NAME \
  --namespace-mappings old-ns-1:new-ns-1,old-ns-2:new-ns-2
```

# 6.2 当用户删除还原对象时会发生什么

还原对象表示还原操作。还原对象有两种类型的删除：

1. 使用 velero restore delete 删除。此命令将删除代表它的自定义资源及其单独的日志和结果文件。但是，它不会从集群中删除它创建的任何对象。
2. 使用 kubectl -n velero delete restore 删除。此命令将删除代表还原的自定义资源，但不会删除对象存储中的日志/结果文件，或在集群中还原期间创建的任何对象。



# 参考链接

* https://velero.io/docs/v1.5/disaster-case/