---
title: k8s pv pvc 源码
comments: true

---

> 2022-08-17 更新

通过 kubernetes 设计模式，去理解整个过程，当时写下这篇文章时，代码只是代码，没有任何结构上的思考，不知道为什么这样设计，只知道创建一个 PVC 需要执行一系列的步骤。

结合 **informer listAndWatch** 机制，理解起来会更加清晰，更加具有普适性。

---

最近在做集群对接存储的工作，期间经历了很多挫折，遇到了很多问题，从一知半解到了解，从纸上谈兵到实战演练，环环相扣。在开始前我先抛出两个问题，带着问题去思考收获可能会更多。

- 动态配置中，创建 PVC 后，是先有 PV 还是现有 Volume
- PV 与 PVC 怎么样才算绑定到一起

<!--more-->

从 *Dynamic Provisioning Storage* 搞起，基本概念前面文章都有提到过 [k8s-storage](./k8s-storage.md) 所以在次就不再赘述。

负责 PV PVC 的 controller 为 `PersistentVolumeController`

```go
func NewController(p ControllerParameters) (*PersistentVolumeController, error) {
	eventRecorder := p.EventRecorder
	if eventRecorder == nil {
		broadcaster := record.NewBroadcaster()
		broadcaster.StartStructuredLogging(0)
		broadcaster.StartRecordingToSink(&v1core.EventSinkImpl{Interface: p.KubeClient.CoreV1().Events("")})
		eventRecorder = broadcaster.NewRecorder(scheme.Scheme, v1.EventSource{Component: "persistentvolume-controller"})
	}

	controller := &PersistentVolumeController{
		volumes:                       newPersistentVolumeOrderedIndex(),
		claims:                        cache.NewStore(cache.DeletionHandlingMetaNamespaceKeyFunc),
		kubeClient:                    p.KubeClient,
		eventRecorder:                 eventRecorder,
		runningOperations:             goroutinemap.NewGoRoutineMap(true /* exponentialBackOffOnError */),
		cloud:                         p.Cloud,
		enableDynamicProvisioning:     p.EnableDynamicProvisioning,
		clusterName:                   p.ClusterName,
		createProvisionedPVRetryCount: createProvisionedPVRetryCount,
		createProvisionedPVInterval:   createProvisionedPVInterval,
        // pvc 对应的 workerqueue
		claimQueue:                    workqueue.NewNamed("claims"),
        // pv 对应的 workerqueue
		volumeQueue:                   workqueue.NewNamed("volumes"),
		resyncPeriod:                  p.SyncPeriod,
		operationTimestamps:           metrics.NewOperationStartTimeCache(),
	}

	// Prober is nil because PV is not aware of Flexvolume.
	if err := controller.volumePluginMgr.InitPlugins(p.VolumePlugins, nil /* prober */, controller); err != nil {
		return nil, fmt.Errorf("could not initialize volume plugins for PersistentVolume Controller: %w", err)
	}

    // informer 从 etcd 那里获取到的 事件-对象 根据事件类型处罚相应事件
	p.VolumeInformer.Informer().AddEventHandler(
		cache.ResourceEventHandlerFuncs{
			AddFunc:    func(obj interface{}) { controller.enqueueWork(controller.volumeQueue, obj) },
			UpdateFunc: func(oldObj, newObj interface{}) { controller.enqueueWork(controller.volumeQueue, newObj) },
			DeleteFunc: func(obj interface{}) { controller.enqueueWork(controller.volumeQueue, obj) },
		},
	)
    // 从 apiserver 获取数据同步到本地 informer 缓存中
	controller.volumeLister = p.VolumeInformer.Lister()
    // 用来确定是否同步过
	controller.volumeListerSynced = p.VolumeInformer.Informer().HasSynced

	p.ClaimInformer.Informer().AddEventHandler(
		cache.ResourceEventHandlerFuncs{
			AddFunc:    func(obj interface{}) { controller.enqueueWork(controller.claimQueue, obj) },
			UpdateFunc: func(oldObj, newObj interface{}) { controller.enqueueWork(controller.claimQueue, newObj) },
			DeleteFunc: func(obj interface{}) { controller.enqueueWork(controller.claimQueue, obj) },
		},
	)
 
    
	controller.claimLister = p.ClaimInformer.Lister()
	controller.claimListerSynced = p.ClaimInformer.Informer().HasSynced

	controller.classLister = p.ClassInformer.Lister()
	controller.classListerSynced = p.ClassInformer.Informer().HasSynced
	controller.podLister = p.PodInformer.Lister()
	controller.podIndexer = p.PodInformer.Informer().GetIndexer()
	controller.podListerSynced = p.PodInformer.Informer().HasSynced
	controller.NodeLister = p.NodeInformer.Lister()
	controller.NodeListerSynced = p.NodeInformer.Informer().HasSynced

	// This custom indexer will index pods by its PVC keys. Then we don't need
	// to iterate all pods every time to find pods which reference given PVC.
	if err := common.AddPodPVCIndexerIfNotPresent(controller.podIndexer); err != nil {
		return nil, fmt.Errorf("could not initialize attach detach controller: %w", err)
	}

	csiTranslator := csitrans.New()
	controller.translator = csiTranslator
	controller.csiMigratedPluginManager = csimigration.NewPluginManager(csiTranslator, utilfeature.DefaultFeatureGate)

	controller.filteredDialOptions = p.FilteredDialOptions

	return controller, nil
}
```

以上 `controller` 初始化完成，就可以启动了

```go
func (ctrl *PersistentVolumeController) Run(ctx context.Context) {
	defer utilruntime.HandleCrash()
	defer ctrl.claimQueue.ShutDown()
	defer ctrl.volumeQueue.ShutDown()

	klog.Infof("Starting persistent volume controller")
	defer klog.Infof("Shutting down persistent volume controller")
	
	if !cache.WaitForNamedCacheSync("persistent volume", ctx.Done(), ctrl.volumeListerSynced, ctrl.claimListerSynced, ctrl.classListerSynced, ctrl.podListerSynced, ctrl.NodeListerSynced) {
		return
	}

    // 初始化本地缓存，与 apiserver 交互，apiserver 从 etcd 中取出数据
    // 然后将数据返回给 informer
	ctrl.initializeCaches(ctrl.volumeLister, ctrl.claimLister)

    // 同步本地缓存中的数据到 worker queue 中
	go wait.Until(ctrl.resync, ctrl.resyncPeriod, ctx.Done())
    // pv worker
	go wait.UntilWithContext(ctx, ctrl.volumeWorker, time.Second)
    // pvc worker
	go wait.UntilWithContext(ctx, ctrl.claimWorker, time.Second)

	metrics.Register(ctrl.volumes.store, ctrl.claims, &ctrl.volumePluginMgr)

	<-ctx.Done()
}
```



```go
// resync 的作用也是从 apiserver 同步数据到本地缓存，
// 多了一步将数据更新到 work queue 
func (ctrl *PersistentVolumeController) resync() {
    klog.V(4).Infof("resyncing PV controller")
	
    pvcs, err := ctrl.claimLister.List(labels.NewSelector())
    if err != nil {
        klog.Warningf("cannot list claims: %s", err)
        return
    }
    for _, pvc := range pvcs {
        ctrl.enqueueWork(ctrl.claimQueue, pvc)
    }

    pvs, err := ctrl.volumeLister.List(labels.NewSelector())
    if err != nil {
        klog.Warningf("cannot list persistent volumes: %s", err)
        return
    }
    for _, pv := range pvs {
        ctrl.enqueueWork(ctrl.volumeQueue, pv)
    }
}
```

剩下的就是研究这个消费者的处理逻辑是什么，以动态配置创建 PVC 为例。



在 PVC 创建阶段我们主要关注这个地方：

```go
// syncClaim is the main controller method to decide what to do with a claim.
// It's invoked by appropriate cache.Controller callbacks when a claim is
// created, updated or periodically synced. We do not differentiate between
// these events.
// For easier readability, it was split into syncUnboundClaim and syncBoundClaim
// methods.
func (ctrl *PersistentVolumeController) syncClaim(ctx context.Context, claim *v1.PersistentVolumeClaim) error {
    klog.V(4).Infof("synchronizing PersistentVolumeClaim[%s]: %s", claimToClaimKey(claim), getClaimStatusForLogging(claim))

    // Set correct "migrated-to" annotations on PVC and update in API server if
    // necessary
    newClaim, err := ctrl.updateClaimMigrationAnnotations(ctx, claim)
    if err != nil {
        // Nothing was saved; we will fall back into the same
        // condition in the next call to this method
        return err
    }
    claim = newClaim
	
    // 这里只区分了两种状态的 pvc 未绑定，已绑定
    // 如果是已绑定的 pvc 会被打上 pv.kubernetes.io/bind-completed 的标签
    if !metav1.HasAnnotation(claim.ObjectMeta, storagehelpers.AnnBindCompleted) {
        return ctrl.syncUnboundClaim(ctx, claim)
    } else {
        return ctrl.syncBoundClaim(claim)
    }
}
```

未绑定的 PVC 自然走的就是 `ctrl.syncUnboundClaim` 函数：

```go
// syncUnboundClaim is the main controller method to decide what to do with an
// unbound claim.
func (ctrl *PersistentVolumeController) syncUnboundClaim(ctx context.Context, claim *v1.PersistentVolumeClaim) error {
    // pvc is "Pending"
    if claim.Spec.VolumeName == "" {
        // 绑定模式，waitforfirstcustomer or immediate
        delayBinding, err := storagehelpers.IsDelayBindingMode(claim, ctrl.classLister)
        if err != nil {
            return err
        }

        // 查一下是否有存在并且满足 PVC 中条件的 PV
        volume, err := ctrl.volumes.findBestMatchForClaim(claim, delayBinding)
        if err != nil {
            klog.V(2).Infof("synchronizing unbound PersistentVolumeClaim[%s]: Error finding PV for claim: %v", claimToClaimKey(claim), err)
            return fmt.Errorf("error finding PV for claim %q: %w", claimToClaimKey(claim), err)
        }
        if volume == nil {
            klog.V(4).Infof("synchronizing unbound PersistentVolumeClaim[%s]: no volume found", claimToClaimKey(claim))
            // No PV could be found
            // OBSERVATION: pvc is "Pending", will retry
            switch {
                // 延迟绑定
                case delayBinding && !storagehelpers.IsDelayBindingProvisioning(claim):
                    if err = ctrl.emitEventForUnboundDelayBindingClaim(claim); err != nil {
                        return err
                    }
                // 获取 pvc 的 storageclass
                case storagehelpers.GetPersistentVolumeClaimClass(claim) != "":
                    if err = ctrl.provisionClaim(ctx, claim); err != nil {
                        return err
                    }
                    return nil
                default:
                	ctrl.eventRecorder.Event(claim, v1.EventTypeNormal, events.FailedBinding, "no persistent volumes available for this claim and no storage class is set")
            }

            // Mark the claim as Pending and try to find a match in the next
            // periodic syncClaim
            if _, err = ctrl.updateClaimStatus(claim, v1.ClaimPending, nil); err != nil {
                return err
            }
            return nil
        } else /* pv != nil */ {
            // Found a PV for this claim
            // OBSERVATION: pvc is "Pending", pv is "Available"
            claimKey := claimToClaimKey(claim)
            klog.V(4).Infof("synchronizing unbound PersistentVolumeClaim[%s]: volume %q found: %s", claimKey, volume.Name, getVolumeStatusForLogging(volume))
            if err = ctrl.bind(volume, claim); err != nil {
                // On any error saving the volume or the claim, subsequent
                // syncClaim will finish the binding.
                // record count error for provision if exists
                // timestamp entry will remain in cache until a success binding has happened
                metrics.RecordMetric(claimKey, &ctrl.operationTimestamps, err)
                return err
            }
            // OBSERVATION: claim is "Bound", pv is "Bound"
            // if exists a timestamp entry in cache, record end to end provision latency and clean up cache
            // End of the provision + binding operation lifecycle, cache will be cleaned by "RecordMetric"
            // [Unit test 12-1, 12-2, 12-4]
            metrics.RecordMetric(claimKey, &ctrl.operationTimestamps, nil)
            return nil
        }
    } else /* pvc.Spec.VolumeName != nil */ {
        // [Unit test set 2]
        // User asked for a specific PV.
        klog.V(4).Infof("synchronizing unbound PersistentVolumeClaim[%s]: volume %q requested", claimToClaimKey(claim), claim.Spec.VolumeName)
        obj, found, err := ctrl.volumes.store.GetByKey(claim.Spec.VolumeName)
        if err != nil {
            return err
        }
        if !found {
            // User asked for a PV that does not exist.
            // OBSERVATION: pvc is "Pending"
            // Retry later.
            klog.V(4).Infof("synchronizing unbound PersistentVolumeClaim[%s]: volume %q requested and not found, will try again next time", claimToClaimKey(claim), claim.Spec.VolumeName)
            if _, err = ctrl.updateClaimStatus(claim, v1.ClaimPending, nil); err != nil {
                return err
            }
            return nil
        } else {
            volume, ok := obj.(*v1.PersistentVolume)
            if !ok {
                return fmt.Errorf("cannot convert object from volume cache to volume %q!?: %+v", claim.Spec.VolumeName, obj)
            }
            klog.V(4).Infof("synchronizing unbound PersistentVolumeClaim[%s]: volume %q requested and found: %s", claimToClaimKey(claim), claim.Spec.VolumeName, getVolumeStatusForLogging(volume))
            if volume.Spec.ClaimRef == nil {
                // User asked for a PV that is not claimed
                // OBSERVATION: pvc is "Pending", pv is "Available"
                klog.V(4).Infof("synchronizing unbound PersistentVolumeClaim[%s]: volume is unbound, binding", claimToClaimKey(claim))
                if err = checkVolumeSatisfyClaim(volume, claim); err != nil {
                    klog.V(4).Infof("Can't bind the claim to volume %q: %v", volume.Name, err)
                    // send an event
                    msg := fmt.Sprintf("Cannot bind to requested volume %q: %s", volume.Name, err)
                    ctrl.eventRecorder.Event(claim, v1.EventTypeWarning, events.VolumeMismatch, msg)
                    // volume does not satisfy the requirements of the claim
                    if _, err = ctrl.updateClaimStatus(claim, v1.ClaimPending, nil); err != nil {
                        return err
                    }
                } else if err = ctrl.bind(volume, claim); err != nil {
                    // On any error saving the volume or the claim, subsequent
                    // syncClaim will finish the binding.
                    return err
                }
                // OBSERVATION: pvc is "Bound", pv is "Bound"
                return nil
            } else if storagehelpers.IsVolumeBoundToClaim(volume, claim) {
                // User asked for a PV that is claimed by this PVC
                // OBSERVATION: pvc is "Pending", pv is "Bound"
                klog.V(4).Infof("synchronizing unbound PersistentVolumeClaim[%s]: volume already bound, finishing the binding", claimToClaimKey(claim))

                // Finish the volume binding by adding claim UID.
                if err = ctrl.bind(volume, claim); err != nil {
                    return err
                }
                // OBSERVATION: pvc is "Bound", pv is "Bound"
                return nil
            } else {
                // User asked for a PV that is claimed by someone else
                // OBSERVATION: pvc is "Pending", pv is "Bound"
                if !metav1.HasAnnotation(claim.ObjectMeta, storagehelpers.AnnBoundByController) {
                    klog.V(4).Infof("synchronizing unbound PersistentVolumeClaim[%s]: volume already bound to different claim by user, will retry later", claimToClaimKey(claim))
                    claimMsg := fmt.Sprintf("volume %q already bound to a different claim.", volume.Name)
                    ctrl.eventRecorder.Event(claim, v1.EventTypeWarning, events.FailedBinding, claimMsg)
                    // User asked for a specific PV, retry later
                    if _, err = ctrl.updateClaimStatus(claim, v1.ClaimPending, nil); err != nil {
                        return err
                    }
                    return nil
                } else {
                    // This should never happen because someone had to remove
                    // AnnBindCompleted annotation on the claim.
                    klog.V(4).Infof("synchronizing unbound PersistentVolumeClaim[%s]: volume already bound to different claim %q by controller, THIS SHOULD NEVER HAPPEN", claimToClaimKey(claim), claimrefToClaimKey(volume.Spec.ClaimRef))
                    claimMsg := fmt.Sprintf("volume %q already bound to a different claim.", volume.Name)
                    ctrl.eventRecorder.Event(claim, v1.EventTypeWarning, events.FailedBinding, claimMsg)

                    return fmt.Errorf("invalid binding of claim %q to volume %q: volume already claimed by %q", claimToClaimKey(claim), claim.Spec.VolumeName, claimrefToClaimKey(volume.Spec.ClaimRef))
                }
            }
        }
    }
}
```

逻辑上其实一点难度都没，注释写的也很详细（我想加注释的时候发现有点冗余），错误处理方式没有任何简化，官方称为：`space shuttle style`，而且另一点值得学习的地方：

```go
if volume == nil {
    // logic..
} else /* pv != nil */ { 
    // logic..
}
```

在 else 中填写注释，当我们把 if 代码块折叠起来时，就是这样的：

```go
if volume == nil {} else /* pv != nil */ {}
```

个人觉得这种注释方式挺好，当 if 代码块中逻辑特别长时，可能就忘了其中的条件判断..看到 else 时，直接就一脸懵，这是哪个 if 的 else..



然后我们接着看配置 PVC 的部分：

```go
// provisionClaim starts new asynchronous operation to provision a claim if
// provisioning is enabled.
func (ctrl *PersistentVolumeController) provisionClaim(ctx context.Context, claim *v1.PersistentVolumeClaim) error {
    if !ctrl.enableDynamicProvisioning {
        return nil
    }
    klog.V(4).Infof("provisionClaim[%s]: started", claimToClaimKey(claim))
    opName := fmt.Sprintf("provision-%s[%s]", claimToClaimKey(claim), string(claim.UID))
	// 外部存储情况 plugin == nil
    plugin, storageClass, err := ctrl.findProvisionablePlugin(claim)
    // findProvisionablePlugin does not return err for external provisioners
    if err != nil {
        ctrl.eventRecorder.Event(claim, v1.EventTypeWarning, events.ProvisioningFailed, err.Error())
        klog.Errorf("error finding provisioning plugin for claim %s: %v", claimToClaimKey(claim), err)
        // failed to find the requested provisioning plugin, directly return err for now.
        // controller will retry the provisioning in every syncUnboundClaim() call
        // retain the original behavior of returning nil from provisionClaim call
        return nil
    }
    ctrl.scheduleOperation(opName, func() error {
        // create a start timestamp entry in cache for provision operation if no one exists with
        // key = claimKey, pluginName = provisionerName, operation = "provision"
        claimKey := claimToClaimKey(claim)
        ctrl.operationTimestamps.AddIfNotExist(claimKey, ctrl.getProvisionerName(plugin, storageClass), "provision")
        var err error
        if plugin == nil {
            // 外部存储，把 provisioner 填到 PVC 中
            _, err = ctrl.provisionClaimOperationExternal(ctx, claim, storageClass)
        } else {
            _, err = ctrl.provisionClaimOperation(ctx, claim, plugin, storageClass)
        }
        // if error happened, record an error count metric
        // timestamp entry will remain in cache until a success binding has happened
        if err != nil {
            metrics.RecordMetric(claimKey, &ctrl.operationTimestamps, err)
        }
        return err
    })
    return nil
}
```

又发现一个问题，这里没有看到任何调用外部存储的地方，如果是 plugin == nil 只是将 storageClass 中的 provisioner 

存储到了 PVC 中。



还有一点需要注意的是，存储插件在 kube controller manager 启动的时候就初始化好了，当我们使用外部存储的时候，集群如何使用它呢？

```sh
# kube controller manager 日志
Loaded volume plugin "kubernetes.io/portworx-volume"
Loaded volume plugin "kubernetes.io/scaleio"
Loaded volume plugin "kubernetes.io/local-volume"
Loaded volume plugin "kubernetes.io/storageos"
Loaded volume plugin "kubernetes.io/csi"
# ...
```

我一开始使用内置的 `kubernetes.io/rbd` 对接 ceph ，但是会出现 pvc 无限处于 pending 状态的问题，**最难受的是 events 中没有任何信息**，后来运维同事帮忙排查出来是内置在 kube controller manager 容器中的 ceph 客户端版本问题。所以，最后采用了实现了 CSI 接口的 rbd 插件 `rbd.csi.ceph.com` 所以我这里的场景是 plugin == nil：

```go
func (ctrl *PersistentVolumeController) findProvisionablePlugin(claim *v1.PersistentVolumeClaim) (vol.ProvisionableVolumePlugin, *storage.StorageClass, error) {
    // provisionClaim() which leads here is never called with claimClass=="", we
    // can save some checks.
    claimClass := storagehelpers.GetPersistentVolumeClaimClass(claim)
    class, err := ctrl.classLister.Get(claimClass)
    if err != nil {
        return nil, nil, err
    }

    // Find a plugin for the class
    if ctrl.csiMigratedPluginManager.IsMigrationEnabledForPlugin(class.Provisioner) {
        // CSI migration scenario - do not depend on in-tree plugin
        return nil, class, nil
    }
    plugin, err := ctrl.volumePluginMgr.FindProvisionablePluginByName(class.Provisioner)
    if err != nil {
        // 不是以 kubernetes.io/ 开头的
        if !strings.HasPrefix(class.Provisioner, "kubernetes.io/") {
            // External provisioner is requested, do not report error
            // 外部存储只返回了 sc
            return nil, class, nil
        }
        return nil, class, err
    }
    return plugin, class, nil
}
```

针对 k8s 从哪里发出去的请求这个问题，从 k8s 的源码中没有找到答案，但是！源码中的注释值得让人揣摩，在填好 provisioner 之后，下面的注释就说：

```go
// External provisioner has been requested for provisioning the volume
// Report an event and wait for external provisioner to finish
```

所以我就猜测，是不是在 CSI 插件中有一个定时任务，去监控PVC列表，然后根据PVC去创建卷。事实也确实是这样:

![图片来自：极客时间](https://static001.geekbang.org/resource/image/d4/ad/d4bdc7035f1286e7a423da851eee89ad.png?wh=1880*941)

CSI 插件体系的设计思想，就是把这个 Provision 阶段，以及 Kubernetes 里的一部分存储管理功能，从主干代码里剥离出来，做成了几个单独的组件。



### External Components

- Dirver Registrar

  Driver Registrar 组件，负责将插件注册到 kubelet 里面（这可以类比为，将可执行文件放在插件目录下）。而在具体实现上，Driver Registrar 需要请求 CSI 插件的 Identity 服务来获取插件信息。

  

- External Provisoiner

  External Provisioner 组件，负责的正是 Provision 阶段。在具体实现上，External Provisioner 监听（Watch）了 APIServer 里的 PVC 对象。当一个 PVC 被创建时，它就会调用 CSI Controller 的 CreateVolume 方法，为你创建对应 PV。

  从图中也能看出来，并不是 K8S 直接与 CSI 插件，而是通过 External Provisoiner 与 CSI 插件通信，所以也证实了我们刚刚的猜想，肯定是有某个东西鉴定 PVC 列表了。

  

- External Attacher

  External Attacher 组件，负责的是将卷 Attach 的 Node 上的操作。在具体实现上，它监听了 APIServer 里 VolumeAttachment 对象的变化。VolumeAttachment 对象是 Kubernetes 确认一个 Volume 可以进入“Attach 阶段”的重要标志。

> Volume Mount 的阶段并不属于，External Components 的职责，当 kubelet 的 VolumeManagerReconciler 控制循环检查到它需要执行 Mount 操作的时候，会通过 pkg/volume/csi 包，直接调用 CSI Node 服务完成 Volume 的“Mount 阶段”。

### Custom Components

- CSI Identity
- CSI Controller
- CSI Node

### 静态配置流程（两阶段处理）

#### Attach

当一个 Pod 被调度到某个节点上时，Node 上的 Kubelet 就会为这个 Pod 创建它的 Volume 目录，形如：/var/lib/kubelet/pods/<Pod的ID>/volumes/kubernetes.io~<Volume类型>/<Volume名字>

如果说我们用的是远程存储系统，kubelet 会调用 API 接口创建远程持久卷，然后将其挂载到Pod所在的Node上。

#### Mount

mount 的操作就是将Attach阶段创建出来的卷，挂载到 Pod 的 volume 目录。



疑问：

1.每个 Node 上都需要运行一个CSI plugin 吗？纠结于这一点的原因是：没有搞清楚 mount 操作是谁负责的。

解答：由于是 kubelet 直接调用 CSI 插件，所以需要在每一个Node 上都部署一个 CSI 插件，和 kubelet 一对一的部署起来。
