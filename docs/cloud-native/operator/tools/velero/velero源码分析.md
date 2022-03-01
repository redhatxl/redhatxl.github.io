## Velero 控制器流程



## 一 控制器通用

在 velero 所有的业务逻辑控制器中都包含通用控制器

```go
// velero/pkg/controller/generic_controller.go


// 通用控制器
type genericController struct {
	name             string														//通用控制器名称
	queue            workqueue.RateLimitingInterface	// 限速队列
	logger           logrus.FieldLogger								// 日志
	syncHandler      func(key string) error						// 同步函数
	resyncFunc       func()														// 重新同步方法
	resyncPeriod     time.Duration										// 重新同步周期
	cacheSyncWaiters []cache.InformerSynced						// cacheSync
}

// 
```

### 1.1 Run方法

在控制器中定义有控制器实现的接口

```go
// velero/pkg/controller/interface.go 

// 代表一个可运行的组建
type Interface interface {
	// Run runs the component.
	Run(ctx context.Context, workers int) error
}

```

通用接口具体 Run 方法实现

```go
// velero/pkg/controller/generic_controller.go

func (c *genericController) Run(ctx context.Context, numWorkers int) error {
  // 必须提供syncHandler和resyncFunc
	if c.syncHandler == nil && c.resyncFunc == nil {
		// programmer error
		panic("at least one of syncHandler or resyncFunc is required")
	}

	var wg sync.WaitGroup
	// 最后处理
	defer func() {
		c.logger.Info("Waiting for workers to finish their work")
		// 关闭队列
		c.queue.ShutDown()

		// We have to wait here in the deferred function instead of at the bottom of the function body
		// because we have to shut down the queue in order for the workers to shut down gracefully, and
		// we want to shut down the queue via defer and not at the end of the body.
    // 等待所有的 worker waitgroup完成
		wg.Wait()

		c.logger.Info("All workers have finished")

	}()

	c.logger.Info("Starting controller")
	defer c.logger.Info("Shutting down controller")

	// only want to log about cache sync waiters if there are any
  // 如果存在缓存syncwaiter，等待informer sync完成
	if len(c.cacheSyncWaiters) > 0 {
		c.logger.Info("Waiting for caches to sync")
    // 等待sync 完成
		if !cache.WaitForCacheSync(ctx.Done(), c.cacheSyncWaiters...) {
			return errors.New("timed out waiting for caches to sync")
		}
		c.logger.Info("Caches are synced")
	}
	// 开始执行syncHandler
	if c.syncHandler != nil {
		wg.Add(numWorkers)
		for i := 0; i < numWorkers; i++ {
			go func() {
        // 运行runWorker
				wait.Until(c.runWorker, time.Second, ctx.Done())
				wg.Done()
			}()
		}
	}

  // 写成运行resync
	if c.resyncFunc != nil {
		if c.resyncPeriod == 0 {
			// Programmer error
			panic("non-zero resyncPeriod is required")
		}

		wg.Add(1)
		go func() {
      
			wait.Until(c.resyncFunc, c.resyncPeriod, ctx.Done())
			wg.Done()
		}()
	}

	<-ctx.Done()

	return nil
}
```

### 1.2 runWorker

runWorker 方法就是不断运行 processNextWorkItem。

```go
func (c *genericController) runWorker() {
	// continually take items off the queue (waits if it's
	// empty) until we get a shutdown signal from the queue
	for c.processNextWorkItem() {
	}
}
```

### 1.3 processNextWorkItem

processNextWorkItem 方法，中主要是从限速队列获取元素，并调用用户实现的 syncHandler 业务逻辑方法对元素进行处理，如果处理错误则再次延迟添加至队列，等待下一次处理。

```go
func (c *genericController) processNextWorkItem() bool {
  // 从workqueue获取元素
	key, quit := c.queue.Get()
  // 如果shutdown，则直接返回false
	if quit {
		return false
	}

  // 最后标记改元素以及处理完成
	defer c.queue.Done(key)
	// 通过syncHandler 处理具体业务逻辑
	err := c.syncHandler(key.(string))
	if err == nil {
		// 标记改元素处理完成
		c.queue.Forget(key)
		return true
	}

	c.logger.WithError(err).WithField("key", key).Error("Error in syncHandler, re-adding item to queue")
	// 再次延迟添加至队列中
	c.queue.AddRateLimited(key)

	return true
}

// 利用MetaNamespaceKeyFunc 获取元素namespace/resourcename 添加进队列
func (c *genericController) enqueue(obj interface{}) {
	key, err := cache.MetaNamespaceKeyFunc(obj)
	if err != nil {
		c.logger.WithError(errors.WithStack(err)).
			Error("Error creating queue key, item not added to queue")
		return
	}

	c.queue.Add(key)
}

```

## 二 backup_controller

backupController 具有以下方法

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20220218145210.png)

### 2.1 初始化

```go
// velero/pkg/controller/backup_controller.go 

//
type backupController struct {
	*genericController
	discoveryHelper             discovery.Helper
	backupper                   pkgbackup.Backupper
	lister                      velerov1listers.BackupLister
	client                      velerov1client.BackupsGetter
	kbClient                    kbclient.Client
	clock                       clock.Clock
	backupLogLevel              logrus.Level
	newPluginManager            func(logrus.FieldLogger) clientmgmt.Manager
	backupTracker               BackupTracker
	defaultBackupLocation       string
	defaultVolumesToRestic      bool
	defaultBackupTTL            time.Duration
	snapshotLocationLister      velerov1listers.VolumeSnapshotLocationLister
	defaultSnapshotLocations    map[string]string
	metrics                     *metrics.ServerMetrics
	backupStoreGetter           persistence.ObjectBackupStoreGetter
	formatFlag                  logging.Format
	volumeSnapshotLister        snapshotv1beta1listers.VolumeSnapshotLister
	volumeSnapshotContentLister snapshotv1beta1listers.VolumeSnapshotContentLister
}

// backupController 构造方法
func NewBackupController(
	backupInformer velerov1informers.BackupInformer,
	client velerov1client.BackupsGetter,
	discoveryHelper discovery.Helper,
	backupper pkgbackup.Backupper,
	logger logrus.FieldLogger,
	backupLogLevel logrus.Level,
	newPluginManager func(logrus.FieldLogger) clientmgmt.Manager,
	backupTracker BackupTracker,
	kbClient kbclient.Client,
	defaultBackupLocation string,
	defaultVolumesToRestic bool,
	defaultBackupTTL time.Duration,
	volumeSnapshotLocationLister velerov1listers.VolumeSnapshotLocationLister,
	defaultSnapshotLocations map[string]string,
	metrics *metrics.ServerMetrics,
	formatFlag logging.Format,
	volumeSnapshotLister snapshotv1beta1listers.VolumeSnapshotLister,
	volumeSnapshotContentLister snapshotv1beta1listers.VolumeSnapshotContentLister,
	backupStoreGetter persistence.ObjectBackupStoreGetter,
) Interface {
	c := &backupController{
		genericController:           newGenericController(Backup, logger),
		discoveryHelper:             discoveryHelper,
		backupper:                   backupper,
		lister:                      backupInformer.Lister(),
		client:                      client,
		clock:                       &clock.RealClock{},
		backupLogLevel:              backupLogLevel,
		newPluginManager:            newPluginManager,
		backupTracker:               backupTracker,
		kbClient:                    kbClient,
		defaultBackupLocation:       defaultBackupLocation,
		defaultVolumesToRestic:      defaultVolumesToRestic,
		defaultBackupTTL:            defaultBackupTTL,
		snapshotLocationLister:      volumeSnapshotLocationLister,
		defaultSnapshotLocations:    defaultSnapshotLocations,
		metrics:                     metrics,
		formatFlag:                  formatFlag,
		volumeSnapshotLister:        volumeSnapshotLister,
		volumeSnapshotContentLister: volumeSnapshotContentLister,
		backupStoreGetter:           backupStoreGetter,
	}
	// 利用外部传入的processBackup 来重新赋值通用控制器的，也就是具体该控制器实现的业务逻辑
	c.syncHandler = c.processBackup
  // 重新同步的方法
	c.resyncFunc = c.resync
	c.resyncPeriod = time.Minute
	
  // 添加backup resourceEventHandler
	backupInformer.Informer().AddEventHandler(
		cache.ResourceEventHandlerFuncs{
			AddFunc: func(obj interface{}) {
        // 首先判断是否是backup对象
				backup := obj.(*velerov1api.Backup)

				switch backup.Status.Phase {
        // 仅处理新的backups
				case "", velerov1api.BackupPhaseNew:
					// only process new backups
				default:
					c.logger.WithFields(logrus.Fields{
						"backup": kubeutil.NamespaceAndName(backup),
						"phase":  backup.Status.Phase,
					}).Debug("Backup is not new, skipping")
					return
				}
				// 获取对象key
				key, err := cache.MetaNamespaceKeyFunc(backup)
				if err != nil {
					c.logger.WithError(err).WithField(Backup, backup).Error("Error creating queue key, item not added to queue")
					return
				}
        // 将其添加进队列
				c.queue.Add(key)
			},
		},
	)

	return c
}
```

### 2.2 processBackup

controller首先运行的processBackup

```go
func (c *backupController) processBackup(key string) error {
	log := c.logger.WithField("key", key)

	log.Debug("Running processBackup")
  // 分割key
	ns, name, err := cache.SplitMetaNamespaceKey(key)
	if err != nil {
		log.WithError(err).Errorf("error splitting key")
		return nil
	}

	log.Debug("Getting backup")
  // 获取备份对象
	original, err := c.lister.Backups(ns).Get(name)
	if apierrors.IsNotFound(err) {
		log.Debugf("backup %s not found", name)
		return nil
	}
	if err != nil {
		return errors.Wrap(err, "error getting backup")
	}
	
  // 检测当前备份的状态，万一有多个控制器实例正在运行，控制器 A 有可能成功将阶段更改为 InProgress，而控制器 B 尝试修补阶段失败。当控制器 B 重新处理相同的备份时，它将显示为 New（通知者尚未看到更新）或 InProgress。在前一种情况下，补丁尝试将再次失败，直到通知者看到更新。在后一种情况下，在通知者看到 InProgress 的更新后，我们仍然需要此检查，因此我们可以返回 nil 以指示我们已完成处理此密钥（即使它是空操作）。
	switch original.Status.Phase {
	case "", velerov1api.BackupPhaseNew:
		// only process new backups
	default:
		return nil
	}
	// 处理prepareBackupRequest
	log.Debug("Preparing backup request")
	request := c.prepareBackupRequest(original)

  // 如果验证正常，标记正在处理
	if len(request.Status.ValidationErrors) > 0 {
		request.Status.Phase = velerov1api.BackupPhaseFailedValidation
	} else {
		request.Status.Phase = velerov1api.BackupPhaseInProgress
		request.Status.StartTimestamp = &metav1.Time{Time: c.clock.Now()}
	}

	// 更新备份任务状态
	updatedBackup, err := patchBackup(original, request.Backup, c.client)
	if err != nil {
		return errors.Wrapf(err, "error updating Backup status to %s", request.Status.Phase)
	}
	// 
	original = updatedBackup
	request.Backup = updatedBackup.DeepCopy()

	if request.Status.Phase == velerov1api.BackupPhaseFailedValidation {
		return nil
	}

	c.backupTracker.Add(request.Namespace, request.Name)
	defer c.backupTracker.Delete(request.Namespace, request.Name)

  // 运行backup
	log.Debug("Running backup")
	// 获取定时备份的名称
	backupScheduleName := request.GetLabels()[velerov1api.ScheduleNameLabel]
	c.metrics.RegisterBackupAttempt(backupScheduleName)

	// 执行并上传备份
	if err := c.runBackup(request); err != nil {
		// 即使 runBackup 在将工件上传到对象存储之前设置了备份的阶段，我们也必须在此处再次检查错误并在发现错误时更新阶段，因为在将工件上传到对象存储时可能会出错，这将导致备份失败。
		log.WithError(err).Error("backup failed")
		request.Status.Phase = velerov1api.BackupPhaseFailed
	}

	switch request.Status.Phase {
	case velerov1api.BackupPhaseCompleted:
		c.metrics.RegisterBackupSuccess(backupScheduleName)
	case velerov1api.BackupPhasePartiallyFailed:
		c.metrics.RegisterBackupPartialFailure(backupScheduleName)
	case velerov1api.BackupPhaseFailed:
		c.metrics.RegisterBackupFailed(backupScheduleName)
	case velerov1api.BackupPhaseFailedValidation:
		c.metrics.RegisterBackupValidationFailure(backupScheduleName)
	}

	log.Debug("Updating backup's final status")
	if _, err := patchBackup(original, request.Backup, c.client); err != nil {
		log.WithError(err).Error("error updating backup's final status")
	}

	return nil
}

```

### 2.3 runBackup

runBackup运行并上传经过验证的备份，此函数返回的任何错误都会导致备份失败；如果为返回备份错误，检查备份状态错误字段以查看备份是否部分失败。

设置临时备份日志文件，设置备份文件，获取备份对象actions，设置备份存储以检查备份是否存在，执行backupController中的传递进来的backupper执行Backup动作，

如果启用EnableCSI，则进行volumeSnapshotLister，和volumeSnapshotContentLister 的List操作获取列表。

记录备份对象状态中Warnings 和 Errors数字，重新实例化备份存储persistBackup，

```go
func (c *backupController) runBackup(backup *pkgbackup.Request) error {
  // 设置备份
	c.logger.WithField(Backup, kubeutil.NamespaceAndName(backup)).Info("Setting up backup log")
	// 制定日志文件
	logFile, err := ioutil.TempFile("", "")
	if err != nil {
		return errors.Wrap(err, "error creating temp file for backup log")
	}
	gzippedLogFile := gzip.NewWriter(logFile)
	// Assuming we successfully uploaded the log file, this will have already been closed below. It is safe to call
	// close multiple times. If we get an error closing this, there's not really anything we can do about it.
	defer gzippedLogFile.Close()
	defer closeAndRemoveFile(logFile, c.logger.WithField(Backup, kubeutil.NamespaceAndName(backup)))

  // 日志包含备份日志和标准控制它输出，日志可以帮助我们排查上传发生了什么事情，同时可以在失败时查看原因。
	logger := logging.DefaultLogger(c.backupLogLevel, c.formatFlag)
	logger.Out = io.MultiWriter(os.Stdout, gzippedLogFile)

	logCounter := logging.NewLogCounterHook()
	logger.Hooks.Add(logCounter)

	backupLog := logger.WithField(Backup, kubeutil.NamespaceAndName(backup))
	// 设置备份临时文件
	backupLog.Info("Setting up backup temp file")
	backupFile, err := ioutil.TempFile("", "")
	if err != nil {
		return errors.Wrap(err, "error creating temp file for backup")
	}
	defer closeAndRemoveFile(backupFile, backupLog)

	backupLog.Info("Setting up plugin manager")
  // 设置plugin manager
	pluginManager := c.newPluginManager(backupLog)
	defer pluginManager.CleanupClients()

	backupLog.Info("Getting backup item actions")
  // 获取备份对象的actions
	actions, err := pluginManager.GetBackupItemActions()
	if err != nil {
		return err
	}
	// 开获取备份存储
	backupLog.Info("Setting up backup store to check for backup existence")
	backupStore, err := c.backupStoreGetter.Get(backup.StorageLocation, pluginManager, backupLog)
	if err != nil {
		return err
	}
	// 如果备份已经存在，或检测发生错误，则返回错误
	exists, err := backupStore.BackupExists(backup.StorageLocation.Spec.StorageType.ObjectStorage.Bucket, backup.Name)
	if exists || err != nil {
		backup.Status.Phase = velerov1api.BackupPhaseFailed
		backup.Status.CompletionTimestamp = &metav1.Time{Time: c.clock.Now()}
		if err != nil {
			return errors.Wrapf(err, "error checking if backup already exists in object storage")
		}
		return errors.Errorf("backup already exists in object storage")
	}
	
  // 开始执行备份，将错误添加到fatalErrs
	var fatalErrs []error
	if err := c.backupper.Backup(backupLog, backup, backupFile, actions, pluginManager); err != nil {
		fatalErrs = append(fatalErrs, err)
	}

  // 判断如果启用EnableCSI
	// Empty slices here so that they can be passed in to the persistBackup call later, regardless of whether or not CSI's enabled.
	// This way, we only make the Lister call if the feature flag's on.
	var volumeSnapshots []*snapshotv1beta1api.VolumeSnapshot
	var volumeSnapshotContents []*snapshotv1beta1api.VolumeSnapshotContent
	if features.IsEnabled(velerov1api.CSIFeatureFlag) {
    // 获取可以为velero.io/backup-name的selector
		selector := label.NewSelectorForBackup(backup.Name)

    // 如果volumeSnapshotLister 不为nil，则根据标签选择将结果存储到volumeSnapshots 列表中
		if c.volumeSnapshotLister != nil {
			volumeSnapshots, err = c.volumeSnapshotLister.List(selector)
			if err != nil {
				backupLog.Error(err)
			}
		}
		// 如果volumeSnapshotContentLister 不为nil，则根据标签将volumeSnapshotCountent 添加进volumeSnapshotContents列表中
		if c.volumeSnapshotContentLister != nil {
			volumeSnapshotContents, err = c.volumeSnapshotContentLister.List(selector)
			if err != nil {
				backupLog.Error(err)
			}
		}
	}

	// 在序列化和上传之前标记完成时间戳。否则，对象存储中的 JSON 文件的 CompletionTimestamp 为“null”。
	backup.Status.CompletionTimestamp = &metav1.Time{Time: c.clock.Now()}
	// 遍历VolumeSnapshots，进行完成VolumeSnapshotsCompleted记数
	backup.Status.VolumeSnapshotsAttempted = len(backup.VolumeSnapshots)
	for _, snap := range backup.VolumeSnapshots {
		if snap.Status.Phase == volume.SnapshotPhaseCompleted {
			backup.Status.VolumeSnapshotsCompleted++
		}
	}
	// 记录backupmetrics
	recordBackupMetrics(backupLog, backup.Backup, backupFile, c.metrics)
	// 关闭日志
	if err := gzippedLogFile.Close(); err != nil {
		c.logger.WithField(Backup, kubeutil.NamespaceAndName(backup)).WithError(err).Error("error closing gzippedLogFile")
	}
	// 记录备份对象状态中Warnings 和 Errors数字
	backup.Status.Warnings = logCounter.GetCount(logrus.WarnLevel)
	backup.Status.Errors = logCounter.GetCount(logrus.ErrorLevel)

	// 将 finalize 阶段分配到尽可能接近结束的位置，以便捕获记录到 backupLog 的任何错误。这是在将工件上传到对象存储之前完成的，以便对象存储中备份的 JSON 表示具有终端阶段集。
	switch {
	case len(fatalErrs) > 0:
		backup.Status.Phase = velerov1api.BackupPhaseFailed
	case logCounter.GetCount(logrus.ErrorLevel) > 0:
		backup.Status.Phase = velerov1api.BackupPhasePartiallyFailed
	default:
		backup.Status.Phase = velerov1api.BackupPhaseCompleted
	}

	// 重新实例化备份存储，因为自原始实例化以来凭据可能已更改，如果这是一个长时间运行的备份
	backupLog.Info("Setting up backup store to persist the backup")
	backupStore, err = c.backupStoreGetter.Get(backup.StorageLocation, pluginManager, backupLog)
	if err != nil {
		return err
	}
	// 执行持久化备份
	if errs := persistBackup(backup, backupFile, logFile, backupStore, c.logger.WithField(Backup, kubeutil.NamespaceAndName(backup)), volumeSnapshots, volumeSnapshotContents); len(errs) > 0 {
		fatalErrs = append(fatalErrs, errs...)
	}

	c.logger.WithField(Backup, kubeutil.NamespaceAndName(backup)).Info("Backup completed")

	// 如果我们返回一个非零错误，调用函数会将备份的阶段更新为失败。
	return kerrors.NewAggregate(fatalErrs)
}
```

### 2.4 resync

resync方法，利用backupController传入的BackupLister 进行list，getLastSuccessBySchedule 为每个计划查找最近完成的备份，并返回计划名称 -> 最近完成备份的完成时间的映射。此映射包含一个用于 ad-hocnon-scheduled 备份的条目，其中键是空字符串。

```go
func (c *backupController) resync() {
	// 重新计算 backup_total 指标
	backups, err := c.lister.List(labels.Everything())
	if err != nil {
		c.logger.Error(err, "Error computing backup_total metric")
	} else {
		c.metrics.SetBackupTotal(int64(len(backups)))
	}

	// 重新计算每个计划的 backup_last_successful_timestamp 指标（包括空计划，即临时备份）
	for schedule, timestamp := range getLastSuccessBySchedule(backups) {
		c.metrics.SetBackupLastSuccessfulTimestamp(schedule, timestamp)
	}
}

// 根据list出来的backups，获取schedule最新的备份
func getLastSuccessBySchedule(backups []*velerov1api.Backup) map[string]time.Time {
	lastSuccessBySchedule := map[string]time.Time{}
	for _, backup := range backups {
		if backup.Status.Phase != velerov1api.BackupPhaseCompleted {
			continue
		}
		if backup.Status.CompletionTimestamp == nil {
			continue
		}

		schedule := backup.Labels[velerov1api.ScheduleNameLabel]
		timestamp := backup.Status.CompletionTimestamp.Time

		if timestamp.After(lastSuccessBySchedule[schedule]) {
			lastSuccessBySchedule[schedule] = timestamp
		}
	}

	return lastSuccessBySchedule
}
```



### 2.5 prepareBackupRequest

对输入参数进行解析并处理，最总防护backup.Request对象

```go
func (c *backupController) prepareBackupRequest(backup *velerov1api.Backup) *pkgbackup.Request {
  // 为了防止修改原始对象，在此进行深度copy
	request := &pkgbackup.Request{
		Backup: backup.DeepCopy(), // don't modify items in the cache
	}

  // 设置备份主版本，新版本使用tatus.FormatVersion
	request.Status.Version = pkgbackup.BackupVersion
	request.Status.FormatVersion = pkgbackup.BackupFormatVersion
	// 设置备份TTL
	if request.Spec.TTL.Duration == 0 {
		request.Spec.TTL.Duration = c.defaultBackupTTL
	}

	// calculate expiration
	request.Status.Expiration = &metav1.Time{Time: c.clock.Now().Add(request.Spec.TTL.Duration)}
	// 默认不使用restic
	if request.Spec.DefaultVolumesToRestic == nil {
		request.Spec.DefaultVolumesToRestic = &c.defaultVolumesToRestic
	}

	// 查找可食用的bsl，如果用户未指定，则使用default的bsl
	var serverSpecified bool
	if request.Spec.StorageLocation == "" {
		// when the user doesn't specify a location, use the server default unless there is an existing BSL marked as default
		// TODO(2.0) c.defaultBackupLocation will be deprecated
		request.Spec.StorageLocation = c.defaultBackupLocation

		locationList, err := storage.ListBackupStorageLocations(context.Background(), c.kbClient, request.Namespace)
		if err == nil {
			for _, location := range locationList.Items {
				if location.Spec.Default {
					request.Spec.StorageLocation = location.Name
					break
				}
			}
		}
		serverSpecified = true
	}

	// get the storage location, and store the BackupStorageLocation API obj on the request
  // 获取storage location，并存储BackupStorageLocation API对象到请求中
	storageLocation := &velerov1api.BackupStorageLocation{}
	if err := c.kbClient.Get(context.Background(), kbclient.ObjectKey{
		Namespace: request.Namespace,
		Name:      request.Spec.StorageLocation,
	}, storageLocation); err != nil {
		if apierrors.IsNotFound(err) {
			if serverSpecified {
				// TODO(2.0) remove this. For now, without mentioning "server default" it could be confusing trying to grasp where the default came from.
				request.Status.ValidationErrors = append(request.Status.ValidationErrors, fmt.Sprintf("an existing backup storage location wasn't specified at backup creation time and the server default '%s' doesn't exist. Please address this issue (see `velero backup-location -h` for options) and create a new backup. Error: %v", request.Spec.StorageLocation, err))
			} else {
				request.Status.ValidationErrors = append(request.Status.ValidationErrors, fmt.Sprintf("an existing backup storage location wasn't specified at backup creation time and the default '%s' wasn't found. Please address this issue (see `velero backup-location -h` for options) and create a new backup. Error: %v", request.Spec.StorageLocation, err))
			}
		} else {
			request.Status.ValidationErrors = append(request.Status.ValidationErrors, fmt.Sprintf("error getting backup storage location: %v", err))
		}
	} else {
    
    // 存储storageLocation API对象到request中
		request.StorageLocation = storageLocation
		// 更新storageLocation 的访问模式
		if request.StorageLocation.Spec.AccessMode == velerov1api.BackupStorageLocationAccessModeReadOnly {
			request.Status.ValidationErrors = append(request.Status.ValidationErrors,
				fmt.Sprintf("backup can't be created because backup storage location %s is currently in read-only mode", request.StorageLocation.Name))
		}
	}
  // 增加label为了更方便的过滤
	if request.Labels == nil {
		request.Labels = make(map[string]string)
	}
	request.Labels[velerov1api.StorageLocationLabel] = label.GetValidName(request.Spec.StorageLocation)

  // 校验并获取vsl并存储VolumeSnapshotLocation API  对象到request中
	if locs, errs := c.validateAndGetSnapshotLocations(request.Backup); len(errs) > 0 {
		request.Status.ValidationErrors = append(request.Status.ValidationErrors, errs...)
	} else {
		request.Spec.VolumeSnapshotLocations = nil
		for _, loc := range locs {
			request.Spec.VolumeSnapshotLocations = append(request.Spec.VolumeSnapshotLocations, loc.Name)
			request.SnapshotLocations = append(request.SnapshotLocations, loc)
		}
	}

  // 获取集群版本的所有信息 - 对未来的跳过级迁移很有用
	if request.Annotations == nil {
		request.Annotations = make(map[string]string)
	}
	request.Annotations[velerov1api.SourceClusterK8sGitVersionAnnotation] = c.discoveryHelper.ServerVersion().String()
	request.Annotations[velerov1api.SourceClusterK8sMajorVersionAnnotation] = c.discoveryHelper.ServerVersion().Major
	request.Annotations[velerov1api.SourceClusterK8sMinorVersionAnnotation] = c.discoveryHelper.ServerVersion().Minor

  // 校验included/excluded 资源
	for _, err := range collections.ValidateIncludesExcludes(request.Spec.IncludedResources, request.Spec.ExcludedResources) {
		request.Status.ValidationErrors = append(request.Status.ValidationErrors, fmt.Sprintf("Invalid included/excluded resource lists: %v", err))
	}

  // 校验included/excluded 名称空间
	for _, err := range collections.ValidateIncludesExcludes(request.Spec.IncludedNamespaces, request.Spec.ExcludedNamespaces) {
		request.Status.ValidationErrors = append(request.Status.ValidationErrors, fmt.Sprintf("Invalid included/excluded namespace lists: %v", err))
	}

	return request
}

```

### 2.6 Backup

在执行runBackup中，内部针对k8s资源对象具体的备份逻辑实现是在velero/pkg/backup 目录下



```go
//velero/pkg/backup/backup.go

func (kb *kubernetesBackupper) Backup(log logrus.FieldLogger, backupRequest *Request, backupFile io.Writer, actions []velero.BackupItemAction, volumeSnapshotterGetter VolumeSnapshotterGetter) error {
  // 获取压缩写入对象tw
	gzippedData := gzip.NewWriter(backupFile)
	defer gzippedData.Close()

	tw := tar.NewWriter(gzippedData)
	defer tw.Close()

	log.Info("Writing backup version file")
	if err := kb.writeBackupVersion(tw); err != nil {
		return errors.WithStack(err)
	}
	// 获取includesexcludes的namespace和resource并赋值给backuprequest
	backupRequest.NamespaceIncludesExcludes = getNamespaceIncludesExcludes(backupRequest.Backup)
	log.Infof("Including namespaces: %s", backupRequest.NamespaceIncludesExcludes.IncludesString())
	log.Infof("Excluding namespaces: %s", backupRequest.NamespaceIncludesExcludes.ExcludesString())

	backupRequest.ResourceIncludesExcludes = collections.GetResourceIncludesExcludes(kb.discoveryHelper, backupRequest.Spec.IncludedResources, backupRequest.Spec.ExcludedResources)
	log.Infof("Including resources: %s", backupRequest.ResourceIncludesExcludes.IncludesString())
	log.Infof("Excluding resources: %s", backupRequest.ResourceIncludesExcludes.ExcludesString())
	log.Infof("Backing up all pod volumes using restic: %t", *backupRequest.Backup.Spec.DefaultVolumesToRestic)

	var err error
  // 获取resourceHooks并赋值给backupRequest
	backupRequest.ResourceHooks, err = getResourceHooks(backupRequest.Spec.Hooks.Resources, kb.discoveryHelper)
	if err != nil {
		return err
	}
	// 获取bresolvedAction并赋值给backupRequest
	backupRequest.ResolvedActions, err = resolveActions(actions, kb.discoveryHelper)
	if err != nil {
		return err
	}

	backupRequest.BackedUpItems = map[itemKey]struct{}{}

	podVolumeTimeout := kb.resticTimeout
	if val := backupRequest.Annotations[velerov1api.PodVolumeOperationTimeoutAnnotation]; val != "" {
		parsed, err := time.ParseDuration(val)
		if err != nil {
			log.WithError(errors.WithStack(err)).Errorf("Unable to parse pod volume timeout annotation %s, using server value.", val)
		} else {
			podVolumeTimeout = parsed
		}
	}

	ctx, cancelFunc := context.WithTimeout(context.Background(), podVolumeTimeout)
	defer cancelFunc()
	// 获取resticBackupper
	var resticBackupper restic.Backupper
	if kb.resticBackupperFactory != nil {
		resticBackupper, err = kb.resticBackupperFactory.NewBackupper(ctx, backupRequest.Backup)
		if err != nil {
			return errors.WithStack(err)
		}
	}

	// set up a temp dir for the itemCollector to use to temporarily
	// store items as they're scraped from the API.
  // 生成空目录来存储备份文件
	tempDir, err := ioutil.TempDir("", "")
	if err != nil {
		return errors.Wrap(err, "error creating temp dir for backup")
	}
	defer os.RemoveAll(tempDir)
	// 生成具体collector 对象
	collector := &itemCollector{
		log:                   log,
		backupRequest:         backupRequest,
		discoveryHelper:       kb.discoveryHelper,
		dynamicFactory:        kb.dynamicFactory,
		cohabitatingResources: cohabitatingResources(),
		dir:                   tempDir,
	}
	// 获取所有item
	items := collector.getAllItems()
	log.WithField("progress", "").Infof("Collected %d items matching the backup spec from the Kubernetes API (actual number of items backed up may be more or less depending on velero.io/exclude-from-backup annotation, plugins returning additional related items to back up, etc.)", len(items))

	backupRequest.Status.Progress = &velerov1api.BackupProgress{TotalItems: len(items)}
	patch := fmt.Sprintf(`{"status":{"progress":{"totalItems":%d}}}`, len(items))
	if _, err := kb.backupClient.Backups(backupRequest.Namespace).Patch(context.TODO(), backupRequest.Name, types.MergePatchType, []byte(patch), metav1.PatchOptions{}); err != nil {
		log.WithError(errors.WithStack((err))).Warn("Got error trying to update backup's status.progress.totalItems")
	}

	itemBackupper := &itemBackupper{
		backupRequest:           backupRequest,
		tarWriter:               tw,
		dynamicFactory:          kb.dynamicFactory,
		discoveryHelper:         kb.discoveryHelper,
		resticBackupper:         resticBackupper,
		resticSnapshotTracker:   newPVCSnapshotTracker(),
		volumeSnapshotterGetter: volumeSnapshotterGetter,
		itemHookHandler: &hook.DefaultItemHookHandler{
			PodCommandExecutor: kb.podCommandExecutor,
		},
	}

	// helper struct to send current progress between the main
	// backup loop and the gouroutine that periodically patches
	// the backup CR with progress updates
	type progressUpdate struct {
		totalItems, itemsBackedUp int
	}

	// the main backup process will send on this channel once
	// for every item it processes.
	update := make(chan progressUpdate)

	// the main backup process will send on this channel when
	// it's done sending progress updates
	quit := make(chan struct{})

	// This is the progress updater goroutine that receives
	// progress updates on the 'update' channel. It patches
	// the backup CR with progress updates at most every second,
	// but it will not issue a patch if it hasn't received a new
	// update since the previous patch. This goroutine exits
	// when it receives on the 'quit' channel.
	go func() {
		ticker := time.NewTicker(1 * time.Second)
		var lastUpdate *progressUpdate
		for {
			select {
			case <-quit:
				ticker.Stop()
				return
			case val := <-update:
				lastUpdate = &val
			case <-ticker.C:
				if lastUpdate != nil {
					backupRequest.Status.Progress.TotalItems = lastUpdate.totalItems
					backupRequest.Status.Progress.ItemsBackedUp = lastUpdate.itemsBackedUp

					patch := fmt.Sprintf(`{"status":{"progress":{"totalItems":%d,"itemsBackedUp":%d}}}`, lastUpdate.totalItems, lastUpdate.itemsBackedUp)
					if _, err := kb.backupClient.Backups(backupRequest.Namespace).Patch(context.TODO(), backupRequest.Name, types.MergePatchType, []byte(patch), metav1.PatchOptions{}); err != nil {
						log.WithError(errors.WithStack((err))).Warn("Got error trying to update backup's status.progress")
					}
					lastUpdate = nil
				}
			}
		}
	}()

	backedUpGroupResources := map[schema.GroupResource]bool{}
	totalItems := len(items)

	for i, item := range items {
		log.WithFields(map[string]interface{}{
			"progress":  "",
			"resource":  item.groupResource.String(),
			"namespace": item.namespace,
			"name":      item.name,
		}).Infof("Processing item")

		// use an anonymous func so we can defer-close/remove the file
		// as soon as we're done with it
		func() {
			var unstructured unstructured.Unstructured

			f, err := os.Open(item.path)
			if err != nil {
				log.WithError(errors.WithStack(err)).Error("Error opening file containing item")
				return
			}
			defer f.Close()
			defer os.Remove(f.Name())

			if err := json.NewDecoder(f).Decode(&unstructured); err != nil {
				log.WithError(errors.WithStack(err)).Error("Error decoding JSON from file")
				return
			}

			if backedUp := kb.backupItem(log, item.groupResource, itemBackupper, &unstructured, item.preferredGVR); backedUp {
				backedUpGroupResources[item.groupResource] = true
			}
		}()

		// updated total is computed as "how many items we've backed up so far, plus
		// how many items we know of that are remaining"
		totalItems = len(backupRequest.BackedUpItems) + (len(items) - (i + 1))

		// send a progress update
		update <- progressUpdate{
			totalItems:    totalItems,
			itemsBackedUp: len(backupRequest.BackedUpItems),
		}

		log.WithFields(map[string]interface{}{
			"progress":  "",
			"resource":  item.groupResource.String(),
			"namespace": item.namespace,
			"name":      item.name,
		}).Infof("Backed up %d items out of an estimated total of %d (estimate will change throughout the backup)", len(backupRequest.BackedUpItems), totalItems)
	}

	// no more progress updates will be sent on the 'update' channel
	quit <- struct{}{}

	// back up CRD for resource if found. We should only need to do this if we've backed up at least
	// one item for the resource and IncludeClusterResources is nil. If IncludeClusterResources is false
	// we don't want to back it up, and if it's true it will already be included.
	if backupRequest.Spec.IncludeClusterResources == nil {
		for gr := range backedUpGroupResources {
			kb.backupCRD(log, gr, itemBackupper)
		}
	}

	// do a final update on progress since we may have just added some CRDs and may not have updated
	// for the last few processed items.
	backupRequest.Status.Progress.TotalItems = len(backupRequest.BackedUpItems)
	backupRequest.Status.Progress.ItemsBackedUp = len(backupRequest.BackedUpItems)

	patch = fmt.Sprintf(`{"status":{"progress":{"totalItems":%d,"itemsBackedUp":%d}}}`, len(backupRequest.BackedUpItems), len(backupRequest.BackedUpItems))
	if _, err := kb.backupClient.Backups(backupRequest.Namespace).Patch(context.TODO(), backupRequest.Name, types.MergePatchType, []byte(patch), metav1.PatchOptions{}); err != nil {
		log.WithError(errors.WithStack((err))).Warn("Got error trying to update backup's status.progress")
	}

	log.WithField("progress", "").Infof("Backed up a total of %d items", len(backupRequest.BackedUpItems))

	return nil
}
```







































