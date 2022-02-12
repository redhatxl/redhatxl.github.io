# Velero源码研读一

# 一 引言

在云原生的开源项目里，velero基本上是容器备份项目的标杆，社区的活跃程度也比较高，有些小企业甚至用velero来为一些非关键业务进行备份恢复。熟悉了解velero的代码逻辑，有助于理解Kubernetes世界中的备份与恢复的思路。本文将尝试对velero的代码进行深入的分析，因为篇幅原因，本文将分为多个部分，每个部分独立成篇。本文是这个系列的第一篇，将以velero v1.5.3代码为基础，会简单介绍一下velero的代码结构，模块结构，再对velero的API进行一个简单梳理。



# 二 代码结构

进入velero的代码目录，包含代码的主要有4个目录：

```shell
cmd/
internal/
pkg/
third_party/
```

其中，`cmd/`是`main.go`所在的目录，这里`cmd/`有两个main的实现，一个是velero的主体，另一个是`velero-restic-restore-helper`，这个`velero-restic-restore-helper`我们将在恢复过程中讲。

`internal/`实现了少数几个方法与结构，会被`pkg/`里面用到的逻辑调用到，而且实现也不多，其实完全可以合并入`pkg/`中。`third_party/`就是引用到的`vendor`的代码。因此，主要的代码都在`pkg/`中。

`pkg/`里有这些目录：

```shell
drwxr-xr-x   3 xuel  staff    96B Jun  3 20:40 apis
drwxr-xr-x   7 xuel  staff   224B Jun  3 20:40 archive
drwxr-xr-x  23 xuel  staff   736B Jun  3 20:40 backup
drwxr-xr-x  30 xuel  staff   960B Jun  3 20:40 builder
drwxr-xr-x   4 xuel  staff   128B Jun  3 20:40 buildinfo
drwxr-xr-x  10 xuel  staff   320B Jun  3 20:40 client
drwxr-xr-x   7 xuel  staff   224B Jun  3 20:40 cmd
drwxr-xr-x  31 xuel  staff   992B Jun  3 20:40 controller
drwxr-xr-x   4 xuel  staff   128B Jun  3 20:40 discovery
drwxr-xr-x   4 xuel  staff   128B Jun  3 20:40 features
drwxr-xr-x   5 xuel  staff   160B Jun  3 20:40 generated
drwxr-xr-x  10 xuel  staff   320B Jun  3 20:40 install
drwxr-xr-x   3 xuel  staff    96B Jun  3 20:40 kuberesource
drwxr-xr-x   4 xuel  staff   128B Jun  3 20:40 label
drwxr-xr-x   3 xuel  staff    96B Jun  3 20:40 metrics
drwxr-xr-x   7 xuel  staff   224B Jun  3 20:40 persistence
drwxr-xr-x   8 xuel  staff   256B Jun  3 20:40 plugin
drwxr-xr-x   4 xuel  staff   128B Jun  3 20:40 podexec
drwxr-xr-x  27 xuel  staff   864B Jun  3 20:40 restic
drwxr-xr-x  39 xuel  staff   1.2K Jun  3 20:40 restore
drwxr-xr-x  18 xuel  staff   576B Jun  3 20:40 test
drwxr-xr-x  10 xuel  staff   320B Jun  3 20:40 util
drwxr-xr-x   3 xuel  staff    96B Jun  3 20:40 volume
```

下面我们对比较重要的目录做一些介绍。

- apis

  定义了velero的API，后面章节会把API简单介绍一下

- backup

  备份相关的逻辑，主要被backup controller用到

- cmd

  velero的客户端命令实现，以及服务端实例的实现

- controller

  对应每个API实现的controller，以及其它不对应API的controller

- persistence

  实现了velero对对象存储的操作，通过plugin对接具体的对象存储实现方（如aws）

- plugin

  velero实现了一个plugin的框架，定义了一个plugin的接口，并把诸如对象存储，备份恢复的action，CSI操作等用plugin的方式实现。

- restic

  封装了restic相关的操作

- restore

  恢复相关逻辑，主要被restore controller用到

对于其它的一些目录，我们将会在后面文章系列介绍各个模块或逻辑时候，会把相关用到的目录捎带的提到。

# 三 模块结构

velero的模块结构大致如下所示：

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210902163657.png)



首先，velero暴露给普通用户的是一个客户端命令`velero`，命令封装了各种参数，可以执行安装、配置、备份、恢复等操作。服务端通过`velero install`来安装，将velero服务端部署成Deployment，并且会根据参数把相关的plugin的镜像一起部署在velero的deployment中。

服务端的实现可以类比成一个典型的kubebuild的operator，首先定义不同的CR，也就是API，下面是controller的实现，最下面的restic表示底层的数据拷贝层用的是restic。中间controller层需要用到一些相对比较独立的服务时，都会通过插件系统来对接到内部或者外部的插件服务。`velero.io/plugins`就代表内部的插件实现，其它都是外部的插件实现，由velero或者第三方厂商来实现。



# 三 API

## 3.1 BackupStorageLocation

备份仓库地址（BSL）。这里velero主要把S3对象存储封装进来作为备份仓库的地址，下面是一个实际的例子：

```yaml
    spec:
      accessMode: ReadWrite
      backupSyncPeriod: 3m0s
      config:
        insecureSkipTLSVerify: "true"
        region: default
        s3ForcePathStyle: "true"
        s3Url: http://132.198.139.59:9010
      objectStorage:
        bucket: test
        prefix: host-cluster
      provider: aws
```



## 3.2 VolumeSnapshotLocation

卷快照地址（VSL）。velero定义了一个VolumeSnapshotter的接口，这个接口需要被第三方存储厂商实现，来提供卷快照的功能。但是这里的快照跟CSI的快照比较容易混淆，比如，[Velero备份实战 - 基于Ceph的CSI快照](https://velero.cn/d/7-velero-cephcsi)这篇文章提到的快照实际上是参考2的CSI插件提供的快照能力，跟VSL其实没有关系。

常用的AWS插件（参考3）实现了VolumeSnapshotter，但如果不用标准的AWS S3快照服务，而是使用了CSI接口来做快照，那么其实并没有用到AWS的快照插件功能，只是用了对象存储的插件接口，因此也没有涉及到VSL。

一句话总结，如果用到了第三方存储厂商的快照功能（例如参考4），那就会用到VSL。

## 3.3 Backup

备份、恢复的API是velero的核心API，备份主要包含了几类参数：

- 资源的选择，比如包含的namespace，不包含的资源等
- 控制参数，比如是否做快照，是否用restic做备份等
- BSL，VSL
- hook

## 3.4 Restore

恢复的API首先需要备份CR的名字，然后也包含几类参数：

- 资源的选择，比如恢复哪个namespace，或者某个资源
- 控制参数，控制namespace名字的映射，以及是否恢复PV
- hook

## 3.5 Schedule

- Schedule其实就是一个Cronjob的字符串加上一个备份API的Spec。



## 3.6 PodVolumeBackup

- PodVolumeBackup主要是velero内部使用的一个API，它封装了restic备份一个Pod的一个PV的操作，然后CR的Status会得到restic的层面一个snapshotID。

## 3.7 PodVolumeRestore

- PodVolumeRestore封装了restic恢复一个Pod的PV的操作，上面PodVolumeBackup Status的snapshotID是一个关键参数。

## 3.8 ResticRepository

- 跟上面PodVolumeBackup、PodVolumeRestore类似，ResticRepository也是一个内部使用的API，封装了restic需要的一个仓库的信息。



## 3.9 DeleteBackupRequest

- velero把删除一个备份以及相关资源的操作封装到DeleteBackupRequest里，当一个DeleteBackupRequest CR被创建后，控制逻辑会尝试删除正在进行中的备份，对备份的资源，比如当前集群中生成的PV的Volumesnapshot等，都会一起清理掉。但是对于对象存储中备份的数据包，并不会删除，这个删除由另一个控制器在备份留存周期超时后来完成。





# 参考资料

* https://velero.cn/d/13-velero