

# external provisioner V1.6.0

在Provisioner 的入口函数中，实例化了 ProvisionerController，然后调用 Run 函数启动所有的 controller

external-provisioner/cmd/csi-provisioner/csi-provisioner.go

```
func main() {
    ....
    csiProvisioner := ctrl.NewCSIProvisioner(
    ....
    )
    ....
    provisionController = controller.NewProvisionController(
        clientset,
	provisionerName,
	csiProvisioner,
	provisionerOptions...,
    )  
    ....
    // 启动所有的 controller
    provisionController.Run(ctx)
    ....
}

```

Run 方法启动了 runVolumeWorker, 这个方法循环调用方法 processNextVolumeWorkItem，直到这个方法返回 false。

```

func (ctrl *ProvisionController) Run(_ <-chan struct{}) {
	...
	run := func(ctx context.Context) {
	    ....
	    for i := 0; i < ctrl.threadiness; i++ {
		go wait.Until(ctrl.runClaimWorker, time.Second, context.TODO().Done())
		go wait.Until(ctrl.runVolumeWorker, time.Second, context.TODO().Done())
	    }
	    ....
	}
}

func (ctrl *ProvisionController) runVolumeWorker() {
	for ctrl.processNextVolumeWorkItem() {
	}
}

// processNextVolumeWorkItem 处理来自 volumeQueue 的 items，只有 volumeQueue 被 shutdown 了，才返回 false
func (ctrl *ProvisionController) processNextVolumeWorkItem() bool {
	obj, shutdown := ctrl.volumeQueue.Get()

	if shutdown {
		return false
	}

	err := func(obj interface{}) error {
	    	....
		if err := ctrl.syncVolumeHandler(key); err != nil {
			if ctrl.failedDeleteThreshold == 0 {
				glog.Warningf("Retrying syncing volume %q, failure %v", key, ctrl.volumeQueue.NumRequeues(obj))
				ctrl.volumeQueue.AddRateLimited(obj)
			} else if ctrl.volumeQueue.NumRequeues(obj) < ctrl.failedDeleteThreshold {
				glog.Warningf("Retrying syncing volume %q because failures %v < threshold %v", key, ctrl.volumeQueue.NumRequeues(obj), ctrl.failedDeleteThreshold)
				ctrl.volumeQueue.AddRateLimited(obj)
			} else {
				glog.Errorf("Giving up syncing volume %q because failures %v >= threshold %v", key, ctrl.volumeQueue.NumRequeues(obj), ctrl.failedDeleteThreshold)
				// Done but do not Forget: it will not be in the queue but NumRequeues
				// will be saved until the obj is deleted from kubernetes
			}
			return fmt.Errorf("error syncing volume %q: %s", key, err.Error())
		}
	}(obj)

	if err != nil {
		utilruntime.HandleError(err)
		return true
	}

	return true
}

// syncVolumeHandler 从 informer 的缓冲中获取 volume，然后调用 syncVolume 方法
func (ctrl *ProvisionController) syncVolumeHandler(key string) error {
	volumeObj, exists, err := ctrl.volumes.GetByKey(key)
	....
	return ctrl.syncVolume(volumeObj)
}
// 检查传进来的 Volume 是否是 PersistentVolume，然后判断是否需要删除。
func (ctrl *ProvisionController) syncVolume(obj interface{}) error {
	volume, ok := obj.(*v1.PersistentVolume)
	if !ok {
		return fmt.Errorf("expected volume but got %+v", obj)
	}
	// 这里判断是否需要删除这个 volume
	if ctrl.shouldDelete(volume) {
		startTime := time.Now()
		err := ctrl.deleteVolumeOperation(volume)
		ctrl.updateDeleteStats(volume, err, startTime)
		return err
	}
	return nil
}



```

shouldDelete 方法这里通过多个判断条件来决定是否可以删除

```
// shouldDelete returns whether a volume should have its backing volume
// deleted, i.e. whether a Delete is "desired"
func (ctrl *ProvisionController) shouldDelete(volume *v1.PersistentVolume) bool {
	// 断言是否是 DeletionGuard 类型，是则调用 ShouldDelete 继续判断。
	if deletionGuard, ok := ctrl.provisioner.(DeletionGuard); ok {
		if !deletionGuard.ShouldDelete(volume) {
			return false
		}
	}


	// In 1.9+ PV protection means the object will exist briefly with a
	// deletion timestamp even after our successful Delete. Ignore it.
	// k8s 1.9 版本以上
	if ctrl.kubeVersion.AtLeast(utilversion.MustParseSemantic("v1.9.0")) {
		// addFinalizer 为 true，且这个 volume 的 finalizers 中有 "external-provisioner.volume.kubernetes.io/finalizer"
		// 且被设置了删除时间戳，则不需要删除
		if ctrl.addFinalizer && !ctrl.checkFinalizer(volume, finalizerPV) && volume.ObjectMeta.DeletionTimestamp != nil {
			return false
		// 被设置了删除时间戳，则不需要删除
		} else if volume.ObjectMeta.DeletionTimestamp != nil {
			return false
		}
	}

	// In 1.5+ we delete only if the volume is in state Released. In 1.4 we must
	// delete if the volume is in state Failed too.
	// k8s v1.5.0 以上
	if ctrl.kubeVersion.AtLeast(utilversion.MustParseSemantic("v1.5.0")) {
		// volume 的状态不是 Released，不需要删除
		if volume.Status.Phase != v1.VolumeReleased {
			return false
		}
	// v1.5.0 以下
	} else {
		// volume 状态不是 Released 且不是 Failed
		if volume.Status.Phase != v1.VolumeReleased && volume.Status.Phase != v1.VolumeFailed {
			return false
		}
	}
	// PV 回收策略不是 Delete，不需要删除
	if volume.Spec.PersistentVolumeReclaimPolicy != v1.PersistentVolumeReclaimDelete {
		return false
	}
	// annotation 中有 "pv.kubernetes.io/provisioned-by"，不需要删除
	if !metav1.HasAnnotation(volume.ObjectMeta, annDynamicallyProvisioned) {
		return false
	}

	ann := volume.Annotations[annDynamicallyProvisioned]
	migratedTo := volume.Annotations[annMigratedTo]
	"pv.kubernetes.io/provisioned-by" 和 "pv.kubernetes.io/migrated-to" 的值都不是 csi driverName，不需要删除
	if ann != ctrl.provisionerName && migratedTo != ctrl.provisionerName {
		return false
	}

	return true
}
```

满足删除条件之后，调用 deleteVolumeOperation，这里就会调用 csi 实现的 RPC DeleteVolume 来删除 volume

```

func (ctrl *ProvisionController) deleteVolumeOperation(volume *v1.PersistentVolume) error {
	operation := fmt.Sprintf("delete %q", volume.Name)
	glog.Info(logOperation(operation, "started"))
	....
	// 调用 csi 的 RPC DeleteVolume 
	err = ctrl.provisioner.Delete(volume)
	if err != nil {
		if ierr, ok := err.(*IgnoredError); ok {
			// Delete ignored, do nothing and hope another provisioner will delete it.
			glog.Info(logOperation(operation, "volume deletion ignored: %v", ierr))
			return nil
		}
		// Delete failed, emit an event.
		glog.Error(logOperation(operation, "volume deletion failed: %v", err))
		ctrl.eventRecorder.Event(volume, v1.EventTypeWarning, "VolumeFailedDelete", err.Error())
		return err
	}

	glog.Info(logOperation(operation, "volume deleted"))

	// 删除 persistent volume
	if err = ctrl.client.CoreV1().PersistentVolumes().Delete(volume.Name, nil); err != nil {
		// Oops, could not delete the volume and therefore the controller will
		// try to delete the volume again on next update.
		glog.Info(logOperation(operation, "failed to delete persistentvolume: %v", err))
		return err
	}

	if ctrl.addFinalizer {
		if len(newVolume.ObjectMeta.Finalizers) > 0 {
			// Remove external-provisioner finalizer

			// need to get the pv again because the delete has updated the object with a deletion timestamp
			newVolume, err := ctrl.client.CoreV1().PersistentVolumes().Get(volume.Name, metav1.GetOptions{})
			if err != nil {
				// If the volume is not found return, otherwise error
				if !apierrs.IsNotFound(err) {
					glog.Info(logOperation(operation, "failed to get persistentvolume to update finalizer: %v", err))
					return err
				}
				return nil
			}
			finalizers := make([]string, 0)
			for _, finalizer := range newVolume.ObjectMeta.Finalizers {
				if finalizer != finalizerPV {
					finalizers = append(finalizers, finalizer)
				}
			}

			// Only update the finalizers if we actually removed something
			if len(finalizers) != len(newVolume.ObjectMeta.Finalizers) {
				newVolume.ObjectMeta.Finalizers = finalizers
				if _, err = ctrl.client.CoreV1().PersistentVolumes().Update(newVolume); err != nil {
					if !apierrs.IsNotFound(err) {
						// Couldn't remove finalizer and the object still exists, the controller may
						// try to remove the finalizer again on the next update
						glog.Info(logOperation(operation, "failed to remove finalizer for persistentvolume: %v", err))
						return err
					}
				}
			}
		}
	}

	glog.Info(logOperation(operation, "persistentvolume deleted"))

	glog.Info(logOperation(operation, "succeeded"))
	return nil
}

```



构造 delete volume request, 调用 CSI RPC DeleteVolume。

```
func (p *csiProvisioner) Delete(volume *v1.PersistentVolume) error {
	if volume == nil {
		return fmt.Errorf("invalid CSI PV")
	}

	if volume.Spec.CSI == nil {
		return fmt.Errorf("invalid CSI PV")
	}

	volumeId := p.volumeHandleToId(volume.Spec.CSI.VolumeHandle)

	rc := &requiredCapabilities{}
	if err := p.checkDriverCapabilities(rc); err != nil {
		return err
	}
	// 构造 DeleteVolumeRequest
	req := csi.DeleteVolumeRequest{
		VolumeId: volumeId,
	}

	// get secrets if StorageClass specifies it
	storageClassName := util.GetPersistentVolumeClass(volume)
	if len(storageClassName) != 0 {
		if storageClass, err := p.scLister.Get(storageClassName); err == nil {
			// Resolve provision secret credentials.
			provisionerSecretRef, err := getSecretReference(provisionerSecretParams, storageClass.Parameters, volume.Name, &v1.PersistentVolumeClaim{
				ObjectMeta: metav1.ObjectMeta{
					Name:      volume.Spec.ClaimRef.Name,
					Namespace: volume.Spec.ClaimRef.Namespace,
				},
			})
			....

			credentials, err := getCredentials(p.client, provisionerSecretRef)
			if err != nil {
				// Continue with deletion, as the secret may have already been deleted.
				klog.Errorf("Failed to get credentials for volume %s: %s", volume.Name, err.Error())
			}
			req.Secrets = credentials
		} else {
			klog.Warningf("failed to get storageclass: %s, proceeding to delete without secrets. %v", storageClassName, err)
		}
	}
	....
	// 调用 CSI RPC DeleteVolume
	_, err = p.csiClient.DeleteVolume(ctx, &req)

	return err
}
```
