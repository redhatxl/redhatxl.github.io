# velero 源码深入分析二

# 一 引言

velero的插件系统理解起来大致可以分为三部分，一是跟参考1的`go-plugin`库的集成，二是插件框架部分，三是插件的客户端管理部分，本文主要会围绕这三部分展开，来深入的分析一下velero的插件系统。

# 二 背景

`go-plugin`这个github项目是著名的开源软件公司Hashicorp的一个项目，最初用来当作Hashicorp其它项目的一个工具。`go-plugin`主要是为go语言写的用来支持RPC或者gRPC的一个插件系统，但是，只能用来支持本地的可靠网络，不支持远程的实际网络。

velero使用`go-plugin`主要是为了与不同的存储厂商更好的集成，尽量降低与外部不同厂商的耦合度。参考3给出了目前实现velero插件的厂商，部分厂商既支持对象存储的插件，也支持快照的插件实现。比较常见的插件实现有：AWS，CSI，Azsure等。

除了上面这些厂商的集成，velero还把备份恢复过程中使用的一些操作（Action），例如对PV、Pod、StorageClass等资源的操作，也抽象出来变成了插件。



# 三 整体视图

velero通过`go-plugin`构造了一个两级的插件系统，如下图所示：

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210902171503.png)

其中第一级定义了主要的插件类型，每一个插件会实现`go-plugin`定义的接口。而每个一级插件，都可以定义具体的gRPC的通讯接口，并且可以包含不同的子插件操作，从而实现二级插件的定义。

例如，`BackupItemAction`定义了一个接口，PV，Pod等操作可以实现这个接口，从而扩展了`BackupItemAction`的二级插件操作。



# 四 插件集成

[plugin.go](https://github.com/hashicorp/go-plugin/blob/master/plugin.go)定义了插件库的RPC和GRPC插件接口:

```go
type Plugin interface {
	Server(*MuxBroker) (interface{}, error)
	Client(*MuxBroker, *rpc.Client) (interface{}, error)
}

type GRPCPlugin interface {
	GRPCServer(*GRPCBroker, *grpc.Server) error
	GRPCClient(context.Context, *GRPCBroker, *grpc.ClientConn) (interface{}, error)
}
```

如果不想支持RPC的插件，可以使用上面`plugin.go`中定义的`NetRPCUnsupportedPlugin`，然后实现`GRPCPlugin`定义的接口来集成。

velero同插件库的集成分为几个部分：

## 4.1 实现了上面插件接口的部分

实现了`GRPCPlugin`接口的主要是这几个文件：

```bash
    pkg/plugin/framework/backup_item_action.go
    pkg/plugin/framework/restore_item_action.go
    pkg/plugin/framework/object_store.go
    pkg/plugin/framework/volume_snapshotter.go
    pkg/plugin/framework/plugin_lister.go
    pkg/plugin/framework/delete_item_action.go
```

其中，`delete_item_action`是velero新引入的一个插件接口，目前并没有服务端的实现方，因此后面的文章中会略过这个插件。所有的插件都实现了`GRPCServer`和`GRPCClient`这两个接口，会在下一章插件框架中详细阐述。

## 4.2 velero对插件操作的接口定义

对上面的每一个插件类型，velero会定义一个属于这个插件类型的独特接口（即上一节说的二级插件），以实现对每一种插件类型的通讯需求。具体的接口定义会在后续部分展开。

## 4.3 实现gRPC的客户/服务端的部分

客户端的实现：

```
    pkg/plugin/framework/backup_item_action_client.go
    pkg/plugin/framework/restore_item_action_client.go
    pkg/plugin/framework/object_store_client.go
    pkg/plugin/framework/volume_snapshotter_client.go
    pkg/plugin/framework/plugin_lister_client.go
```

服务端的实现：

```
    pkg/plugin/framework/backup_item_action_server.go
    pkg/plugin/framework/restore_item_action_server.go
    pkg/plugin/framework/object_store_server.go
    pkg/plugin/framework/volume_snapshotter_server.go
    pkg/plugin/framework/plugin_lister_server.go
```

gRPC的客户端与服务端被封装成结构体，然后会被某个插件类型的`GRPCServer`和`GRPCClient`实现返回，返回的结果最终可以得到第二级的某个具体的插件操作的实现，从而实现两级的插件封装。

## 4.4 Protocol buffer的定义以及自动产生的代码

`pkg/plugin/velero/`下面是velero定义的不同插件类型的接口，`pkg/plugin/proto/`是protocol buffer的定义，`pkg/plugin/generated/`是protocol buffer生成的代码。

## 4.5 使用插件库的服务

1. 主要包括插件的初始化注册，velero对插件服务端的初始化，以及插件客户端的分发，将在下一节展开。

通过与`go-plugin`的集成，velero对不同种类的插件构建了不同的gRPC通道，如下图所示：

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210903142329.png)

- 插件的使用端（左侧彩色）

  左侧velero的备份或者恢复逻辑中，在需要使用插件的地方，首先要拿到不同的插件操作的客户端，然后调用插件操作的接口。例如备份的操作是`BackupItemAction`这个接口，接口的实现会根据插件操作的名字对应到相应的客户端入口，通过接口实现远程的服务调用。

- gRPC的连接部分（中间红色）

  中间的gRPC客户端和服务端就是上面第3点的实现部分。

- 插件的实现端（右侧彩色）

  插件的API请求到了服务端之后，会根据注册的操作把请求定向到实际的插件实现中，从而完成整个插件的请求。



# 五 插件框架

插件框架的主要代码在`pkg/plugin/framework/`。插件框架主要定义了插件的各个类型以及数据结构，并实现了服务端同`go-plugin`的集成。以下是插件框架的数据结构关系图：

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210903142854.png)

插件框架的核心是`Server`，其中主要包括了velero实现的第一级插件的指针。每一个第一级插件会嵌入一个`PluginBase`的结构，这个结构主要作用是指向一个`ServerMux`的结构。`ServerMux`包含的数据结构会定位到第二级插件操作，并实现注册插件操作、`Get`等操作，从而可以得到每一个具体的第二级插件操作的实现。



## 5.1 插件类型的实现

对每一个一级插件，都要实现`go-plugin`插件库的`plugin`接口。例如，以下是`BackupItemActionPlugin`的实现，在`pkg/plugin/framework/backup_item_action.go`：

```go
type BackupItemActionPlugin struct {
	plugin.NetRPCUnsupportedPlugin
	*pluginBase
}

// GRPCClient returns a clientDispenser for BackupItemAction gRPC clients.
func (p *BackupItemActionPlugin) GRPCClient(_ context.Context, _ *plugin.GRPCBroker, clientConn *grpc.ClientConn) (interface{}, error) {
	return newClientDispenser(p.clientLogger, clientConn, newBackupItemActionGRPCClient), nil
}

// GRPCServer registers a BackupItemAction gRPC server.
func (p *BackupItemActionPlugin) GRPCServer(_ *plugin.GRPCBroker, server *grpc.Server) error {
	proto.RegisterBackupItemActionServer(server, &BackupItemActionGRPCServer{mux: p.serverMux})
	return nil
}
```

可以看到，`BackupItemActionPlugin`的`GRPCClient`主要是返回一个`BackupItemAction`的gRPC客户端的分发器，并且参数会带一个gRPC的通道；而`GRPCServer`主要是为了注册一个`BackupItemAction`的gRPC服务端的实现，在服务端初始化时会被`go-plugin`调用到。



## 5.2 插件操作接口定义

velero对每个插件操作的接口定义在`pkg/plugin/velero/`下面，例如，拿备份的操作举例，`pkg/plugin/velero/backup_item_action.go`定义的接口是这样：

```go
type BackupItemAction interface {
	AppliesTo() (ResourceSelector, error)
	Execute(item runtime.Unstructured, backup *api.Backup) (runtime.Unstructured, []ResourceIdentifier, error)
}

// ResourceIdentifier describes a single item by its group, resource, namespace, and name.
type ResourceIdentifier struct {
	schema.GroupResource
	Namespace string
	Name      string
}
```

恢复操作的接口看起来一样，但其实参数类型和返回值不一样，这里就不列出。

`AppliesTo`接口的实现主要是为了对执行操作的资源做进一步的筛选，而`Execute`主要是为了执行具体的动作，然后返回这个资源。例如，对卷快照的备份来说，`Execute`的实现可能会去创建一个PVC的`VolumeSnapshot`。

备份、恢复等操作的实现大部分是在velero内部实现，像对象仓库和卷快照主要是外部的插件来实现。

对象仓库的接口，在`pkg/plugin/velero/object_store.go`:

```go
type ObjectStore interface {
	Init(config map[string]string) error
	PutObject(bucket, key string, body io.Reader) error
	ObjectExists(bucket, key string) (bool, error)
	GetObject(bucket, key string) (io.ReadCloser, error)
	ListCommonPrefixes(bucket, prefix, delimiter string) ([]string, error)
	ListObjects(bucket, prefix string) ([]string, error)
	DeleteObject(bucket, key string) error
	CreateSignedURL(bucket, key string, ttl time.Duration) (string, error)
}
```

卷快照接口定义在`pkg/plugin/velero/volume_snapshotter.go`：

```go
type VolumeSnapshotter interface {
	Init(config map[string]string) error
	CreateVolumeFromSnapshot(snapshotID, volumeType, volumeAZ string, iops *int64) (volumeID string, err error)
	GetVolumeID(pv runtime.Unstructured) (string, error)
	SetVolumeID(pv runtime.Unstructured, volumeID string) (runtime.Unstructured, error)
	GetVolumeInfo(volumeID, volumeAZ string) (string, *int64, error)
	CreateSnapshot(volumeID, volumeAZ string, tags map[string]string) (snapshotID string, err error)
	DeleteSnapshot(snapshotID string) error
}
```

`PluginLister`是一个比较特殊的插件类型，它的作用就是为了能得到当前velero服务端支持的所有插件的类型。`PluginLister`的实现也相对比较直接，服务端的实现就是把在插件框架的`Server`中注册的所有的二级插件都列出来。

## 5.3 内部插件

`pkg/cmd/server/plugin/plugin.go`可以看到velero注册的内部插件类型：

```go
				RegisterBackupItemAction("velero.io/pv", newPVBackupItemAction).
				RegisterBackupItemAction("velero.io/pod", newPodBackupItemAction).
        ...
        ...
```

这里`newPVBackupItemAction`是一个函数，会返回一个结构：

```go
func newPVBackupItemAction(logger logrus.FieldLogger) (interface{}, error) {
	return backup.NewPVCAction(logger), nil
}
```

这个结构就是二级插件操作的具体实现。例如，这里的`NewPVCAction`的实现在`pkg/backup/backup_pv_action.go`，这里会返回一个`PVCAction`的结构，并且实现了`AppliesTo`和`Execute`这两个velero定义的接口。

其它的内部插件实现机制类似，本文就不再展开。具体的实现可以去`pkg/backup/`和`pkg/restore/`下面参考带`_action.go`后缀的文件。

## 5.4 外部插件

这里稍微分析一下比较常见的外部插件：

### 5.4.1 CSI插件

首先看`main.go`，注册的插件类型是备份和恢复的操作，而没有注册`VolumeSnapshotter`类型的插件：

```go
func main() {
	veleroplugin.NewServer().
		BindFlags(pflag.CommandLine).
		RegisterBackupItemAction("velero.io/csi-pvc-backupper", newPVCBackupItemAction).
		RegisterBackupItemAction("velero.io/csi-volumesnapshot-backupper", newVolumeSnapshotBackupItemAction).
		RegisterBackupItemAction("velero.io/csi-volumesnapshotclass-backupper", newVolumesnapshotClassBackupItemAction).
		RegisterBackupItemAction("velero.io/csi-volumesnapshotcontent-backupper", newVolumeSnapContentBackupItemAction).
		RegisterRestoreItemAction("velero.io/csi-pvc-restorer", newPVCRestoreItemAction).
		RegisterRestoreItemAction("velero.io/csi-volumesnapshot-restorer", newVolumeSnapshotRestoreItemAction).
		RegisterRestoreItemAction("velero.io/csi-volumesnapshotclass-restorer", newVolumeSnapshotClassRestoreItemAction).
		RegisterRestoreItemAction("velero.io/csi-volumesnapshotcontent-restorer", newVolumeSnapshotContentRestoreItemAction).
		Serve()
}
```

进一步去看`internal/backup/`或者`internal/restore/`的实现，可以看到主要的逻辑也比较直接。CSI插件主要会针对PVC，VolumeSnapshot，VolumeSnapshotClass和VolumeSnapshotContent这几个CSI支持的资源实现`AppliesTo`和`Execute`这两个接口。具体实现了逻辑这里就不展开 ，感兴趣的读者可以自己去看CSI插件的代码，见参考4。

### 5.4.2 AWS插件

AWS插件之所以比较常用是因为大部分的对象存储都实现了AWS的S3接口，因此大部分的公有云对象存储都可以通过AWS插件来支持。看`main.go`：

```go
func main() {
	veleroplugin.NewServer().
		BindFlags(pflag.CommandLine).
		RegisterObjectStore("velero.io/aws", newAwsObjectStore).
		RegisterVolumeSnapshotter("velero.io/aws", newAwsVolumeSnapshotter).
		Serve()
}
```

可以看到AWS插件实现的是`ObjectStore`和`VolumeSnapshotter`的接口。而实际上由于AWS的S3接口的广泛性，我们大部分用到的都是`ObjectStore`的插件实现，用来对接不同的对象存储。具体的逻辑也不展开，读者可以去参考5进一步查阅AWS插件的代码。



# 六 服务端初始化

插件框架对服务端的初始化可以参考上面AWS插件的`main.go`，可以看到一个新的`Server`创建出来，注册了一些插件之后，就是调用`Serve()`这个函数（`pkg/plugin/framework/server.go`）：

```go
	var pluginIdentifiers []PluginIdentifier
	...
  ...

	pluginLister := NewPluginLister(pluginIdentifiers...)

	plugin.Serve(&plugin.ServeConfig{
		HandshakeConfig: Handshake(),
		Plugins: map[string]plugin.Plugin{
			string(PluginKindBackupItemAction):  s.backupItemAction,
			string(PluginKindVolumeSnapshotter): s.volumeSnapshotter,
			string(PluginKindObjectStore):       s.objectStore,
			string(PluginKindPluginLister):      NewPluginListerPlugin(pluginLister),
			string(PluginKindRestoreItemAction): s.restoreItemAction,
			string(PluginKindDeleteItemAction):  s.deleteItemAction,
		},
		GRPCServer: plugin.DefaultGRPCServer,
	})
```

`Serve()`这个函数大致做了几件事情：

1. 对所有一级插件类型生成二级插件ID并全部保存到`pluginIdentifiers`数组中
2. 初始化`go-plugin`的`ServeConfig`并用`Serve()`来服务

第二步中，`go-plugin`会把所有的插件都调用一遍每个插件实现的`GRPCServer()`接口，从而把每个一级插件的服务端实现注册到`go-plugin`中，然后进行典型的服务端的监听与服务过程。

## 插件客户端管理

插件客户端管理的代码在`pkg/plugin/clientmgmt/`。客户端管理部分主要定义了管理客户端相关的几个数据结构和方法，把二级插件的使用跟备份恢复等主要逻辑对接起来。以下是插件客户端管理的数据结构关系图：

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210903143924.png)

由上面的关系图可以看出，`Manager`是核心的结构，它包含了`Registry`和一个`RestartableProcess`的集合。`Register`用来管理插件的注册和分发，而`RestartableProcess`则把二级插件封装成一个类似进程的结构，可以方便的进行操控，并且得到该插件的信息。而`Manager`本身提供了一系列`Get`方法来得到插件的客户端，从而方便的使用插件的服务。

### 插件注册

`Registry`（`pkg/plugin/clientmgmt/registry.go`）实现了插件的注册，主要的方法是`DiscoverPlugins`：

```go
func (r *registry) DiscoverPlugins() error {
	plugins, err := r.readPluginsDir(r.dir)
	if err != nil {
		return err
	}

	// Start by adding velero's internal plugins
	commands := []string{os.Args[0]}
	// Then add the discovered plugin executables
	commands = append(commands, plugins...)

	return r.discoverPlugins(commands)
}
```

这里`commands`会包括`velero`自己这个二进制文件，另外会包括velero的pod里面`/plugins`下面的`velero-plugin-for-csi`，`velero-plugin-for-aws`等外部的插件程序。

`discoverPlugins`会对每个`command`封装一个`Process`，然后通过`Process`的插件分发得到`PluginLister`这个插件，并且执行`PluginLister`的`ListPlugins()`接口，就得到全部的插件名字以及类型。最终所有的二级插件信息会被记录在`Registry`的`pluginsByID`和`pluginsByKind`这两个map中，使用的时候只要通过插件的名字来访问。

### 插件分发

当使用插件的时候，会用到插件客户端的分发功能来得到具体的一个插件客户端的实例，从而使用插件的服务。插件客户端的分发大致可以分为两步：

1. 通过`Manager`的`Get`方法得到一个插件的代理实例

   `pkg/plugin/clientmgmt/`下面为每一个插件类型实现了一个代理结构。例如`restartable_backup_item_action.go`实现了`restartableBackupItemAction`，而这个代理结构实现了相同的`AppliesTo()`、`Execute()`接口。

2. 通过这个代理实例来得到实际的客户端实例，然后用客户端实例的`AppliesTo`和`Execute`接口来使用插件

   例如，`restartableBackupItemAction`会使用`RestartableProcess`提供的方法来得到具体的插件客户端实例，从而使用客户端的`AppliesTo()`、`Execute()`实现。

上面的1.比较直接，但是2.的实现过程相对来说有一点复杂，涉及到多个模块，包括`go-plugin`插件库这边的配合。以下是一个以`BackupItemAction`为例的简化的时序图：

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210903143957.png)

可以看到，插件分发的过程会通过`go-plugin`插件库去调用每个一级插件实现的`GRPCClient()`接口，这个接口会返回一个`clientDispenser`的结构，其中记录了这个插件的`GRPCClient`的初始化方法，例如`newBackupItemActionGRPCClient`。调用这个初始化方法会得到一个`BackupItemActionGRPCClient`的实例，这个实例实现了插件客户端的`AppliesTo`接口，插件的用户最终会用到这个接口来获得服务端的插件操作服务。

## 总结

通过上面的分析，我们可以看出，velero实现的这一套插件系统的确可以把很多跟核心备份、恢复流程不相关的操作解耦，放到各个插件中实现，并且不同的厂商可以比较方便的实现自己的插件支持。当更多的厂商进来支持velero的插件之后，velero就可以做到没有厂商锁定，而且可以在更多的平台提供备份、恢复以及迁移的功能。

总的来说，velero实现了插件与核心逻辑解耦的目的，让插件的实现方比较简单，但是velero跟`go-plugin`的集成还是略复杂。